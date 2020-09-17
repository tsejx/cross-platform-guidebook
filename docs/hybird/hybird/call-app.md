---
nav:
  title: Hybrid
  order: 1
group:
  title: Hybrid
  order: 1
title: 唤端技术方案
order: 4
---

# 唤端技术方案

唤端技术：

- intent
- localserver
- scheme：deep link / applink / Universal Link
- smart app banner

## 唤端媒介

### URL Scheme

我们的手机上有许多私密信息，联系方式、照片、银行卡信息等。我们不希望这些信息可以被手机应用随意获取到，信息泄露的危害甚大。所以，如何保证个人信息在设备所有者知情并允许的情况下被使用，是智能设备的核心安全问题。
对此，苹果使用了名为 SandBox（沙盒）的机制：应用只能访问它声明可能访问的资源。但沙盒也阻碍了应用间合理的信息共享，某种程度上限制了应用的能力。

因此，我们急需要一个辅助工具来帮助我们实现应用通信，URL Scheme 就是这个工具。

#### 组成部分

我们来看一下 URL 的组成：

```
[scheme:][//authority][path][?query][#fragment]
```

我们拿 `https://www.baidu.com` 来举例，`scheme` 通信协议自然就是 `https` 了。

就像给服务器资源分配一个 URL，以便我们去访问它一样，我们同样也可以给手机 APP 分配一个特殊格式的 URL，用来访问这个 APP 或者这个 APP 中的某个功能来实现通信。APP 得有一个标识，好让我们可以定位到它，它就是 URL 的 Scheme 部分。

常见 APP 的 URL Scheme：

| APP        | 微信        | 支付宝      | 淘宝        | 微博           | QQ       | 知乎       | 短信     |
| :--------- | :---------- | :---------- | :---------- | :------------- | :------- | :--------- | :------- |
| URL Scheme | `weixin://` | `alipay://` | `taobao://` | `sinaweibo://` | `mqq://` | `zhihu://` | `sms://` |

#### 语法

上面表格中都是最简单的用于打开 APP 的 URL Scheme，下面才是我们常用的 URL Scheme 格式：

```
     行为(应用的某个功能)
            |
scheme://[path][?query]
   |               |
应用标识       功能需要的参数
```

### Intent（Android）

安卓的原生谷歌浏览器自从 Chrome25 版本开始对于唤端功能做了一些变化，URL Scheme 无法再启动 Android 应用。 例如，通过 `iframe` 指向 `weixin://`，即使用户安装了微信也无法打开。所以，APP 需要实现谷歌官方提供的 `intent:` 语法，或者实现让用户通过自定义手势来打开 APP，当然这就是题外话了。

#### 语法

```
intent:
   HOST/URI-path // Optional host
   #Intent;
      package=[string];
      action=[string];
      category=[string];
      component=[string];
      scheme=[string];
   end;
```

如果用户未安装 APP，则会跳转到系统默认商店。当然，如果你想要指定一个唤起失败的跳转地址，添加下面的字符串在 `end;` 前就可以了:

```
S.browser_fallback_url=[encoded_full_url]
```

示例：

下面是打开 Zxing 二维码扫描 APP 的 intent。

```
intent:
   //scan/
   #Intent;
      package=com.google.zxing.client.android;
      scheme=zxing;
   end;
```

打开这个 APP ，可以通过如下的方式：

```html
<a
  href="intent://scan/#Intent;scheme=zxing;package=com.google.zxing.client.android;S.browser_fallback_url=http%3A%2F%2Fzxing.org;end"
>
  Take a QR code
</a>
```

### Universal Link（iOS）

[Universal Link](../native/universal-link.md) 是苹果在 WWDC2015 上为 iOS9 引入的新功能，通过传统的 HTTP 链接即可打开 APP。如果用户未安装 APP，则会跳转到该链接所对应的页面。

传统的 Scheme 链接有以下几个痛点：

- 在 iOS 上会有确认弹窗提示用户是否打开，对于用户来说唤端，多出了一步操作。若用户未安装 APP ，也会有一个提示窗，告知我们 `打不开该网页，因为网址无效`
- 传统 Scheme 跳转无法得知唤端是否成功，Universal Link 唤端失败可以直接打开此链接对应的页面
- Scheme 在微信、微博、QQ 浏览器、手百中都已经被禁止使用，使用 Universal Link 可以避开它们的屏蔽（ 截止到 18 年 8 月 21 日，微信和 QQ 浏览器已经禁止了 Universal Link，其他主流 APP 未发现有禁止 ）

> 如何让 APP 支持 Universal Link

