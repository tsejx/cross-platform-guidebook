---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 5
title: 生命周期
order: 13
---

# 生命周期

```jsx | inline
import React from 'react';
import img from '../../assets/electron/flow.png';

export default () => <img alt="Electron" src={img} width="64%" height="64%" />;
```

## 退出场景

### 正常退出

- `Command + Q` 或者菜单中的退出按钮，Windows 中 `Alt + F4`
- `app.quit()`
- `autoupdater.quitAndInstall()`
- `app.reluanch()`

### 异常退出

比如常见的在主进程调用 `process.crash()`

### SIG 信号退出

我们常见的命令行退出软件的方式有 `Ctrl + C`，命令行给进程发送了 `SIGKILL` 信号，其实还有其它常见关闭进程的方式，可以通过 `kill` 对进程发送信号，比如 `kill -s KILL 24567` 或者 `kill -9 24567`。

## 启动场景

软件常见的启动有很多，比如通过 `open` 或者 Windows 的 `start` 启动，或者通过 URL Scheme 然后系统启动，或者拖动文档到 Dock 或者 Tray 启动。下面来说说启动软件后，软件参数和环境的路径。

### 普通启动

一般双击软件启动，会经过 `will-finish-launching` 和 `ready`，然后正常进入应用界面。

### 单例模式下的应用启动

在应用启动的时候，检测 `app.requestSingleInstanceLock()` 是否是单例模式，如果是就退出，并且发送一个事件给 `second-instance`，从这里再获取退出的应用的 `argv` 和 `cwd`。

其实这里检测是否是单例模式的方法是，看看 `app.getPath('userData')` 下是否有 `lock` 文件。

### 通过命令行启动

命令行执行 `/path/to/app --arg1 value1 --arg2 value2 document/path`，会在应用启动的时候，通过 `process.argv` 和 `process.cwd`。

### 通过 URL Scheme 启动

通过 `app.setAsDefaultProtocolClient('electron-test')` 注册 URL Scheme，然后在 APP 事件 `open-url` 的 URL 参数获取 URL 信息：

```
app-event - did-become-active
app-event - open-url - electron-test://happy?abc=eee#ii=aa
```

### 通过拖拽到 dock 或者 tray 启动

通过 `dock` 拖拽启动，需要在 `info.plist` 中声明支持的文件类型，比如 `electronBuilder` 可以通过 `extendInfo` 字段中，声明 `CFBundleDocumentTypes` 注册支持的类型。触发的是事件 `open-file`。

## 参考资料

- [Electron 生命周期介绍](https://zhuanlan.zhihu.com/p/352668011)
