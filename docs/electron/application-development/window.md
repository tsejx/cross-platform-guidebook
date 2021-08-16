---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 窗口管理
order: 3
---

# 窗口管理

## 创建窗口

通过 [BrowserWindow](https://www.electronjs.org/docs/api/browser-window) 模块来 **创建** 或者 **管理** 新的浏览器窗口，每个浏览器窗口都有一个进程来管理。

```js
// Main Process
const { BrowserWindow } = require('electron');

const win = new BrowserWindow({
  width: 800,
  height: 600,
});

// Load a remote URL
win.loadURL('https://github.com');

// Or load a local HTML file
win.loadURL(`file://${__dirname}/app/index.html`);
```

问题：Electron 的 `BrowserWindow` 模块在创建时，如果没有配置 `show: false`，在创建之时就会显示出来，且默认的背景是白色；然后窗口请求 HTML，会出现视觉闪烁。

```js
const { BrowserWindow } = require('electron');
const win = new BrowserWindow({ show: false });

win.loadURL('https://github.com');

win.on('ready-to-show', () => {
  win.show();
});
```

其他常用属性：

| 常用属性                          | 说明                                                                                                     | 相关文档                                                           |
| :-------------------------------- | :------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------- |
| `preload`                         | 为渲染进程加载页面注入脚本，无论是否集成 Node.js，此脚本都可以访问所有 Node API 脚本路径为文件的绝对路径 |                                                                    |
| `webPreferences.webSecurity`      | 禁用同源策略                                                                                             |                                                                    |
| `webPreferences.contextIsolation` | 是否开启沙箱模式                                                                                         |                                                                    |
| `frame`                           | 无边框窗口                                                                                               | [无边框窗口](https://www.electronjs.org/docs/api/frameless-window) |

### 管理窗口

所谓的管理窗口，相当于主进程可以干预窗口多少。

- 窗口的路由跳转
- 窗口打开新的窗口
- 窗口大小、位置等
- 窗口的显示
- 窗口类型（无边框窗口、父子窗口）
- 窗口内 JavaScript  的 node  权限，预加载脚本等
- ....

这些个方法都存在于 BrowserWindow 模块中。

#### 管理应用创建的窗口

BrowserWindow 模块在创建窗口时，会返回 **窗口实例**，这些 **窗口实例** 上有许多功能方法，我们利用这些方法，管理控制这个窗口。

在这里使用 `Map` 对象来存储这些 窗口实例。

```ts
const BrowserWindowsMap = new Map<number, BrowserWindow>();
let mainWindowId: number;

const browserWindows = new BrowserWindow({ show: false });
browserWindows.loadURL('https://github.com');
browserWindows.once('ready-to-show', () => {
  browserWindows.show();
});
BrowserWindowsMap.set(browserWindow.id, browserWindow);

// 记录当前窗口为主窗口
mainWindowId = browserWindow.id;
```

**窗口被关闭**，得把 `Map` 中的实例删除。

```js
browserWindow.on('closed', () => {
  BrowserWindowsMap?.delete(browserWindowID);
});
```

#### 管理用户创建的窗口

主进程可以控制窗口许多行为，这些行为会在后续文章一一列举；以下以主进程控制窗口建立新窗口的行为为例。

使用 `new-window` 监听新窗口创建

```js
// 创建窗口监听
browserWindow.webContents.on('new-window', (event, url, frameName, disposition) => {
  /** @params {string} disposition
  *  new-window : window.open调用
  *  background-tab: command+click
  *  foreground-tab: 右键点击新标签打开或点击a标签target _blank打开
  * /
})
```

> 关于 `disposition` 字段的解释，移步 [Electron 文档](https://www.electronjs.org/docs/api/web-contents#webcontents)、[Electron 源码](https://github.com/electron/electron/blob/72a089262e31054eabd342294ccdc4c414425c99/shell/browser/api/electron_api_web_contents.cc)、[Chrome 源码](https://chromium.googlesource.com/chromium/src/+/66.0.3359.158/ui/base/mojo/window_open_disposition_struct_traits.h)

扩展 `new-window`：经过实验，并不是所有新窗口的建立，`new-window` 都能捕捉到的。

以下方式打开的窗口可以被 `new-window1 事件捕捉到：

```js
window.open('https://gibhub.com');
```

```html
<a href="https://github.com" target="__blank">链接</a>
```

渲染进程中使用 BrowserWindow 创建新窗口，不会被 `new-window` 事件捕捉到。

```js
const { BrowserWindow } = require('electron');

const win = new BrowserWindow();
win.loadURL('https://github.com');
```

应用 `new-window` 控制着窗口新窗口的创建，我们利用这点，可以做到很多事情；比如链接校验、浏览器打开链接等等。默认浏览器打开链接代码如下：

```js
import { shell } from 'electron'
function openExternal(url: string) {
  const HTTP_REGEXP = /^https?:\/\//
  // 非http协议不打开，防止出现自定义协议等导致的安全问题
  if (!HTTP_REGEXP) {
    return false
  }
  try {
    await shell.openExternal(url, options)
    return true
  } catch (error) {
    console.error('open external error: ', error)
    return false
  }
}
// 创建窗口监听
browserWindow.webContents.on('new-window', (event, url, frameName, disposition) => {
  if (disposition === 'foreground-tab') {
      // 阻止鼠标点击链接
      event.preventDefault()
      openExternal(url)
  }
})
```

[关于 Shell 模块](https://www.electronjs.org/docs/api/shell)

## 关闭窗口

## 主窗口隐藏和恢复

## 窗口的聚焦和失焦

## 父子窗口

## 模态窗口

## 窗口标题栏和边框

### 自定义窗口的标题栏

```js
const win = new BrowserWindow({
  // 去除标题栏
  frame: false,
  webPreferences: {
    nodeIntegration: true,
  },
});
```

系统标题栏去除后，啾无法进行最大化、还原、最小化、关闭、拖动等操作了，因此我们要自己做拖动和最大最小化。

图懂只需要给允许拖动的区域，比如某个 `div`，加上样式属性 `-webkit-app-region: drag` 即可。如果希望 `div` 下某个子元素又不可拖动，可以设置父 `div` 设置为 `drag`，在这个 `div` 内又有子元素系，希望在这个子元素上不能拖动，可以设置属性 `-webkit-app-region: no-drag`。并切父 `div` 下入锅嵌套多层子孙后代 `div`，每层都要设置 `no-drag` 才能见效。

但是 `drag` 有个问题，就是增加 `drag` 后的区域会导致可拖动区域的 `mouseover` 事件出问题。

### 窗口的控制按钮

直接禁止页面刷新

```ts
window.onkeydown = function (e: KeyboradEvent) {
  if (
    (e.ctrlKey && e.keyCode === 82) ||
    (e.metaKey && e.keyCode === 82) ||
    (e.ctrlKey && e.keyCode === 116)
  ) {
    return false;
  }
};
```

如果允许用户刷新，最大化状态值可以保存到 LocalStorage 中。如果每次重启应用要做到恢复重启之前的状态，那就需要把重启前的状态信息保存到本地。

### 适时地显示窗口

如果通过 BrowserWindow 的 `x` 和 `y` 属性调整了窗口位置，你会发现启动程序后并不是直接显示在指定位置，而是先显示在默认中央位置，然后一下跳过去指定位置，这种体验不太好，解决办法是在创建窗口的时候先不显示。

```js
const win = new BrowserWindow({
  show: false,
  x: 100,
  y: 100,
});
```

把显示窗口的代码放到最后，这样当窗口大小、位置都准备好之后再显示。注意，如果调用 `win.maxinize()` 语句时窗口是隐藏的也会变成显示，因为他具有 `show` 的作用。当然，还要注意不要在 `show` 出窗口前处理大量阻塞业务，这会导致窗口迟迟显示不出来，用户体验下滑。

## 不规则窗口

默认创建的窗体都是矩形，我们现在要搞一些不规则窗体，比如搞个小悬浮球。

首先，把窗口宽高都设置为一样，即一个正方形。然后把窗口的透明属性（transparent）设置为 `true`，这样你的画布就是一个透明正方形，然后你在这里面通过 `div + css` 布局就能搞出不规则窗体了。不规则窗体往往需要自定义窗体边框和标题，所以 `frame` 属性也设置为 `false`。然后把 `resizable` 和 `maximizable` 属性也设置为 `false`，不让他最大化和拖动改变大小。

```js
const win = new BrowserWindow({
  width: 80,
  height: 80,
  transparent: true,
  frame: false,
  resizable: false,
  maximizable: false,
});
```

通过设置 CSS 属性：`-webkit-app-region: drag` 可以让小圆球进行拖动。

## 窗口控制

### 阻止窗口关闭

当关闭窗口时弹出提示，问用户是否要关闭，以防止重要操作中误点关闭按钮，在网页开发时。

```js
window.onbeforeunload = function () {
  return false;
};
```

当用户关闭网页时浏览器就会发出一个警告的 `confirm` 框。 在 Electron 中我们可以使用 `onbeforeunload` 来阻止窗口关闭，但是不是不会弹出提示框，也不能在 `onbeforeunload` 事件中用 `alert` 或 `confirn` 来提示。但是可以在 `onbeforeunload` 事件中操作 DOM，比如弹出一个 `div` 来提示用户，当用户点击确定关闭时再关闭。

### 多窗口竞争资源

JavaScript 是单线程执行的事件驱动型语言，在同一个渲染进程中发起多个请求操作同一个文件是不会出问题的（必须使用 Node.js 的 `fs.writeFileSync` 同步方法或者控制好异步回调执行的顺序）。

如果开多个窗口在编辑同一个本地文本文件就会导致写入覆盖。

有两种解决方法：

1. 两个渲染进程之间通过消息通信来保证读写操作顺序，窗口 A 修改内容保存后立马发消息给窗口 B 通知他更新内容。

2. 用 Node.js 提供的 `fs.watch` 来监视文件变化，一旦文件发生改变则加载最新的文件。当需要写数据的时候把内容通过管道阀消息给主进程让主进程去写，这样做的好处是利用了 JavaScript 单线程执行的特性，主进程接收消息有一定的顺序，不管开多少个窗口都可以保证按先来先存的顺序写文件。

第二种方式更推荐，因为这个文件不一定只有你在编辑，比如我在 WebStorm 里面编辑了代码，在 Nodpad++ 中打开它就会提示你有新内容，是否覆盖，这就是通过文件变动检测来实现。

Node.js 提供了两个监控文件变化的 API：`fs.watch` 和 `fs.watchFile`，前者更高效。

## 参考资料

- [Electron Playground 系列 - 窗口篇](https://juejin.cn/post/6888907382806544392)
- [Electron 窗口关闭与托盘处理](https://segmentfault.com/a/1190000039386209)
