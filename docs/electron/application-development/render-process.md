---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 渲染进程模块
order: 2
---

# Renderer Process 模块

- clipboard
- contextBridge
- crashReporter
- desktopCapturer
- ipcRenderer
- nativeImage
- webFrame

## clipboard

[clipboard](https://www.electronjs.org/docs/latest/api/clipboard) 模块能用于在系统剪贴板上执行复制和粘贴操作。

- 文本字符串
- HTML
- 图片 Image
- RTF
- bookmark
- Buffer

详细 API 参考：[clipboard API](https://www.electronjs.org/docs/latest/api/clipboard)

## contextBridge

[contextBridge](https://www.electronjs.org/docs/latest/api/context-bridge) 模块能用于在独立的上下文中创建一个安全、双向、同步的桥接器。

`Main World` 是主渲染器代码运行的 JavaScript 上下文。默认情况下，加载到渲染器中的页面将执行此处的代码。

`Isolated World` 当在 WebPreferences 中启动 contextIsolation，预加载脚本将在此处空间中运行，你可以在安全文档中阅读有关上下文隔离及其影响的更多信息。

```js
// Preload（Isolated World）
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('electron', {
  doThing: () => ipcRenderer.send('do-some-thing'),
  myPromises: [Promise.resolve().Promise.reject(new Error('whoops'))],
  anAsyncFunction: async () => 123,
  data: {
    myFlags: ['a', 'b', 'c'],
    bootTime: 1234,
  },
  nestedAPI: {
    evenDeeper: {
      youCanDoThisAsMuchAsYouWant: {
        fn: () => ({
          returnData: 123,
        }),
      },
    },
  },
});
```

```js
// Renderer（Main World）
window.electron.doThing();
```

### 暴露 Node 全局标记

预加载脚本可以使用 `contextBridge` 为渲染器提供对节点 API 的访问权限。上面描述的支持类型表也适用于通过 `contextBridge` 暴露的 Node API。请注意，许多节点 API 授予对本地系统资源的访问权。对于向不受信任的远程内容公开哪些全局和 API，要非常谨慎。

```js
const { contextBridge } = require('eelectron');
const crypto = require('crypto');

contextBridge.exposeInMainWorld('nodeCrypto', {
  sha256sum(data) {
    const hash = crypto.createHash('sha256');
    hash.update(data);
    return hash.digest('hex');
  },
});
```

## crashReporter

`crashReporter` 模块用于向远程服务器提交奔溃报告。

```js
const { crashReporter } = require('electron');

crashReporter.start({ submitURL: 'https://your-domain.com/url-to-submit' });
```

你可以使用以下项目接收崩溃报告：

- [socorro](https://github.com/mozilla/socorro)
- [mini-breakpad-server](https://github.com/electron/mini-breakpad-server)

或者使用第三方解决方案：

- [Backtrace](https://backtrace.io/electron/)
- [Sentry](https://docs.sentry.io/clients/electron)
- [BugSplat](https://www.bugsplat.com/docs/platforms/electron)

## desktopCapturer

使用 `ngvigator.mediaDevices.getUserMedia` API 访问有关可用于从桌面捕捉音频和视频的媒体源的信息。

```js
// In the renderer process.
const { desktopCapturer } = require('electron');

desktopCapturer.getSources({ types: ['window', 'screen'] }).then(async (sources) => {
  for (const source of sources) {
    if (source.name === 'Electron') {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({
          audio: false,
          video: {
            mandatory: {
              chromeMediaSource: 'desktop',
              chromeMediaSourceId: source.id,
              minWidth: 1280,
              maxWidth: 1280,
              minHeight: 720,
              maxHeight: 720,
            },
          },
        });
        handleStream(stream);
      } catch (e) {
        handleError(e);
      }
      return;
    }
  }
});

function handleStream(stream) {
  const video = document.querySelector('video');
  video.srcObject = stream;
  video.onloadedmetadata = (e) => video.play();
}

function handleError(e) {
  console.log(e);
}
```

## ipcRenderer

`ipcRenderer` 模块是一个 EventEmitter，它提供了一些方法，因此你可以将同步和异步消息从呈现进程（网页）发送到主进程，还可以从主进程接收回复。

| 方法                                                    | 说明                                                                                                 |
| :------------------------------------------------------ | :--------------------------------------------------------------------------------------------------- |
| `ipcRenderer.on(channel, listener)`                     | 监听 `channel` 频道，当有新消息到达 `listener` 时，会执行 `listener(event, args...)`                 |
| `ipcRenderer.once(channel, listener)`                   | 为事件添加一个一次性的 `listener`，只有在下一次将消息发送到通道时才会调用此 `listener`，然后将其删除 |
| `ipcRenderer.removeListener(channel, listener)`         | 从指定 `channel` 的 `listner` 队列中删除指定的 `listener`                                            |
| `ipcRenderer.removeAllListeners(channel)`               | 移除指定 `channel` 所有 `listener`                                                                   |
| `ipcRenderer.send(chanel, ...args)`                     | 通过 `channel` 向主进程发送异步消息并带上参数                                                        |
| `ipcRenderer.invoke(channel, ...args)`                  | 通过 `channel` 向主进程发送消息，并异步等待结果                                                      |
| `ipcRenderer.sendSync(channel, ...args)`                | 通过 `channel` 向主进程发送消息，并同步等待结果                                                      |
| `ipcRenderer.postMessage(channel, message, [transfer])` | 通过 `channel` 向主进程发送消息，可以选择转移零个或多个 MessagePort 对象的所有权                     |
| `ipcRenderer.sendTo(webContentsId, channel, ...args)`   | 通过 `channel` 向具有 `webContentId` 的窗口发送消息                                                  |
| `ipcRenderer.sendToHost(channel, ...args)`              | 与 `send` 类似，但事件将发送到主页面的 `<webview>` 元素，而不是主进程                                |

`ipcRenderer.send`、`ipcRenderer.invoke` 方法发送消息携带的参数都将使用结构化克隆算法序列化，就像 `window.postMessage` 一样，所以原型链将不包括在内（发送 Function、Promise、Symbol、WeakMap 或 WeakSet 会引发异常）。

```js
// Renderer Process
ipcRenderer.invoke('some-name', someArgument).then((result) => {
  // do something
});

// Main Process
ipcMain.handle('some-name', async (event, someArgument) => {
  const result = await doSomeWork(someArgument);
  return result;
});
```

## nativeImage

使用 PNG 或 JPG 文件创建 Tray、Dock 和应用程序图标。

## webFrame

`webFrame` 用于自定义当前网页的呈现方式

```js
const { webFrame } = require('electron');

webFrame.setZoomFactor(2);
```
