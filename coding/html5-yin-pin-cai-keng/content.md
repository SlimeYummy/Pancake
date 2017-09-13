# HTML5 音频踩坑

- NaNuNoo
- 2016-08-10
- https://fenqi.io/coding/html5-yin-pin-cai-keng/


这篇文章中的内容是 HTML5 音频库实现一文内容的先行研究。按理来说先行研究没有必要整理成文，没有太多值得记录与分享的内容。然而 HTML5 音频库五花八门的问题令人发指，才有了这篇文章，帮助后人快速地爬出这些坑。

HTML5 原生支持音频播放有一段时间了，有 Audio 元素和 WebAdio 两套音频相关的 API，针对不同的应用场景设计。

* Audio 元素支持音频文件播放，针对音乐类多媒体。出现时间较早，浏览器的支持也较好。

* WebAudio 支持音频编解码，支持音效混合，针对游戏类、音频处理类应用。还比较新，移动浏览器支持稍差。

对于游戏引擎，通行的做法是优先使用 WebAudio API，并支 Audio 元素作为降级处理。但是，和多数 HTML5 新特性一样，Audio 元素和 WebAudio 也有非常严重的兼容性问题，以下总结了 FenQi.Engine开发过程中遇到的坑，附带常见的的解决方案。

##Audio 支持的音频格式

不同浏览器上，Audio 元素支持的音频格式是不同的，尤其是在老旧的桌面浏览器上。

* 早期 FireFox 不支持 mp3
* IE9、早期 Safari 不支持 ogg
* aac 和 webm 早期浏览器多不支持

总体来说，mp3 和 ogg 格式的兼容性比较好，为了保持所有浏览器的兼容，至少需要为同一份音频准备 mp3 和 ogg 两种格式的文件。另外 aac 和 webm 在最新版浏览器中的支持也不错，文件体积相对小。

可以用 audio.canPlayType() 函数检测 Audio 元素对不同音频的支持情况。注意 audio.canPlayType() 支持的音频返回 "maybe"、"probably"，不支持返回 ""。

```js
var audio = new Audio();
var canPlay = audio.canPlayType("audio/ogg");
if (canPlay === "maybe" || canPlay === "probably") {
    return true;
} else {
    return false;
}
```

官方的说法是，音视频编码格式比较复杂，难以保证 100% 可解。千万不要以为这是随便说说哦。对于特定比特率、特定压缩方式的音频文件，确实存在浏览器支持该格式，却无法播放的情况。所以建议音频发布先在目标环境中测试一遍。

##移动端 Audio 元素无法自动播放

移动端浏览器，主要是Safari和移动版Chrome，默认状态下 Audio 元素是不可用的，既不能自动播放，也不能缓冲，

```html
<!-- preload failed -->
<audio preload src="1.mp3"></audio>

<!-- autoplay failed -->
<audio autoplay src="2.ogg"></audio>
```

在 2007 年，第一代 iPhone 上市时，移动网络还不想今天这样廉价。音频文件动辄几百K、几M，出于节约流量的考虑，浏览器限制用户至少和网页互动一次后，网页才能激活 Audio 元素播放声音。在 JS 中访问未激活的 Audio 元素不会报错，只是全部调用没有效果，播放不出任何声音。

如何激活 Audio 元素？需要监听网页上任意的一个点击事件，并在事件处理函数中进行 audio.paly()，或别的播放、缓冲音频的操作，此后，该网页上的全部 Audio 元素都被激活。才能在非点击事件处理函数中使用 Audio 元素。

```js
var audio = new Audio();
element.addEventListener("click", function(event){
    audio.play("1.mp3");
    // do something else ...
}, false);
```

也许有人会想到在进入网页时由 JS 立即发起一个点击事件以突破限制。的确，在某些版本的 Safari 中这个方法真的有用，但Apple很快修复了这个 “BUG”。

当然，仍然可以捕获页面中的第一个点击操作，并播放一段空白的音效以激活 Audio 元素。事实上 FenQi.Engine 也是这么做的。

##移动端 Audio 元素可能是单例？

又是移动端的问题。这个问题同样集中出现于早期浏览器中，这次不止 Safari 还包括 UC、猎豹等大量国产浏览器和 Android 4.x 以前的内置浏览器。在 JS 中 Audio 元素的使用方法如下：

```js
var audio1 = new Audio("a.aac");
var audio2 = new Audio("b.aac");
audio1.play();
audio2.play();
```

通过 new 创建新的 Audio 元素，无论从语义上还是标准上来说，audio1 和 audio2 都应当是两个互不相关的元素，对 audio1 进行操作应当不对影响 audio2。不幸的是，在部分浏览器中 audio2.play(); 这一句代码会导致 audio1 播放的音频停止。

在这类浏览器中，整个页面只能同时播放一个音频文件。仿佛 audio1 和 audio2 指向同一个 Audio 对象。Audio 元素像整个网页公用的一个单例。

```js
var audio1 = new Audio();
var audio2 = new Audio();
audio1 === audio2 // => false
```

说像单例是因为并不能用上述代码检测 Audio 元素是否只能播放同一段声音。实际上你不能用 JS 代码判断当前网页中的 Audio 是否只能播放一个声音，除非用人耳。

解决方法有两个：

* 默认全部 Audio 标签都只能播放一个声音。
* 为 Audio 支持多声音的浏览器建立 UserAgrent 白名单。

## 两个 Audio 元素不能使用同一个音频文件

这个问题出现在支持两个 Audio 元素同时播放不同声音的浏览器中，算是上个问题的一个衍生问题。

```js
var audio1 = new Audio("sprite.mp3");
var audio2 = new Audio("sprite.mp3");
audio1.play();
audio2.play(); // error here
```

audio1 和 audio2 同时播放了 sprite.mp3 会导致错误，audio1 的音频停止，只有 audio2 的音频会被播放出来。这个问题主要影响拼接音效的使用，无法将多个音效放在同一个文件中，再使用不同 Audio 元素分别播放两个。

## Audio 元素播放延迟

某些移动浏览器上，无论 Audio 元素是否已经缓冲，播放时均存在长达一秒至几秒的延迟。建议不要用 Audio 元素播放音效。

## 移动端 WebAudio 无法自动播放

和 Audio 元素表象相同，建议的处理方法同样是捕获页面中的第一个点击操作，并播放一段空白的音效以激活 全部 WebAudio 操作。

## WebAudio 无法检测支持的解码类型

WebAudio 解码使用 webAudio.decodeAudioData() 函数，但 WebAudio 上没有类似 audio.canPlayType() 的函数用以检测支持的解码类型。

FenQi.Engine 的做法是默认 Audio 元素支持播放的音频格式，WebAudio也支持解码，在程序初始化时检测当前环境支持的解码类型。

```js
var support = {}
var audio = new Audio();
var result = audio.canPlayType("aac");
support["aac"] = (result === "maybe" || result === "probably");
result = audio.canPlayType("mp3");
support["mp3"] = (result === "maybe" || result === "probably");
// ...
```

## 实现音频库

以上总结了 FenQi.Engine 音频库开发过程中遇到的坑。Audio 元素的支持比较广泛，但兼容性问题极其严重，WebAudio 虽然支持有限，但兼容性较好。

FenQi.Engine 实现了一个足够满足 2D 游戏需求的音频库，针对 BGM、长音效、短音效分别做了优化处了，具体实现在下一篇文章[HTML5 音频库实现](http:/fenqi.io/program/HTML5-yin-pin-ku-shi-xian/article.html)中介绍。
