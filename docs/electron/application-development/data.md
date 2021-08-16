---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 数据管理
order: 5
---

# 数据管理

## 本地文件持久化数据

### 用户数据目录

操作系统为应用程序提供专有目录来存储应用程序的用户个性化数据：

- Windows 操作系统：`C:\Users\[your user name]\AppData\Roaming`
- macOS 操作系统：`/Users/[your user name]/Library/Application Support/`
- Linux 操作系统：`/home/[your user name]/.config/xiangxuema`

当然，Electron 为我们提供了一个便捷的 API 来获取此路径：

```js
const userData = app.getPath('userData');
```

给 `app.getPath` 方法传入不同的参数，可以获取不同用途的路径。

- `tmp`：对应系统临时文件夹路径
- `exe`：对应当前执行程序的路径
- `appData`：对应应用程序用户个性化数据的目录
- `userData`：是 `appData` 路径后再加上应用名的路径，时 `appData` 的子路径。

除此之外，也可以使用 Node.js 的能力获取系统默认路径。

```js
// 获取当前用户的主目录，如 "C:\Users\allen"
const homedir = require('os').homedir();

// 获取默认临时文件夹目录
const tmpdir = require('os').tmpdir();
```

## 浏览器技术持久化数据

### 读写受限访问的 Cookie

在浏览器中读写 Cookie 存在无法读写 HttpOnly 标记的 Cookie 和其他域下的 Cookie 的限制，但是 Electron 提供了专门用于读取 [Cookie](https://www.electronjs.org/docs/api/cookies) 的 API：

```js
const { BrowserWindow } = require('electron');

const win = new BrowserWindow({ width: 800, height: 600 });
win.loadURL('http://github.com');

const ses = win.webContents.session;

const getCookie = async (name) => {
  const cookies = await ses.defaultSession.cookies.get({ name });

  if (cookies.length > 0) {
    return cookies[0].value;
  } else {
    return '';
  }
};

const setCookie = async (cookie) => {
  await ses.defaultSession.cookies.set(cookie);
};
```

[session](https://www.electronjs.org/docs/api/session) 模块是 Electron 用于管理浏览器会话、Cookie、缓存、代理设置等的模块。

### 清空浏览器缓存

通过 Electron 为我们提供的 `clearStorageData` 方法，我们可以快速清除保存在 Cookie 内或 IndexedDB 中的数据。

```js
await remote.session.defaultSession.clearStorageData({
  storages: 'cookie,localstorage',
});
```

该方法接收 `option` 对象，对象的 `storages` 属性可设置以下值任意一个或多个（多个值由英文逗号分隔）：`appcache`、`cookies`、`filesystem`、`indexdb`、`localstorage`、`shadercache`、`websql`、`servicworkers`、`cachestorage`。

另外还可以为 `option` 对象设置配额和 `oriign` 属性来更精细地控制清理的条件。

## 社区精选模块包

读写本地文件

| 模块包           | 说明                                                                   |
| :--------------- | :--------------------------------------------------------------------- |
| `fs-extra`       | 继承 Node.js `fs` 模块的扩展库，增加非常多实用的 API                   |
| `lowdb`          | 基于 Lodash 提供更上层的封装，能轻松操作 JSON 数据，嗨内置文件读写支持 |
| `electron-store` | 支持数据加密以防止被恶意窃取，且不需指定文件路径和文件名的轻量级数据库 |

数据库

| 模块包         | 说明                                       |
| :------------- | :----------------------------------------- |
| `dexie.js`     | IndexedDB 版本 API 的封装，更易用          |
| `pounchdb`     | 使用 Idb 作为 IndexedDB 的适配器           |
| `node-sqlite3` | 对 SQLite3 的简单封装                      |
| `knexjs`       | SQL 指令构建器，可编写串行化的数据访问代码 |
