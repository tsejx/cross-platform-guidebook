---
nav:
  title: Electron
  order: 5
group:
  title: 主进程模块
  order: 2
title: Main Process
order: 1
---

# Main Process 模块

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
