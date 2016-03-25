1. 不被GC眷顾的地方
众所周知，JS是拥有GC的语言。全部JS对象在你不需要它时，会被JS解释器自动地回收掉，解释器为我们管理资源，那什么要自己动手咧？
嘛嘛……让我们先回忆一下我们怎样创建WebGL纹理吧。
var tex = gl.createTexture()
gl.bindTexture(gl.TEXTURE_2D, tex)
gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, image)
gl.bindTexture(gl.TEXTURE_2D, null)
还有销毁纹理对象。
gl.deleteTexture(tex)
如果你熟悉OpenGL或OpenGLES一定读得懂这段代码。gl.createTexture创建纹理并返回；gl.bindTexture绑定我们要操作的纹理；gl.deleteTexture函数销毁纹理。
等等……销毁纹理……JS是有GC的说，为什么要手动销毁纹理？或许是为了和OpenGL保持一致，或许是别的原因，API被设计成这样，资源管理变成了我们的任务。联想到WebGL/OpenGL中buffer、shader、framebuffer与texture的相似的操作流程……
卧槽，WebGL完美地绕开了JS虚拟机，这些存在于显存中的对象是不受JS虚拟机GC管辖的。真的是这样吗？是，也不是。
var canvas = document.createElement("canvas")
var gl = canvas.getContext("webgl")
// do any thing you like.
gl = null
canvas = null
这是我们初始化webGL的代码，webGL是从一个canvas上创建的，我这边特别写出了gl和canvas置空的代码，因为很重要。JS虚拟机的做法是将webGL中用到的全部资源和它所属的canvas关联。当程序不再引用canvas时，WebGL上的全部资源和canvas一起等待被GC销毁。
于是WebGL成了JS虚拟机直辖下的特别行政区。
问：你们JS有没有虚拟机？
答：有。
问：那么这个texture你们管不管？
答：不管。
我#@~:%@&=$+……从未见过如此厚颜无耻之……代码。
对此你可能还有疑问，在OpenGL中gl.createTexture返回的是一个整数类型（非指针），一个纯粹的句柄，所以texture等对象一定由OpenGL运行时管理。但是webGL不是。
console.log(textrue) // => WebGLTexture {}
在chrome中打印一下textrue，发现textrue是个叫做WebGLTexture的原生对象。欸~是对象，对象可以被GC呀，说不定gl.deleteTexture仅仅是给JS虚拟机的一个暗示（hint），告诉JS虚拟机立刻回收纹理。理由是JS的GC实现（mark and sweep）有一定周期，不会立刻回收废弃的显存对象。gl.deleteTexture是用来弥补这个缺点的，告诉虚拟机不用等GC了，立刻删了它，这事我说了算。
perfect！这个解释看似完美。怀着和你一样的侥幸我Google了一下答案是……否定的。有兴趣的同学可以参见这里和这里。
并且JS是没有析构函数的。既没有C++中~Class()那种析构时必定调用的函数，也有没Java中finalize()、Python中__del__()，会由虚拟机自动调用的资源释放函数。texture释放由程序员自己保证，一夜间我们回到了上世纪70年代C语言岁月。
好在还是可以抢救一下，下面我们来尝试一些方法。


2. 方案一：视而不见
假如你的WebGL只是用来展示几个3D模型，一块不大的固定场景，或者一个轻度的手游，那么对此视而不见吧。
因为当今（2016年）多数电脑和大部分手机，提供300M-500M的显存（内存）不是问题。在3D应用中，将资源维持在显存（内存）里还能提升应用的流畅度。等你不需要<cnavas>时，JS一并帮你回收掉。何乐而不为咯。
问题存在于需要长时间运行，不断加载新资源的应用中。比如AVG游戏中，随着剧情推进不断出现的背景、立绘和CG，甚至过场动画。再比如一个拥有巨大地图的RPG。


3. 方案二：A hack way
这是我近一段时间内看到过的最聪明的做法，虽然存在一些局限性（我之后说明），仍然值得学习。方法并非我原创，出自。
回想一下我们的目的：管理WebGL中的纹理，并非WebGLTexture对象。什么意思？我们一直默认WebGLTexture就是纹理，真的是吗？WebGLTexture也可以不对应纹理，在gl.texImage2D()调用之前。
gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, image)
这个调用中image是什么？多半是HTML中的<image>也可以是<canvas><video>等。
OK！WebGLTexture JS虚拟机不管，<image>总得管吧。试试看从<image>上做文章。
var texArray = [
    gl.createTexture(),
    gl.createTexture(),
    gl.createTexture(),
    gl.createTexture(),
]
//  0 <= index <= 3
function bindImage(index, image) {
    gl.bindTexture(gl.TEXTURE_2D, tex[index])
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGB, gl.RGB, gl.UNSIGNED_BYTE, image)
    gl.activeTexture(_webGL.TEXTURE0 + index)
    gl.bindTexture(gl.TEXTURE_2D, null)
}
使用纹理的时候不创建WebGLTexture，代之：
bindImage(0, image0)
bindImage(1, image1)
// ...
gl.drawArray()
初始化好足够的WebGLTexture。这里是4个，也可以更多，看你需要。不要直接将<image>绑定到WebGLTexture而是等用到时再绑定。然后你会发现，问题不存在了，除了当前被绑定的<image>，其他<image>已经可以被JS的GC识别啦。
来做额外说明，如果你和我一样用过OpenGL，你大概会怀疑这个做法的效率问题。OpenGL中gl.texImage2D()函数会实打实地将image中的数据由内存传输到显存。好在浏览器环境下不一定。现代浏览器为了效率会借助GPU来渲染页面，会不会<image>本身就是一块纹理，已经经存在于显存中了呢。会不会gl.texImage2D()调用实际上并没有传输数据，仅仅取出了<image>中本来存在的OpenGL句柄呢。
以上纯属猜测，依赖于浏览器的实现，万一猜错，我们亏大了。再想一想gl.texImage2D()中的最后一个参数除了<image>还能是<canvas>和<video>这两货常驻显存中没什么争议了吧。<canvas>的2dAPI中有一个叫做drawImage()的函数，我们可以把<image>画到<canvas>上，用<canvas>代替<image>。
至于该方法的局限性，只考虑了texture的管理，可以用<canvas>代替framebuffer，还有shader和buffer没法管理。
做过2D游戏（cocos2d啊，egret什么的）都知道，2d游戏中不存在复杂的模型，绝大多数绘制对象是映射了纹理的矩形，被称之为sprite。sprite通常是没有对应buffer的，绘图时，引擎将sprite的顶点数据写入到一个公用的buffer中，最后批量提交GPU绘制。矩形的数据量小，共用buffer还能做一些特别的优化（主要是批量绘制，减少WebGL的draw call 之类的）。buffer的问题也得以解决。
至于shader，全部常驻内存又何妨呢？
所以这个方法在2D游戏中完全是可用的，AVG可以用吗？可以。
嗯……方法已找到，完结撒花。
作为一个有追求的骚年，我还想做3D的背景哩、粒子系统和Live2d类型的网格动画做做做，3渲2的卡通渲染好想试一试……


4. 来自C++
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













