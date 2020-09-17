---
nav:
  title: Hybrid
  order: 1
group:
  title: Hybrid
  order: 1
title: JSBridge
order: 5
---

# JSBridge

## 实现形式

## 安全问题

在 APP 内 JSBridge 可以实现 Web 和 Native 的通信，但是如果 APP 打开一个恶意的页面，页面可以任意调用 JSBridge 方法，获取各种隐私的数据，就会引起安全问题。

JSBridge 的安全问题的解决方式有两种：

1. 通过 Native 进行白名单配置，通过 Server 云端授权（Server 的云端授权这块，放到后续 JSSDK 的设计部分进行详细讲解）
2. 通过 Native 的方式来控制 JSBridge 的安全。

### 云端授权

### 限定域名权限

假设 JSBridge 的协议格式如下：

```js
hybrid://action/method?arg1=xxx&arg2=xxx
```

可以通过下面方式进行安全设置：

1. 配置某些方法的使用范围，比如固定的 Webview，固定的 `domain`
2. 通过正则来设置细化的权限，比如：
   - `baidu.com` 网页可以使用 `*`
   - `taobao.com` 可以使用：`hybrid://taobao/*`

## 最佳实践

> 什么是最佳实践的 JSBridge 呢？

结合文章内容，要求 JSBridge 做到以下几点：

- 官方认可，符合规范
- 跨平台通用
- APP 内和 APP 外规范通用
- 安全可靠
- 约定大于配置的原则

综合上文介绍的内容，JSBridge 的最佳实践是：

1. 协议规范都使用：`hybrid://action/method?arg1=xxx&arg2=xxx`
2. iOS 使用 Universal Link 和 UIWebview 的 `delegate`
3. 安卓使用 `shouldOverrideUrlLoading` 和 Applink

### 规范和约定

先贴个 URL scheme 的图片，理解下 URL 的组成部分：

约定我们的规范如下：

```plain
yourappscheme://module/action?arg1=x&arg2=x&ios_version=xxx&andr_version=xxx&upgrade=1/0&callback=xxx&sendlog=1/0
```

- 整体小写
- `yourappscheme`：就是你的 scheme，可辨识，别冲突，通过这个可以进行 Universal Link 和 Applink 的分发
- `module` 和 `action`：某个模块组件的某个方法
- `?` 后面是 `querystring`，这里预定了几个特殊的参数：
  - `ios_version/andr_version`：非必须，iOS 和安卓的最小版本，即本协议从哪个版本开始支持的，低版本不支持则忽略，配合 upgrade 使用进行 APP 升级
  - `upgrade`：是否强制升级，即当版本低于设置的 ios/andr_version 是否弹出提示用户升级的对话框（yourappscheme 已经可以调起 app，只不过功能可能因为版本低不支持，这时候可以引导用户升级）
  - `callback`：异步回调函数，下面详细树下 callback 的最佳实践
  - `sendlog`：调起后是否打点发送日志

示例：

```plain
// 账号相关
// 打开用户个人主页
fb://account/userprofile?id=xxx

// 打开登录界面
fb://account/login?callback=xxx

// 工具类
// 获取定位
fb://utils/getgeolocation?callback=xx
```

### 回调方法设计

当 Native 操作成功之后，会将处理结束后的结果或者数据通过 `callback` 回调传给 Web，当然有成果就又失败，`callback` 的参数设计有两种方式：

#### 错误优先

即下面的回调方法格式：

```js
function callback(error, data) {
  if (error) {
    throw error;
  }
  console.log(data);
}
```

#### JSON 结构

即回调方法只接收一个 JSON 对象，JSON 格式如下：

```json
{
  "error_code": 0,
  "data": {}
}
```

### 预留升级/日志能力

做 APP 开发经常会遇见下面的问题：

1. 功能/端能力是从某个版本开始的，低版本用不了，但是 `scheme` 还是会调起 APP。
2. 对于低版本，PM 希望提示用户升级
3. 统计调起成功率，分发次数之类的统计需求

`scheme` 的 `querystring` 部分由 `ios_version/andr_version` 和 `upgrade` 这三个成对的参数，可以解决升级问题，`sendlog` 解决日志统计问题。

- `ios_version/andr_version`：是标示该协议的最低支持版本，如果低于这个版本可能因为功能并未实现而能识别。
- `upgrade`：是是否强制低版本弹出升级对话框
- `sendlog`：当为 1 的时候，则发送调起成功失败之类的统计需求

## 简易代码实现

简单封装下 JSBridge 调用的方法，参数如下：

- `module`：类名称，如果 `account`
- `action`：具体操作方法，如 `login`
- `args`：非必须，协议参数，支持 `string` 和对象
- `callback`：非必须，回调单独提出来，方便全局方法命名

具体代码如下

```js
function invoke(module, action, args, callback) {
  let scheme = `yourappscheme://${module}/${action}?`;
  if (isFunction(args)) {
    callback = args;
    args = null;
  }
  // 处理下参数
  if (isString(args)) {
    scheme += args;
  } else if (isObject(args)) {
    each(args, (k, v) => {
      if (isObject(v) || isArray(v)) {
        v = JSON.stringify(v);
      }
      scheme += `${k}=${v}`;
    });
  }
  // callback 独立传，方便全局函数名命名
  if (isFunction(callback)) {
    var funcName = '_jsbridge_cb_' + getId();
    window[funcName] = function() {
      callback.apply(window, [].slice.call(arguments, 0));
    };
    scheme += (!~scheme.indexOf('?') ? '&' : '?') + `callback=${funcName}`;
  }

  if (os.ios && versionCompare(os.version, '9.0') >= 0) {
    window.location.href = scheme;
  } else {
    var $node = document.createElement('iframe');
    $node.style.display = 'none';
    $node.src = scheme;
    var body = document.body || document.getElementsByTagName('body')[0];
    body.appendChild($node);
    setTimeout(function() {
      body.removeChild($node);
      $node = null;
    }, 10);
  }
}
```

---

**参考资料：**

- [Hybrid App 技术解析 -- 实战篇](https://juejin.im/post/6844903648510607373)
- [Hybird 开发：JSBridge - Web 和客户端双向通信](https://js8.in/2017/03/16/Hybrid%20%E5%BC%80%E5%8F%91%EF%BC%9AJsBridge%20-%20Web%E5%92%8C%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%8F%8C%E5%90%91%E9%80%9A%E4%BF%A1/)
