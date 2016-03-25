# WebGL资源管理


## 不被 GC 眷顾的地方

众所周知，JS 是拥有 GC 的语言。JS 中全部的对象在你不需要它时会被 JS 解释器自动地回收掉，既然 JS 为我们做了管理，那么什么要自己动手？

嘛嘛……先让我们回忆一下我们怎样创建 WebGL 纹理。

```js
var texture = gl.createTexture()
gl.bindTexture(gl.TEXTURE_2D, texture)
gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, image)
gl.bindTexture(gl.TEXTURE_2D, null)
```

还有销毁纹理对象。

```js
gl.deleteTexture(texture)
```

如果你熟悉 OpenGL 或 OpenGLES 一定读得懂。gl.createTexture() 创建纹理并返回；gl.bindTexture() 绑定我们要操作的纹理；gl.deleteTexture() 销毁纹理。

等等……销毁纹理…… JS 是有 GC 的语言，为什么要手动销毁纹理？也许是为了和 WebGL/OpenGL 保持一致？或者是别的不得而知的原因，API就是被设计成了这样啊。于是资源管理成了我们的任务。联想到 WebGL/OpenGL 中 buffer、shader、framebuffer 与 texture 相似的操作流程……

卧槽，WebGL完美地绕开了JS虚拟机，这些存在于显存中的对象是不受JS虚拟机GC管辖的。真的是这样吗？

先说答案：是，也不是。

```js
var canvas = document.createElement("canvas")
var gl = canvas.getContext("webgl")
// do something you like ...
gl = null
canvas = null
```

这是我们初始化 WebGL 的代码，WebGL 是从一个 <canvas> 上创建的，这里特别写出了 gl 和 <canvas> 置空的代码，因为很重要。JS 虚拟机的做法是将 WebGL 中用到的全部资源和它所属的 <canvas> 关联。当程序不再引用 <canvas> 时，WebGL 上的全部资源和 <canvas> 一起等待被 GC 销毁。

于是 WebGL 成了 JS 虚拟机直辖下的特别行政区。
* 问：你们 JS 有没有虚拟机？
* 答：有。
* 问：那么这个 texture 你们管不管？
* 答：不管。

我#@~:%@&=$+……从未见过如此厚颜无耻之……代码。

对此你可能还有疑问，在 OpenGL 中 gl.createTexture() 返回的是一个整数类型（非指针），一个纯粹的句柄类型，所以 texture 等对象一定由 OpenGL 运行时引用并管理。但是 WebGL 不是，尝试打印我们的纹理对象。

```js
console.log(textrue) // => WebGLTexture {}
```

在 chrome的结果显示，纹理是个叫做 WebGLTexture 的原生对象。欸~是对象，对象可以被 GC 。说不定 gl.deleteTexture() 仅仅是给 JS 虚拟机的一个暗示（hint），告诉 JS 虚拟机立刻回收 texture。理由是 JS 的 GC 算法（mark and sweep）有一定周期，不会立刻回收废弃的显存对象。gl.deleteTexture() 是用来弥补这个缺点的，告诉虚拟机不用等GC了，立刻删了它，这事我说了算。

