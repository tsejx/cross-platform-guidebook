---
nav:
  title: Hybrid
  order: 1
group:
  title: Hybrid
  order: 1
title: 跨语言通信方案
order: 3
---

# 跨语言通信方案

Hybrid App 的本质，其实是在原生的 App 中，使用 WebView 作为容器直接承载 Web 页面。因此，最核心的点就是 Native 端 与 H5 端 之间的双向通讯层，其实这里也可以理解为我们需要一套 **跨语言通讯方案**，来完成 Native（Java
、Objective-c 等等） 与 JavaScript 的通讯。这个方案就是我们所说的 [JSBridge](./jsbridge)，而实现的关键便是作为容器的 WebView，一切的原理都是基于 WebView 的机制。

Web 与 原生 APP 的交互，本质上来说，就是两种调用：

- APP 调用 Web 的代码
- Web 调用 APP 的代码

## APP 调用 Web

由于 Native 可以算作 H5 的宿主，因此拥有更大的权限，上面也提到了 Native 可以通过 WebView API 直接执行 JS 代码。这样的权限也就让这个方向的通讯变得十分的便捷。只需要在 Web 中曝露一些方法在全局对象上，然后在原生 APP 中调用这些挂载在全局对象上的方法就能实现 APP 与 Web 的通讯。

**Web**

```js
window.JSBridge = {
  double = val => val * 2,
  triple = val => val * 3,
};
```

**Android**

⚠️ **注意**: 当系统低于 4.4 时，`evaluateJavascript` 是无法使用的，因此单纯的使用 `loadUrl` 无法获取 JS 返回值，这时我们需要使用前面提到的 `prompt` 的方法进行兼容，让 H5 端 通过 `prompt` 进行数据的发送，客户端进行拦截并获取数据。

```java
// Android loadUrl(4.4-)
// 调用 JS 的 JSBridge.double 方法
// 该方法的弊端是无法获取函数返回值
webview.loadUrl('javascript:JSBridge.double(10)')

// Android evaluateJavascript(4.4+)
webview.evaluateJavascript('window.JSBridge.double(10)', new ValueCallback<String>() {
  @Override
  public void onReceiveValue(String value) {
    // 此处为 JS 返回的结果 -> 20
  }
});
```

二者区别：

1. `loadUrl()` 回刷新页面，`evaluateJavascript()` 则不会使页面刷新，所以 `evaluateJavascript()` 的效率更高
2. `loadUrl()` 得不到 JS 的返回值，`evaluateJavascript()` 可以获取返回值
3. `evaluateJavascript()` 在 Android 4.4 之后才可以使用

**iOS**

```cpp
NSString *func = @"window.JSBridge.double(10)";
NSString *str = [webview stringByEvaluatingJavaScriptFromString:func]; // 20
```

基于上面的原理，我们已经明白 JSBridge 最基础的原理，并且能实现 Native <=> H5 的双向通讯机制了。

## Web 调用 APP

因为 Web 不能直接访问宿主 AOO，所以这种调用就相对复杂一点。

基于 WebView 的机制和开放的 API，实现这个功能有三种常见的方案：

- 由 APP 向 Web 注入全局 API，原理其实就是 Native 获取 JavaScript 环境上下文，并直接在全局对象 `window` 上面挂载对象或者方法，使 JavaScript 可以直接调用，Android 与 iOS 分别拥有对应的挂载方式。
- WebView 中的 `prompt/console/alert` 拦截，通常使用 `prompt`，因为这个方法在前端中使用频率低，比较不会出现冲突；
- 由 Web 发起一个 WebView URL Scheme 自定义协议请求，APP 拦截这个请求后，再由 APP 调用 Web 中的回调函数

### 全局对象注入

