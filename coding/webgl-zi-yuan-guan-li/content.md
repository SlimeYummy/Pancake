# WebGL 资源管理

- NaNuNo
- 17-01-01



WebGL 在给 HTML5 带来 3D 能力的同时也带来了许多 OpenGL 固有的问题，其中 WebGL 显示资源（纹理、Shader、缓冲区等）的管理尤其让人头大。

与 DirectX 显示资源对应于 C++ 对象的管理方式不同，OpenGL 保留了古时基于资源句柄的资源管理方式。这在 C（手动管理）、C++（RAII）中不被视为问题；在 Java（finalize 函数）、Python（\_\_del\_\_ 函数）中被资源回收函数缓解；却在 JS 这种无法触及对象生命周期的语言中产生了巨大的麻烦。

本文会先说明 JS 中 WebGL 资源管理问题的由来，可能遇到该问题的场景，最后再提供三种在 JS 语言范畴内解决或规避问题的思路，每种思路的出发点各不相同，针对的应用场景也完全不同。

## 不被 GC 眷顾的地方

众所周知，JS 是内置 GC 的语言。 JS 中使用到的全部的对象，在失去全部引用后会被 JS 运行时自动地回收掉，既然 JS 运行时已经做好了资源管理，那什么要认为参与呢？

先让回忆一下怎样创建 WebGL 纹理吧。

```js
var texture = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, texture);
gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, image);
gl.bindTexture(gl.TEXTURE_2D, null);
```

还有销毁纹理。

```js
gl.deleteTexture(texture)
```

如果熟悉 OpenGL/OpenGLES 一定读得懂以上代码。gl.createTexture() 创建纹理并返回；gl.bindTexture() 绑定即将要操作的纹理；gl.deleteTexture() 销毁纹理。

等等……销毁纹理…… JS 是自带 GC 的语言，为什么要手动销毁纹理？也许是为了和 OpenGL/OpenGLES 保持一致，也许是别的什么，API 就是被设计成了这样。于是管理 WebGL 资源成了程序员的任务。联想到 buffer、shader、framebuffer 与 texture 相似的操作流程……

WebGL 完美地绕开了 JS GC 机制，这些存在于显存（或内存特殊区域）中的 WebGL 资源是不受 JS GC 管辖的。真的是这样吗？

先说答案：是，也不是。

```js
var canvas = document.createElement("canvas");
var gl = canvas.getContext("webgl");
// do anything you like ......
gl = null; // step A
canvas = null; // step B
```

这是通常初始化 WebGL 的标准流程，WebGL 从一个 canvas 上被创建，这里特别写出了 gl 和 canvas 置空的代码，因为很重要。JS 的做法是将 WebGL 的全部资源和它所属的 canvas 关联。当程序不再引用 canvas 和 gl 时，WebGL 的全部资源和 canvas 一起等待被 GC 销毁。

以上代码中只有到了 step B 这一行运行完毕，WebGL 及全部 WebGL 资源才有可能和 canvas 一起触发 JS CG。

看起来，WebGL 成了 JS 直辖下的特别行政区。

* 问：你们 JS 有没有 GC？
* 答：有。
* 问：那么这个 texture 你们管不管？
* 答：不管。

这样的 GC 方式有卵用啊。

对此比较熟悉 WebGL 的同学可能抱有疑问，因为在 OpenGL/OpenGLES 中 gl.createTexture() 返回的是一个整数类型（非指针）的句柄，一个纯粹的句柄类型，所以 texture 等对象一定由 OpenGL/OpenGLES 运行时引用并管理。

但是 WebGL 不是，在 Chrome 中尝试打印纹理对象。

```js
var texture = gl.createTexture();
console.log(textrue); // => WebGLTexture {}
```

结果显示，WebGL 产生的纹理是个叫做 WebGLTexture 的原生对象。而 JS 对象应当是可以被独立 GC 掉的，会不会浏览器环境的实现是这样：

