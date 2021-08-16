---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 自动更新
order: 14
---

# 自动更新

Electron 应用的自动更新推荐使用 [electron-builder](https://github.com/electron-userland/electron-builder) 中的 Auto Update 功能。

```js
const log = require('electron-log');
const { autoUpdater } = require('electron-updater');
const { updateWindow } = require('./window');
const { app，ipcMain } = require('electron');

//设置日志打印
autoUpdater.logger = log;
autoUpdater.logger.transports.file.level = 'info';

//是否自动下载更新，设置为false时将通过api触发更新下载
autoUpdater.autoDownload = false;
//是否允许版本降级，也就是服务器版本低于本地版本时，依旧以服务器版本为主
autoUpdater.allowDowngrade = true;

//设置服务器版本最新版本查询接口配置
autoUpdater.setFeedURL({
    provider: 'generic',
    url: 'https://www.shaokaotan.com/autoUpdate/',
    channel: 'myProgram',
});

//保存是否需要安装更新的版本状态，因为需要提供用户在下载完成更新之后立即更新和稍后更新的操作
let NEED_INSTALL = false;

//调用api检查是否用更新
const checkUpdate = () => {
    autoUpdater.checkForUpdatesAndNotify().then((UpdateCheckResult) => {
        log.info(UpdateCheckResult, autoUpdater.currentVersion.version);
        //判断版本不一致，强制更新
        if (UpdateCheckResult && UpdateCheckResult.updateInfo.version !== autoUpdater.currentVersion.version) {
            //调起更新窗口
            updateWindow();
        }
    });
};

//api触发更新下载
const startDownload = (callback, downloadSuccessCallback) => {
    //监听下载进度并推送到更新窗口
    autoUpdater.on('download-progress', (data) => {
        callback && callback instanceof Function && callback(null, data);
    });
    //监听下载错误并推送到更新窗口
    autoUpdater.on('error', (err) => {
        callback && callback instanceof Function && callback(err);
    });
    //监听下载完成并推送到更新窗口
    autoUpdater.on('update-downloaded', () => {
        NEED_INSTALL = true;
        downloadSuccessCallback && downloadSuccessCallback instanceof Function && downloadSuccessCallback();
    });
    //下载更新
    autoUpdater.downloadUpdate();
};

//监听窗口发送来的进程消息，开始下载更新
ipcMain.on('startDownload', (event) => {
    startDownload(
            (err, progressInfo) => {
                if (err) {
                    //回推下载错误消息
                    event.sender.send('update-error');
                } else {
                    //回推下载进度消息
                    event.sender.send('update-progress', progressInfo.percent);
                }
            },
            () => {
                //回推下载完成消息
                event.sender.send('update-downed');
            }
        );
})；

//监听用户点击更新窗口立即更新按钮进程消息
ipcMain('quitAndInstall', () => {
    NEED_INSTALL = false;
    autoUpdater.quitAndInstall(true, true);
})

//用户点击稍后安装后程序退出时执行立即安装更新
app.on('will-quit', () => {
    if (NEED_INSTALL) {
        autoUpdater.quitAndInstall(true, true);
    }
});
```

要使上面流程能正确地执行下去，还需要提供一个正确的服务返回一个正确的更新配置：

代码中设置服务器地址：

```js
autoUpdater.setFeedURL({
  provider: 'generic',
  url: 'https://www.electron-server.com/autoUpdate/',
  channel: 'electron-program',
});
```

`electron-updater` 将会请求该链接的服务地址，这个地址需要返回一个更新配置信息文件，这个文件就是 `electron-builder` 打包出来的 YML 文件。

需要注意以下几点：

- `electron-updater` 会校验配置中 `version` 的格式，需要符合为 semver version 规范，但是一般 Windows 应用是四位版本号，所以采用像 `*.*.*-*` 这种形式能通过检验
- `electron-updater` 会校验配置中的 `sha512` 值与更新下载下来的文件的 `sha512` 值，不匹配会抛出更新错误，所以不能随意修改安装程序（只要文件 MD5 值不变即可）
- 配置中的 `path` 值即是 `electron-updater` 执行下载更新的目标地址，是新版本程序安装包的文件地址