这种方式沟通机制简单，比较好理解，并且对于 Web 来说，没有新的东西，所以是比较推荐的一种方式。但这种方式可能存在安全隐患，详细查看 [你不知道的 Android WebView 使用漏洞](https://www.jianshu.com/p/3a345d27cd42)。

**Android**

```java
// 对象映射
webview.addJavascriptInterface(new Object() {
  @JavascriptInterface
  public int double(value) {
    return value * 2;
  }

  @JavascriptInterface
  public int triple(value) {
    return value * 3;
  }
}, "jsBridge");
```

**iOS**

```cpp
@interface JSBridge : NSObject
{}
- (int) double:(int)value;
- (int) triple:(int)value;
@end

@implementation JSBridge
- (int) double:(int)value {
  return value * 2;
}
- (int) triple:(int)value {
  return value * 3;
}
@end

JSContext *context=[webview valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];

JSBridge *jsBridge = [JSBridge new];

context[@"jsBridge"] = jsBridge;
```

**Web**

```js
window.jsBridge.double(10);
// 20
```

### 自定义协议请求

#### 实现原理

在 WebView 中发出的网络请求，客户端都能进行监听和捕获。

#### 协议的定制

这是 URL 的组成：

```
[scheme:][//authority][path][?query][#fragment]
```

我们拿 `https://www.baidu.com/` 来举例，`scheme` 通信协议自然就是 `https` 了。

就像给服务器资源分配一个 URL，以便我们去访问它一样，我们同样也可以给手机 APP 分配一个特殊格式的 URL，用来访问这个 APP 或者这个 APP 中的某个功能来实现通信。APP 得有一个标识，好让我们可以定位到它，它就是 URL 的 Scheme 部分。

常见 APP 的 URL Scheme：

| APP        | 微信        | 支付宝      | 淘宝        | 微博           | QQ       | 知乎       | 短信     |
| :--------- | :---------- | :---------- | :---------- | :------------- | :------- | :--------- | :------- |
| URL Scheme | `weixin://` | `alipay://` | `taobao://` | `sinaweibo://` | `mqq://` | `zhihu://` | `sms://` |

1. `scheme://` 只是一种规则，可以根据业务进行制定，使其具有含义，例如我们定义 `scheme://` 为公司所有 APP 系通用，为通用工具协议
2. 这里不要使用 `location.href` 发送，因为其自身机制有个问题，多次修改 `location.href` 值 Native 层只能收到最后一次请求，也就是同时并发多次请求会被合并成为一次，导致协议被忽略，而并发协议其实是非常常见的功能。我们会使用创建 `iframe` 发送请求的方式。
3. 通常考虑到安全性，需要在 **客户端** 中设置域名白名单或者限制，避免公司内部业务协议被第三方直接调用。

#### 协议的拦截

客户端可以通过 API 对 WebView 发出的请求进行拦截：

- iOS: shouldStartLoadWithRequest
- Android: shouldOverrideUrlLoading

当解析到请求 URL 头为制定的协议时，便不发起对应的资源请求，而是解析参数，并进行相关功能或者方法的调用，完成协议功能的映射。

#### 协议回调

由于协议的本质其实是发送请求，这属于一个异步的过程，因此我们便需要处理对应的回调机制。这里我们采用的方式是 JS 的事件系统，这里我们会用到 `window.addEventListener` 和 `window.dispatchEvent` 这两个基础 API；

1. 发送协议时，通过协议的唯一标识注册自定义事件，并将回调绑定到对应的事件上。
2. 客户端完成对应的功能后，调用 Bridge 的 `dispatch` API，直接携带 `data` 触发该协议的自定义事件。

```js
// 业务调用 API
Bridge.getNetwork(data => {});

// Bridge 层功能
// 生成唯一标识 handler
// 注册自定义事件
// 拼接并发送协议
const handler = 1;
window.addEventListener(`getNetwork_${handler}`, callback, false);
Bridge.send(`scheme://getNetwork?handler=${handler}`);

// Native 层获取网络状态后通过 Bridge 再次传回
// 将网络状态通过事件直接触发自定义事件并传递数据
event.data = network;
window.dispatchEvent(event);
```

通过事件的机制，会让开发方式更符合我们前端的习惯，例如当你需要监听客户端的通知时，同样只需要在通过 `addEventListener` 进行监听即可。

⚠️ **注意**: 这里有一点需要注意的是，应该避免事件的多次重复绑定，因此当唯一标识重置时，需要 `removeEventListener` 对应的事件。

#### 参数传递方式

由于 WebView 对 URL 会有长度的限制，因此常规的通过 `search` 参数 进行传递的方式便具有一个问题，既 **当需要传递的参数过长时，可能会导致被截断**，例如传递 Base64 或者传递大量数据时。

因此我们需要制定新的参数传递规则，我们使用的是函数调用的方式。这里的原理主要是基于:

**Native 可以直接调用 JS 方法并直接获取函数的返回值。**

我们只需要对每条协议标记一个唯一标识，并把参数存入参数池中，到时客户端再通过该唯一标识从参数池中获取对应的参数即可。

#### 代码实现

**Web**

```js
window.bridge = {
  getDouble: value => {
    // 20
  },
  getTriple: value => {
    // more
  },
};

// 通过 iframe 实现对外请求，Native 层检测到 sdk:// 协议开头会拦截请求进行分析
const url = 'sdk://double?value=10';
const iframe = document.createElement('iframe');
iframe.style.display = 'none';
iframe.src = url;
document.body.appendChild(iframe);
setTimeout(function() {
  iframe.remove();
}, 300);
```

**Android**

```java
webview.setWebViewClient(new WebViewClient() {
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        // 判断如果 url 是 sdk:// 协议的就拦截掉
        // 然后从 url sdk://action?params 中取出 action 与params

        Uri uri = Uri.parse(url);
        if ( uri.getScheme().equals("sdk")) {

            // 比如 action = double, params = value=10
            webview.evaluateJavascript('window.bridge.getDouble(20)');

            return true;
        }
        return super.shouldOverrideUrlLoading(view, url);
    }
});
```

**iOS**

```cpp
- (BOOL)webview:(UIWebView *)webview shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
  // 判断如果 url 是 sdk:// 打头的就拦截掉
  // 然后从 url sdk://action?params 中取出 action 与params

  NSString *urlStr = request.URL.absoluteString;

  if ([urlStr hasPrefix:@"sdk://"]) {

    // 比如 action = double, params = value=10
    NSString *func = @"window.bridge.getDouble(20)";
    [webview stringByEvaluatingJavaScriptFromString:func];

    return NO;
  }

  return YES;
}
```

#### 方案实现步骤

大致需要以下几个步骤：

1. 由 App 自定义协议，比如 `sdk://action?params`
2. 在 Web 定义好回调函数，比如 `window.bridge = { getDouble: value => {}, getTriple: value => {} }`
3. 由 Web 发起一个自定义协议请求，比如 `location.href = 'sdk://double?value=10'`
4. App 拦截这个请求后，进行相应的操作，获取返回值
5. 由 App 调用 Web 中的回调函数，比如 `window.bridge.getDouble(responseValue)`