perfect！这个解释看似完美。怀着和你一样的侥幸我 Google 了一下答案是……否定的。有兴趣的同学可以参见：
* [Khronos Group 的 mail list](https://www.khronos.org/webgl/public-mailing-list/archives/1106/msg00102.php)
* [StackOverflow 上的问题](http://stackoverflow.com/questions/31250052)

并且 JS 是没有析构函数的。既没有 C++ 中 ~Class() 这种析构时必定调用的函数，也有没 Java 中 finalize()、Python 中 \_\_del__()，会由虚拟机自动调用的资源释放函数。texture 释放由程序员自己保证，一夜间我们回到了上世纪70年代 C 语言岁月。

好在还是可以抢救一下，下面我们来尝试一些方法。


## 方案一：视而不见

假如你的 WebGL 只是用来展示几个 3D 模型，一块不大的固定场景，一个轻度的手游，那么对此视而不见吧。

因为当今（2016年）多数电脑和大部分手机，提供 300M-500M 的显存（内存）不是问题。3D 应用中，将资源维持在显存（内存）里还能提升应用的流畅度。等你不需要 <cnavas> 时，JS 一并帮你回收掉。何乐而不为。
问题存在于需要长时间运行，不断加载新资源的应用中。比如 AVG 游戏中，随着剧情推进不断出现的背景、立绘和 CG，甚至过场动画。再比如一个拥有巨大地图的 RPG。

### 优点：
* 不需要做任何特殊处理。

### 缺点：
* 要求应用规模不大，运行时间不长。


## 方案二：A hack way

这是我近一段时间内看到过的最聪明的做法，虽然存在一些局限性（我之后说明），仍然值得学习。方法并非我原创，[英文原版看这里](http://blog.tojicode.com/2012/03/javascript-memory-optimization-and.html)。

回想一下我们的目的：管理 WebGL 中的纹理，并非管理 WebGLTexture。什么意思？我们一直默认 WebGLTexture 就是纹理，真的是吗？不是的，WebGLTexture 也可以不对应纹理，例如 gl.texImage2D() 调用之前。

```js
gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, image)
```

这个调用中 image 是什么？多半是 HTML 中的 <image> 也可以是 <canvas> <video> 等。

OK！WebGLTexture JS 虚拟机不管，<image> 总得管吧。我们换个角度，试试看从 <image> 上做文章。

回忆起来，我们如何使用 WebGL 纹理的。WebGL 有一些纹理通道，我们先用 gl.activeTexture() 激活纹理通道，再用 gl.bindTexture() 绑定纹理，这样纹理就和纹理通道关联起来了，之后在 shader 中，我们可以通过纹理通道的编号（0-7 或者更多）来索引我们的纹理。

```js
    gl.activeTexture(_webGL.TEXTURE0)
    gl.bindTexture(gl.TEXTURE_2D, tex[index])
    gl.bindTexture(gl.TEXTURE_2D, null)
    gl.uniform1i(uniform, 0)
   // ...
```

由此可见，整个渲染过程中纹理涉及两次绑定，一次是图片和纹理，一次是纹理和纹理通道。所有 OpenGL 教材教导我们：先图片与纹理，再纹理与纹理通道。为什么不可以反过来呢？

好的，这次我们先绑定纹理与纹理通道，假设我们最多只有8个纹理通道，那么只用8个 WebGLTexture 就够了。WebGLTexture 和 纹理通道一一对应。我们惊讶地发现，现在即使不通过 WebGLTexture 做中介，我们也可以将图片的数据传输到任意一个我们希望的纹理通道。

```js
var texArray = [
    gl.createTexture(),
    gl.createTexture(),
    gl.createTexture(),
    gl.createTexture(),
]
function bindImage(index, image) {
    gl.activeTexture(_webGL.TEXTURE0 + index)
    gl.bindTexture(gl.TEXTURE_2D, tex[index])
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, image)
    gl.bindTexture(gl.TEXTURE_2D, null)
}
```

直接绑定图片与纹理通道。

```js
bindImage(0, image0)
gl.uniform1i(uniform0, 0)
bindImage(1, image1)
gl.uniform1i(uniform1, 1)
```

问题不存在了。

WebGLTexture 数量固定，不会收就不会收吧。除了当前被绑定的 <image>，其他 <image> 都可以被 JS GC 识别并回收。

补充说明：如果你和我一样用过 OpenGL，你大概会怀疑这个做法的效率问题。OpenGL 中 gl.texImage2D() 函数会实打实地将 image 中的数据由内存传输到显存。不过浏览器环境下未必。现代浏览器为了效率会借助GPU来渲染页面，会不会 <image> 本身就是一块纹理，已经经存在于显存中了呢。gl.texImage2D() 调用实际上并没有传输数据，仅仅取出了<image>中本来存在的OpenGL句柄。这依赖于浏览器的实现，万一猜错，我们亏大了。再想一想 gl.texImage2D() 中的最后一个参数除了 <image> 还能是 <canvas> <video>。这两货常驻显存中没什么争议了吧。<canvas> 的 2D API 中有一个叫做 drawImage() 的函数，我们可以把 <image> 画到 <canvas> 上，所以请用 <canvas> 代替 <image>。

最后谈谈该方法的局限性。

这个方法只考虑了 texture 的管理，可以用 <canvas> 代替 framebuffer，还有 shader 和 buffer 没法管理。

做过 2D 游戏（cocos2d 啊，egretD什么的）都知道，2D 游戏中不存在复杂的模型，绝大多数绘制对象是带贴图的矩形，被称为 sprite（精灵）。sprite 通常是没有对应 buffer 的，绘图时，引擎将 sprite 的顶点数据写入到一个公用的 buffer 中，最后批量提交 GPU 绘制。矩形的数据量小，公用 buffer 还能做一些特别的优化（主要是批量绘制，减少 WebGL 的 draw call 之类的）。buffer 的问题也得以解决。

要说shader嘛，全部常驻内存又何妨呢？
所以这个方法在 2D 游戏中完全是可用的，AVG 可以用吗？可以。

### 优点
* 完全基于 JS 的内置功能，额外代码需求少。
* 是一种全自动的回收方式，无需程序员介入就能工作。

### 缺点
* 仅限 texture 可用，或仅限 2D 可用。
* <image> 到 <canvas> 有一定性能损失。

嗯……方法已找到，完结撒花。

等一等。作为一个有追求的骚年，不想试试 3D 的背景？粒子系统和 Live2d 类型的网格动画呢？三渲二的卡通渲染呢？<image> 到 <canvas> 的性能损失也希望尽量避免。


## 古老的方法
之前的讨论，我们一直尝试在JS的范畴内寻找解决办法。我们的需求其实是给不受GC管理的显存加上GC。倘若我们更原始的语言，比如C++中遇到此类问题会作何反应，自己艹一个咯。方法选引用计数，因为，简单，简单，简单。
如果你用过C++，还读过 C++ primer 你多半对引用计数很熟悉。为了尽量减少手动IncRef()和DecRef()，我们这样设计：
function Textrue(url) {
    this.glTexture = gl.createTexture()
    // init texture ...
}
Texture.IncRef = function() { /* ... */ }
Texture.DecRef = function() { /* ... */ }
var textureMap = {}
funciton newTexture(name) {
    if (textureMap[name]) {
        return textureMap[name]
    } else {
        var texture = new Texture(name)
        textureMap[name] = texture
        return texture;
    }
}
为了方便地使用Texture类，我们的显示对象也需要改造。
function Image() {
    this.texture = null;
    // ...
}
Image.prototype.setTexture(texture) {
    texture.IncRef()
    if (this.texture && 0 == this.texture.DecRef()) {
        tex.destory();
    }
}
我们将创建的全部texture保存在一个map中做引用计数，这个map负责资源管理的同时还承担了缓存的任务，当你创建两个url相同的texture时，不再需要新建一个texture。
var image = new Image()
var texture = newTexture(url)
image.setTexture(texture)
将Texture置入Image之后，Image会在合适的时候销毁Texture。
说实话引用计数勉强堪用，只不过在Image之外想引用Texture对象你就有的受了，你得自己增减引用计数，或者搭配上ArrayRef、MapRef之类的特殊容器。还得小心循环引用的问题，好在一般情况下Texture不会引用另一个Tex吧。
比稍原生API有进步，但依然原始。旧版本的FenQi.Engine使用的是引用计数。

5. 