有大量的文章会详细的告诉我们如何配置，你也可以去看[官方文档](https://developer.apple.com/library/archive/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW2)，我这里简单地描述实现须知。

1. 拥有一个支持 HTTPS 的域名
2. 在 开发者中心 ，Identifiers 下 AppIDs 找到自己的 App ID，编辑打开 Associated Domains 服务。
3. 打开工程配置中的 Associated Domains ，在其中的 Domains 中填入你想支持的域名，必须以 `applinks:` 为前缀
4. 配置 `apple-app-site-association` 文件，文件名必须为 `apple-app-site-association`，不带任何后缀
5. 上传该文件到你的 HTTPS 服务器的 `根目录` 或者 `.well-known` 目录下

#### Universal Link 配置中的坑

这里放一下我们在配置过程中遇到的坑，当然首先你在配置过程中必须得严格按照上面的要求去做，尤其是加粗的地方。

1. 跨域问题

IOS 9.2 以后，必须要触发跨域才能支持 Universal Link 唤端。

IOS 那边有这样一个判断，如果你要打开的 Universal Link 和 当前页面是同一域名，iOS 尊重用户最可能的意图，直接打开链接所对应的页面。如果不在同一域名下，则在你的 APP 中打开链接，也就是执行具体的唤端操作。

2. Universal Link 是空页面

Universal Link 本质上是个空页面，如果未安装 APP，Universal Link 被当做普通的页面链接，自然会跳到 404 页面，所以我们需要将它绑定到我们的中转页或者下载页。

## 触发方式

通过前面的介绍，我们可以发现，无论是 URL Scheme 还是 Intent 或者 Universal Link ，他们都算是 URL ，只是 URL Scheme 和 Intent 算是特殊的 URL。所以我们可以拿使用 URL 的方法来使用它们。

### iframe

```html
<iframe src="weixin://qrcode"></iframe>
```

在只有 URL Scheme 的日子里，`iframe` 是使用最多的了。因为在未安装 APP 的情况下，不会去跳转错误页面。但是 `iframe` 在各个系统以及各个应用中的兼容问题还是挺多的，不能全部使用 URL Scheme。

### 链接标签

```html
<a href="intent://scan/#Intent;scheme=zxing;package=com.google.zxing.client.android;end"">扫一扫</a>
```

使用过程中，对于动态生成的 `a` 标签，使用 `dispatch` 来模拟触发点击事件，发现很多种 `event` 传递过去都无效；使用 `click()` 来模拟触发，部分场景下存在这样的情况，第一次点击过后，回到原先页面，再次点击，点击位置和页面所识别位置有不小的偏移，所以 `Intent` 协议从 `a` 标签换成了 `window.location`。

### window.location

URL Scheme 在 iOS 9+ 上诸如 safari、UC、QQ 浏览器中，`iframe` 均无法成功唤起 APP，只能通过 `window.location` 才能成功唤端。

当然，如果我们的 APP 支持 Universal Link，iOS 9+ 就用不到 URL Scheme 了。而 Universal Link 在使用过程中，我发现在 QQ 中，无论是 `iframe` 导航 还是 `a` 标签打开 又或者 `window.location` 都无法成功唤端，一开始我以为是 QQ 和微信一样禁止了 Universal Link 唤端的功能，其实不然，百般试验下，通过 `top.location` 唤端成功了。

## 唤端结果检测

如果唤端失败（APP 未安装），我们总是要做一些处理的，可以是跳转下载页，可以是 iOS 下跳转 App Store。但是 JS 并不能提供给我们获取 APP 唤起状态的能力，Android Intent 以及 Universal Link 倒是不用担心，它们俩的自身机制允许它们唤端失败后直接导航至相应的页面，但是 URL Scheme 并不具备这样的能力，所以我们只能通过一些很 Hack 的方式来实现 APP 唤起检测功能。

```js
// 一般情况下是 visibilitychange
const visibilityChangeProperty = getVisibilityChangeProperty();
const timer = setTimeout(() => {
  const hidden = isPageHidden();
  if (!hidden) {
    cb();
  }
}, timeout);

if (visibilityChangeProperty) {
  document.addEventListener(visibilityChangeProperty, () => {
    clearTimeout(timer);
  });

  return;
}

window.addEventListener('pagehide', () => {
  clearTimeout(timer);
});
```

APP 如果被唤起的话，页面就会进入后台运行，会触发页面的 `visibilitychange` 事件。如果触发了，则表明页面被成功唤起，及时调用 `clearTimeout`，清除页面未隐藏时的失败函数（`callback`）回调。
当然这个事件是有兼容性的，具体的代码实现时做了事件是否需要添加前缀（比如 `-webkit-` ）的校验。如果都不兼容，我们将使用 `pagehide` 事件来做兜底处理。

没有微信白名单，只能到应用宝，我们是友好地提醒用户用浏览器打开然后唤醒 APP。

---

**参考资料：**

- [H5 唤起 APP 指南](https://juejin.im/post/6844903664155525127)
- [唤起 APP 在转转的实践](https://mp.weixin.qq.com/s/TdaIZbHR0-7NBK1LFR4nRQ)