```jsx | inline
import React from 'react';
import img from '../../assets/hybird/native-and-h5.png';

export default () => <img alt="Natvie 和 H5 通讯架构图" src={img} width="520" />;
```

## JSBridge 的接入

接下来，我们来理下代码上需要的资源。实现这套方案，从上图可以看出，其实可以分为两个部分:

- **JS 部分（Bridge）**: 在 JS 环境中注入 Bridge 的实现代码，包含了 **协议的拼装/发送/参数池/回调池** 等一些基础功能。
- **Native 部分（SDK）**: 在客户端中 Bridge 的功能映射代码，实现了 **URL 拦截与解析/环境信息的注入/通用功能映射** 等功能。

通常的做法是，将这两部分一起封装成一个 Native SDK，由客户端统一引入。客户端在初始化一个 WebView 打开页面时，如果 `页面地址` 在白名单中，会直接在 HTML 的头部注入对应的 `bridge.js`。这样的做法有以下的好处：

- 双方的代码统一维护，避免出现版本分裂的情况。有更新时，只要由客户端更新 SDK 即可，不会出现版本兼容的问题；
- App 的接入十分方便，只需要按文档接入最新版本的 SDK，即可直接运行整套 Hybrid 方案，便于在多个 App 中快速的落地；
- H5 端无需关注，这样有利于将 Bridge 开放给第三方页面使用。

这里有一点需要注意的是，协议的调用，一定是需要确保执行在 `bridge.js` 成功注入后。由于客户端的注入行为属于一个附加的异步行为，从 H5 方很难去捕捉准确的完成时机，因此这里需要通过客户端监听页面完成后，基于上面的事件回调机制通知 H5 端，页面中即可通过 `window.addEventListener('bridgeReady', e => {})` 进行初始化。

## APP 中 H5 的接入

将 H5 接入 App 中通常有两种方式：

### 服务器静态资源

在线 H5，这是最常见的一种方式。我们只需要将 H5 代码部署到服务器上，只要把对应的 URL 地址 给到客户端，用 WebView 打开该 URL，即可嵌入。该方式的好处在于:

- 独立性强，有非常独立的开发/调试/更新/上线能力；
- 资源放在服务器上，完全不会影响客户端的包体积；
- 接入成本很低，完全的热更新机制。

但相对的，这种方式也有对应的缺点:

- 完全的网络依赖，在离线的情况下无法打开页面；
- 首屏加载速度依赖于网络，网络较慢时，首屏加载也较慢；

通常，这种方式更适用在一些比较轻量级的页面上，例如一些帮助页、提示页、使用攻略等页面。这些页面的特点是功能性不强，不太需要复杂的功能协议，且不需要离线使用。在一些第三方页面接入上，也会使用这种方式，例如我们的页面调用微信 JS-SDK。

### 内置静态资源包

内置包 H5，这是一种本地化的嵌入方式，我们需要将代码进行打包后下发到客户端，并由客户端直接解压到本地储存中。通常我们运用在一些比较大和比较重要的模块上。其优点是:

- 由于其本地化，首屏加载速度快，用户体验更为接近原生；
- 可以不依赖网络，离线运行；

但同时，它的劣势也十分明显:

- 开发流程/更新机制复杂化，需要客户端，甚至服务端的共同协作；
- 会相应的增加 App 包体积；

这两种接入方式均有自己的优缺点，应该根据不同场景进行选择。

---

**参考资料：**

- [Hybrid APP 技术解析：原理篇](https://juejin.im/post/6844903640520474637)
- [Hybrid APP 技术解析：实战篇](https://juejin.im/post/6844903648510607373)
- [H5 与原生 APP 交互的原理](https://segmentfault.com/a/1190000016759517)
