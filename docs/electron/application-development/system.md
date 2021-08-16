---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 系统管理
order: 6
---

# 系统管理

## 系统对话框

```js
const { dialog } = require('electron');

console.log(dialog.showOpenDialog({ properties: ['openFile', 'multiSelections'] }));
```

## 菜单

隐藏窗口的系统菜单

```js
const win = new BrowserWindow({
  webPrefereces: { nodeIIntegration: true },
  autoHideMenuBar: true,
});
```

自定义系统菜单覆盖默认菜单

```js
const { app, Menu } = require('electron');

const isMac = process.platform === 'darwin';

const template = [
  ...(isMac
    ? [
        {
          label: app.name,
          submenu: [{ role: 'about' }, { type: 'separator' }, { role: 'services' }],
        },
      ]
    : []),
  {
    label: 'File',
    submenu: [isMac ? { role: 'close' } : { role: 'quit' }],
  },
  {
    label: 'Edit',
    submenu: [{ role: 'undo' }, { role: 'redo' }, { type: 'separator' }],
  },
  {
    label: 'View',
    submenu: [
      { role: 'reload' },
      { role: 'forceReload' },
      { role: 'toggleDevTools' },
      { type: 'separator' },
      { role: 'resetZoom' },
      { role: 'zoomIn' },
      { role: 'zoomOut' },
      { type: 'separator' },
      { role: 'togglefullscreen' },
    ],
  },
  {
    label: 'Window',
    submenu: [{ role: 'minimize' }, { role: 'zoom' }],
  },
  {
    role: 'help',
    submenu: [
      {
        label: 'Learn More',
        click: async () => {
          const { shell } = require('electron');
          await shell.openExternal('https://electronjs.org');
        },
      },
    ],
  },
];

const menu = Menu.buildFromTemplate(template);
Menu.setApplicationMenu(menu);
```

## 快捷键

如果希望在窗口处于非激活状态时也能监听到用户的按键，应该使用 Electron 的 [globalShortcut](https://www.electronjs.org/docs/api/global-shortcut) 模块：

```js
const { globalShortcut } = require('electron');

globalShortcut.register('CommandOrControl+K', () => {
  console.log('案件监听');
});
```

[Electron 快捷键文档](https://www.electronjs.org/docs/api/accelerator)

## 托盘图标

Electron 应用在系统托盘处注册一个图标十分简单：

```js
const { app, BrowserWindow, Tray } = require('electron');
const path = require('path');

let tray;
app.on('ready', () => {
  let iconPath = path.join(__dirname, 'icon.png');
  tray = new Tray(iconPath);
});
```

为托盘图标注册菜单的代码：

```js
const { Tray, Menu } = require('electron');
const menu = Menu.buildFromTemplate([
  {
    click() {
      window.show();
    },
    label: '显示窗口',
    type: 'normal',
  },
  {
    click() {
      app.quit();
    },
    label: '退出应用 ',
    type: 'normal',
  },
]);

tray.setCOntextMenu(menu);
```

## 剪切板

## 系统通知

### 主进程内发送系统通知

```js
const { Notification } = require('electron').remote;

const notification = new Notification({
  title: '您接受到新信息',
  body: '此为消息的正文，点击查看消息',
});

notification.show();
notification.on('click', () => {
  aler(5);
});
```

## 其他

### 使用系统默认应用打开文件

使用 Electron 的 `shell` 模块处理该类需求。

```js
const { shell } = require('electron');

const result = await shell.openExternal('https://www.baidu.com');
```

使用默认应用打开 Word 文档

```js
const { shell } = require('electron');

const isOpen = shell.openItem('D:\\工作\\Electron.docx');
```

### 使用系统字体

```html
<div style="font: caption">This is title</div>
<div style="font: menu">菜单中的字体</div>
<div style="font:message-box">对话框中的字体</div>
<div style="font:status-bar">状态栏中的字体</div>
```

### 最近打开的文件

```js
// 增加一个最近打开的文件
app.addRecentDocument('C:UsersAdministratorDesktop\1.jpg');

// 清空最近打开的文件列表
add.clearRecentDocuments();
```

在 Windows 系统中，需要做一些额外的操作才能让 `addRecentDocument` 有效。需要把应用注册为某类文件的处理程序，否则应用不会显示最近打开的文件列表。把应用注册为某累文件的处理程序的方法请参阅微软的官方文档。当用户点击最近打开的文件列表中的某一项时，系统会启动一个新的应用程序的实例，而文件的路径将作为一个命令行参数被传入这个实例。

在 Mac 操作系统中，不需要为应用注册文件类型，当用户点击最近打开的文件列表中的某一项时，会触发 `app` 的 `open-file` 事件。文件的路径会通过参数的形式传递给该事件的回调函数。

## 参考资料

- [electron/nwjs 如何加入开机启动项？auto-launch 如何使用](https://newsn.net/say/node-auto-launch.html)
- [开发实战之记账软件——开机自动启动](https://my.oschina.net/u/3667677/blog/3042628)
