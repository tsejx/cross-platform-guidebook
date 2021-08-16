---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 主进程模块
order: 1
---

# 主进程模块

一个 Electron 应用只有一个主进程，但可以有多个渲染进程。一个 BrowserWindow 实例就代表着一个渲染进程。当 BrowserWindow 示例被销毁后，渲染进程也跟着终结。

主进程负责管理所有的窗口及其对应的渲染进程。每个渲染进程都是独立的，它只关心所运行的 Web 页面。在开启 nodeIntegration 配置后，渲染进程也有能力访问 Node.js 的 API。

在 Electron 中，GUI 相关的模块仅在主进程中可用。如果想在渲染进程中完成创建窗口、创建菜单等操作，可以让渲染进程给主进程发送消息，主进程接收到消息后再完成相应的操作：也可以通过渲染进程的 `remote` 模块来完成相应操作。这两种方法背后实现的机制是一样的。

Electron 框架内置的主要模块归属情况

| 归属情况     | 模块名                         |
| :----------- | :----------------------------- |
| 主进程模块   | app、autoUpdate、BrowserView、BrowserWindow、contenttTracing、dialog、globalShortcut、ipcMainMenu、MenuItem、net、netLog、Notification、powerMonitor、powerSaveBlocker、protocol、screen、session、systemPreferences、TouchBar、Tray、webContents |
| 渲染进程模块 | desktopCapturer、ipcRenderer、remote、webFrame                               |
| 公用模块     |  clipboard、crashReporter、nativeImage、shell                              |

## MainProcess

- BrowserView - 创建和控制视图
- BrowserWindow - 创建和控制浏览器窗口
- app - 控制应用程序的事件生命周期
- autoUpdater - 使应用程序能够自动更新
- ipcMain - 从主进程到渲染进程的异步通信
- clipboard
- contentTracing - 从 Chromium 收集跟踪数据，以发现性能瓶颈和缓慢的操作
- crashReporter
- desktopCapturer
- dialog
- globalShortcut - 当应用程序没有键盘焦点时检测键盘事件
- inAppPurchase
- ipcMain
- MessageChannelMain
- MessagePortMain
- nativeImage
- nativeTheme
- net - 使用 Chromium 的本机网络库发送 HTTP/HTTPS 请求
- netLog - 记录网络日志
- Notification - 弹出系统通知的组件
- powerMonitor - 监控电源状态的更新事件
- powerSaveBlocker - 避免系统进入低电量（睡眠）模式
- process - 进程相关信息及方法
- protocol - 注册自定义协议并拦截现有协议请求
- screen - 检测关于屏幕大小、显示、光标位置等信息
- session - 管理浏览器会话、Cookie、缓存、代理设置等
- shell - 使用默认应用程序管理文件和 URL
- systemPreferences - 获取系统配置
- webContents
- webFrameMain
- TouchBar
- Menu
- Tray
- ShareMenu - 用于在 macOS 中创建 [Share Menu Extensions](https://developer.apple.com/design/human-interface-guidelines/macos/extensions/share-extensions/)

## contentTracing

此模块不包括 Web 界面，请使用 trace viewer，剋月从以下网址获得 `chrome//tracing`。

```js
const { app, contentTracing } = require('electron');

app.whenReady().then(() => {
  (async () => {
    await contentTracing.startRocording({
      included:_categories: ['*']
    });

    console.log('Tracing started')

    await new Promise(resolve => setTimeout(resolve, 5000));

    const path = await contentTracing.stopRecording();

    console.log('Tracing data record to ' + path);
  })
})
```

## protocol

```js
const { app, protocol } = require('electron');
const path = require('path');

app.whenReady().then(() => {
  protocol.registerFileProtocol('atom', (request, callback) => {
    const url = request.url.substr(7);
    callback({ path: path.normalize(`${__dirname}/${url}`) });
  });
});
```

## webContents

`webContents` 是一个 EventEmitter，它负责呈现和控制网页，是 BrowerWindow 对象的一个属性。

```js
const { BrowserWindow } = require('electron');

const win = new BrowserWindow({ width: 800, height: 1500 });
win.loadURL('http://github.com');

const contents = win.webContents;
console.log(contents);
```

## webFrameMain

`webFrameMain` 用于控制网页和 iFrames。

```js
const { BrowserWindow, webFrameMain } = require('electron');

const win = new BrowserWindow({ width: 800, height: 1500 });
win.loadURL('https://your-website.com');

win.webContents.on(
  'did-frame-navigate',
  (event, url, isMainFrame, frameProcessId, frameRoutingId) => {
    const frame = webFrameManin.fromId(frame);
    if (frame) {
      const code = 'document.body.innerHTML = document.body.innerHTML.replaceAll("heck", "h*ck")';

      frame.executeJavaScript(code);
    }
  }
);
```