gl.deleteTexture() 仅仅是给 JS 的一个暗示（hint），告诉 JS 立刻回收 texture。理由是很多 JS GC 算法（比如 mark and sweep）有一定周期性，无法立刻回收被废弃的 WebGL 资源，并且 JS GC 对消耗的内存资源敏感，对显存资源不敏感，分配出去的显存可能需要很久才能得到回收。gl.deleteTexture() 是用来弥补这个缺陷的，告诉虚拟机不用等 GC 运行了，立刻删了它。

这个解释说得通，蛮有道理。那么浏览器究竟是不是这样处理的呢？Google 一下会找到这些：

* [Khronos Group 的 mailing list](https://www.khronos.org/webgl/public-mailing-list/archives/1106/msg00102.php)
* [StackOverflow 上的问题](http://stackoverflow.com/questions/31250052)

答案是否定的，WebGL 会在内部引用其创建的全部资源。

并且 JS 是没有析构函数的。既没有 C++ ~Class() 这种析构时必定调用的函数，也有没 Java finalize()、Python \_\_del\_\_()，会自动调用的资源释放函数。WebGL 资源的释放完全由程序员自己保证，一夜间仿佛回到了上世纪70年代 C 语言的岁月。

有没有办法呢，下面尝试抢救一下。

## 方案一：视而不见

假如 WebGL 程序只是用来展示几个 3D 模型，一块不大的固定场景，一个轻度的手游，那么对此视而不见吧。

因为当今（2016年）多数电脑和大部分手机，提供 300M-500M 的显存（内存）不是问题。3D 应用中，将资源维持在显存（内存）里还能提升应用的流畅度。等程序不需要 cnavas 时，JS 自然会一并回收掉，何乐而不为？

问题会出现在需要长时间运行，执行期间不断有 WebGL 资源被加载的应用。比如一个拥有巨大地图的 RPG 游戏，一个多图的幻灯片应用。显存（内存）再多也不是无底洞，终究有被耗尽的可能。更何况资源相对紧张的移动端和低端设备。

### 优点：
* 不需要做任何特殊处理。

### 缺点：
* 要求应用规模不大，运行时间不长。

## 方案二：Hack 的做法

这是一个极为聪明和另辟蹊径的做法，虽然存在一些局限性（之后说明），仍然值得学习。方法并非 NaNuNo 原创，[英文原版看这里](http://blog.tojicode.com/2012/03/javascript-memory-optimization-and.html)。

先明确，资源管理的目标是管理 WebGL 引用的显存资源，并非管理 WebGLTexture 等对象。上文中一直默认 WebGLTexture 就是纹理，管理纹理就是管理 WebGLTexture 对象，通常的做法也是如此，但 WebGLTexture 也可以不对应显存内资源的。

```js
var texture = gl.creareTexture(); // empty
gl.bindTexture(gl.TEXTURE_2D, texture); // empty
gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, image); // resource
gl.bindTexture(gl.TEXTURE_2D, null); // resource
```

还是文章一开始创建纹理的代码，一直到第三行，调用 gl.texImage2D() 之前，WebGLTexture 都是不对应显存中任何资源的，这提供了一个 管理纹理资源的思路。

gl.texImage2D() 调用中 image 是什么？多半是 HTML DOM 中的 image 对象 也可以是 canvas 对象、video  对象等，而 image 对象、canvas 对象 显然是受 JS GC 管辖的，从 image 上做文章或许有戏。

```js
gl.activeTexture(gl.TEXTURE0);
gl.bindTexture(gl.TEXTURE_2D, textrue);
gl.bindTexture(gl.TEXTURE_2D, null);
gl.uniform1i(uniform, 0);
// do anything you like ......
```

回忆使用 WebGL 纹理的代码。WebGL 有一些纹理通道，先用 gl.activeTexture() 激活纹理通道，再用 gl.bindTexture() 绑定纹理和纹理通道，之后在 shader 中可以通过纹理通道的编号（0-7 或者更多）索引纹理。

由此可见，整个渲染过程纹理涉及两次绑定，第一次绑定涉及图片和纹理，第二次绑定涉及纹理和纹理通道。所有 OpenGL 教材教导我们：先绑定图片与纹理，再绑定纹理与纹理通道。假如反过来呢？

```js
// prepare texture
var texArray = [
    gl.createTexture(),
    gl.createTexture(),
    gl.createTexture(),
    gl.createTexture(),
];
function bindImage(index, image) {
    gl.activeTexture(_webGL.TEXTURE0 + index);
    gl.bindTexture(gl.TEXTURE_2D, tex[index]);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, image);
    gl.bindTexture(gl.TEXTURE_2D, null);
};
// bind image directly
bindImage(0, image0);
gl.uniform1i(uniform0, 0);
bindImage(1, image1);
gl.uniform1i(uniform1, 1);
```

以上代码以 4 个纹理通道为例，将 WebGLTexture 与纹理通道一一对应，使用纹理时则通过 bindImage() 函数直接传入图片。令人惊讶的事发生了，现在，即使不通过 WebGLTexture 做中介，也可以将图片的数据传输到任意的纹理通道，直接供 shader 使用！

直接绑定图片与纹理通道。

问题解决。WebGLTexture 数量固定，不回收就不会收吧。除了当前被绑定的 image，其他 image 都可以被 JS 识别并回收。

做一些补充说明。

熟悉 OpenGL/OpenGLES 的人大概会怀疑这个做法的效率问题。OpenGL/OpenGLES 中 gl.texImage2D() 函数会实打实地将 image 中的数据由内存传输到显存。但 WebGL 运行在浏览器中，现代浏览器使用 GPU 来渲染页面，会不会 image 本身就是一块纹理。那么 gl.texImage2D() 调用实际上并没有传输数据，仅仅取出了 image 中本来存在的 OpenGL 句柄。这依赖于浏览器的实现。

然而 gl.texImage2D() 中的最后一个参数除了 image 还能是 canvas 和 video。这两货常驻显存中没什么争议了吧。 canvas 的 2D API 中有一个叫做 drawImage() 的函数可以把 image 拷贝到 canvas 上。所以请用 canvas 代替 image。

最后谈谈该方法的局限性。最大的问题，该方法只考虑了 texture 的管理，即使用 canvas 代替 framebuffer，还有 shader 和 buffer 没法管理。

做过 2D 游戏（比如 cocos2d）都知道，2D 游戏中不存在复杂的模型，绝大多数绘制对象是带贴图的矩形，被称为 sprite（精灵）。sprite 通常是没有对应 buffer 的，绘图时，引擎将 sprite 的顶点数据写入到一个公用的 buffer 中，最后批量提交 GPU 绘制。矩形的数据量小，公用 buffer 还能做一些特别的优化（主要是批量绘制，减少 WebGL 的 draw call 之类的）。buffer 的问题也得以解决。

要说shader嘛，就那么一点数据，全部常驻内存又何妨呢？

所以这个方法在 2D 游戏中完全是可用的。

### 优点
* 完全基于 JS 的内置功能，额外代码需求少。
* 是一种全自动的回收方式，无需程序员介入就能工作。

### 缺点
* 仅限 texture 可用，或仅限 2D 可用。
* image 到 canvas 有一定性能损失。

## 古老的方法

之前，一直尝试直接组合 JS 内置的能力解决问题，资源管理的需求简单又明确，能不能用 JS 实现一个 WebGL 资源管理器。

在很久很久以前，Lisp 和 SmallTalk 的 GC 功能还躺在实验室里吃灰的时候，C++ 已经开始用基于引用计数的智能指针做资源管理的尝试了。

思路很简单，利用引用计数将 WebGLTexture 包装到 Texture 中，并建立一个 map 来缓存来自同一个 url 的纹理。由 Texture 来做资源管理。

```js
function Textrue(url) {
    this.refCount = 0;
    this.glTexture = gl.createTexture();
    // init texture here .....
}
Texture.prototype.incRef = function() {
    return ++this.refCount;
}
Texture.prototype.decRef = function() {
    if (0 === --this.refCount) {
        gl.deleteTexture(this.glTexture);
        this.glTexture = null;
      	delete textureMap[url];
    }
}

var textureMap = { };

function newTexture(url) {
    if (textureMap[url]) {
        return textureMap[url];
    } else {
        var texture = new Texture(url);
        textureMap[url] = texture;
        return texture;
    }
}

var texture = newTexture("tex.png");
texture.incRef();
// do anything you like ......
texture.decRef();
```

单纯引入引用计数没有解决问题，曾经的 gl.deleteTexture() 函数被替换为了 texture.decRef()，依旧需要手动释放资源。

为了便于使用 Texture 类，尝试将应用中要显示的物体包装成显示对象，每一个显示对象对应一个可渲染的实体。比如一个精灵、一个 Live2d 小人，一个粒子系统、一个天空盒。尽量在显示对象的方法中增减引用计数，而非应用代码中手动处理，理想情况下可以不用手动 incRef() 和 decRef()，显示对象帮忙处理好一切。

```js
function Sprite() {
    this.texture = null;
    // init sprite here ......
}
Sprite.prototype.setTexture = function(texture) {
    texture.incRef();
    if (this.texture) {
        this.texture.decRef();
    }
    this.texture = texture;
}
Sprite.prototype.destory = function() {
    if (this.texture) {
        this.texture.decRef();
    }
}

var sprite = new Sprite()
var texture = newTexture(url)
sprite.setTexture(texture)
// do anything you like
sprite.destory()
```

看出来了吗？目标依旧没有实现，这次 Texture 实现了自动释放，代价是 Sprite 不用时需要调用 sprite.destory()，我们手动管理的对象从 Texture 变成了 Sprite。

究其原因，是因为 JS 中没有析构函数造成的，sprite.destory() 实际上 Sprite 的析构函数，最好像 Java 一样，由 Java 自动帮调用，但 JS 显然没这个功能。

那其它 GC 算法呢？JS 虚拟机本身使用的 GC 算法是 mark and sweep （实际上复杂一些，还包括分代、分步骤渐进、写栅栏等）这种算法需要定时遍历代码中全部的对象，找出死对象。JS 闭包里的对象外界是无权访问的，所以用不了。至于其他 GC 基本属于以上两种的改进。

WebGL 资源管理模块实现失败了。

## 方案三：和场景在一起

先不管失败的方案，记得方案一吗？

方案一之所以可行，在于 WebGL 资源最终会与 canvas 一同销毁。回头来看 JS 的做法本无可厚非。应用之所以加载 WebGL 资源，就是为了绘图嘛，如果不会图了，那么 WebGL 对象就不需要了，可以被回收。WebGL 对象实际上是和应用要绘制的场景高度关联的，JS 的问题在于粒度太大，全部 WebGL 对象和一个 canvas 关联，要能减小些粒度就好了。

在 cocos2d、three.js、unity3d（都没的用过，参考 DOM 树）等游戏引擎中，Sprite 或者 Cube 一类的显示对象被组织在一个树形结构中。场景树的特点是，当你希望显示一个对象时，将这个对象放入场景树，反之从场景树中移除。移除一个节点，该节点的子节点也随之被移除，这个特性很棒。

规定显示对象必须存在于场景树上，显示对象从场景树上移除等价于被销毁（注意这个规定，最后它会成为这个方法要遵循的唯一约定）。请这样理解，之所以要从场景树上移除，是因为被移除的物体不需要被显示了，既然不需要被显示了，那干脆回收它用到的资源吧。

```js
function Node(parent) {
    this.parent = parent;
    this.children = [];
    parent.add(this);
}
Node.prototype.destory = function() {
    for (var idx = 0; idx < this.children.length; ++idx) {
        var child = this.children[idx];
        child.destory();
    }
    // release resource here ......
}
Node.prototype.addChild = function(child) {
    this.children.push(child);
}
Node.prototype.removeFromParent = funciton() {
    this.parent.children.remove(this);
    this.destory();
}
// other tree operates ......
```

这样做的好处是：原本释放资源调用 destory() 容易被忘记，所以手动资源管理不好，而 remove() 就不一样了，不 remove() 显示对象，它就一直显示在那儿，看到效果不对，使用者自然会去移除显示对象，destory() 会递归子节点，移除了父节点也不必担心子节点没有释放。对于那些仅仅想临时隐藏的显示对象，允许 setVisible(false)。

把显示对象和 WebGL 资源人为地从 JS 的环境中拉出来单独做 GC，不知不觉为整个引擎实现了资源管理模块。对于非渲染树上 JS 原生对象对于显示对象的引用，被全部无视了，这些引用等价于一个弱引用。这会造成问题吗？

假设显示对象是活的，可以正常使用，没有问题。假设显示对象是死的，已经被销毁过了，无法渲染，也没有内存泄漏。况且显示对象会死，是因为它移除了渲染树，说明它再也不会被显示了，再也不会被显示的显示对象，留它何用，本来就不应该再被使用。

资源管理模块基本堪用了。再贪心一点，现在的 GC 基于引用计数，可不可以替换成 mark and sweep 算法不需要。之前不用 mark and sweep 是因为遍历不到全部的引用，但是现在不同，需要遍历的引用全部在渲染树上，完全可以递归地标记他们。

```js
var textureMap = {};
function Texture(url) {
    this._gcFlag_ = false;
    // init texture here ......
}
Texture.prototype._gcDestory_ = function() {
    // delete texture here ...
}
function newTexture(url) {
    if (textureMap[url]) {
        return textureMap[url];
    } else {
        var texture = new Texture(url);
        textureMap[url] = texture;
        return texture;
    }
}

function Node(parent) {
    this.parent = parent;
    this.children = [];
    parent.add(this);
}
Node.prototype.addChild = function() {
    // ......
}
Node.prototype.removeFromParent = function() {
    // ......
}
// other method ......
Node.prototype._gcMark_ = function() {
    for (var key in this) {
        var val = this[key]
        if ("function" == typeof(val._gcFlag_)) {
            val._gcFlag_ = true;
        }
    }
}

function gcSteepMark() {
    for allChild in globalRoot {
        allChild._gcMark_();
    }
}
function gcStepSweep() {
    var name, texture;
    for (name, texture in textureMap) {
        if (!texture._gcFlag_) {
            texture._gcDestory_();
            delete textureMap[name];
        }
    }
}
```

这不是一个严谨的 mark and sweep 算法实现，只用于说明流程，仅考虑了 Texture 回收。

由于我们的显示对象不会太多（超不过1000个吧），GC运行的速度尚可接受。而游戏的主循环，天生提供了运行GC的绝佳时机。跑完一轮 GC \_gcFlag\_ 的为 false 的 WebGL资源 就可以被回收啦。

吸取 JS 没有 GC 接口的教训，我们还可以仿照 Lua 提供让使用者主动挂起和发起 GC 的能力。切换场景时关闭 GC 不仅能提高效率，还能提高资源的重复使用率。

### 优点
* 完全的管理 WebGL 资源。
* 提供使用者控制 GC 的能力。

### 缺点
* 和 WebGL API 一样，在 JS 中划出了自治区。

## 最后

首先感谢你看到最后。

WebGL 在赋予程序员资源管理能力的同时也带来了不小的麻烦。以上三种方法作为时候补救的手段都有各自的优势与局限性，FenQi.Engine 选用了第三种实现方式。

如果你对 WebGL 资源管理有更独特的见解，更漂亮的实现，欢迎联系我，写成文@我。

