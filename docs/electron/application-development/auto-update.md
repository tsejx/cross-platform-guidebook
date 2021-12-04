---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 更新管理
order: 14
---

# 更新管理

## 渲染进程界面更新方案

Electron 应用是由主进程和渲染进程组成。主进程主要为 Web 应用提供 Native 能力，而渲染进程负责 UI 交互。在一些业务场景下主进程代码没有改动，只是渲染进程代码有更新，因此客户端只需更新渲染进程即可，也就是主进程创建的 `window` 窗口只需要重新加载新的 HTML 页面即可，具体流程如下。

1. 客户端主进程通过轮询、定时请求或者服务端推送方式接收更新通知
2. 在主进程中对需要更新的渲染进程执行 `webContents.reloadIngnoringCache()` 完成页面重载

## 全量更新

```jsx | inline
import React from 'react';
import img from '../../assets/electron/render-static-resource.jpg';

export default () => <img alt="更新步骤" src={img} width="72%" height="72%" />;
```

当应用代码改动较大时，比如 Electron 的版本升级、项目架构调整等，我们就可能需要用户下载全量的升级包来升级。Electron 官方提供了多种[应用更新方案](https://www.electronjs.org/docs/latest/tutorial/updates#implementing-updates-in-your-app)。主要包括使用 Electron 团队维护的 `update.electron.org` 实现自动更新，以及 `electron-builder` 打包方案。

Electron 应用的自动更新推荐使用 [electron-builder](https://github.com/electron-userland/electron-builder) 中的 AutoUpdate 功能。

在打包配置文件中，需要配置 `publish` 选项，指定 `provider` 和 `url`，其中 `url` 是存放升级更新包的服务器。

```json
{
  "appId": "com.test.app",
  "productName": "APP",
  "files": ["./build/**/*"],
  "compression": "maximum",
  "directories": {
    "output": "dist"
  },
  "publish": [
    {
      "provider": "generic",
      "url": "http://static.test.com" // 资源服务器
    }
  ],
  "mac": {
    "target": ["dmg", "zip"]
  }
}
```

打包完成后，我们将打包目录下生成的所有文件都放到资源服务器，给应用提供下载。

```bash
├── mac
├── app-0.0.2-mac.zip
├── app-0.0.2.dmg
├── app-0.0.2.dmg.blockmap
├── builder-effective-config.yaml
├── latest-mac.yml
```

更新步骤：

1. 客户端通过定时检测、或者服务端推送方式检测是否有更新
2. 执行 `autoUpdater.checkForUpdates()` 的检测逻辑，读取资源服务器 `latest-mac.yml` 文件，对比文件摘要
3. 有更新则执行文件下载操作，可以配合显示下载进度
4. 下载完成之后，通知渲染进程页面显示本次更新的相关内容
5. 应用重启进行更新

```js
const log = require('electron-log');
const { autoUpdater } = require('electron-updater');
const { updatndow } = require('./window');
const { app，ipcMain } = require('electron');

// 设置日志打印
autoUpdater.logger = log;
autoUpdater.logger.transports.file.level = 'info';

// 是否自动下载更新，设置为 false 时将通过 API 触发更新下载
autoUpdater.autoDownload = false;
// 是否允许版本降级，也就是服务器版本低于本地版本时，依旧以服务器版本为主
autoUpdater.allowDowngrade = true;

// 设置服务器版本最新版本查询接口配置
autoUpdater.setFeedURL({
    provider: 'generic',
    url: 'https://www.shaokaotan.com/autoUpdate/',
    channel: 'myProgram',
});

// 保存是否需要安装更新的版本状态，因为需要提供用户在下载完成更新之后立即更新和稍后更新的操作
let NEED_INSTALL = false;

// 调用 API 检查是否用更新
const checkUpdate = () => {
  autoUpdater.checkForUpdatesAndNotify().then((UpdateCheckResult) => {
    log.info(UpdateCheckResult, autoUpdater.currentVersion.version);
    // 判断版本不一致，强制更新
    if (UpdateCheckResult && UpdateCheckResult.updateInfo.version !== autoUpdater.currentVersion.version) {
      // 调起更新窗口
      updateWindow();
    }
  });
};

// API 触发更新下载
const startDownload = (callback, successCallback) => {
  // 检测开始
  autoUpdater.on('checking-for-update', function() {
    console.log('checking-for-update')
  })
  // 更新可用
  autoUpdater.on('update-available', function() {
    console.log('update-available')
  })
  // 更新不可用
  autoUpdater.on('update-not-available', function() {
    console.log('update-not-available')
  })
  // 监听下载进度并推送到更新窗口
  autoUpdater.on('download-progress', (data) => {
    callback && callback instanceof Function && callback(null, data);
  });
  // 监听下载错误并推送到更新窗口
  autoUpdater.on('error', (err) => {
    callback && callback instanceof Function && callback(err);
  });
  // 监听下载完成并推送到更新窗口
  autoUpdater.on('update-downloaded', () => {
    NEED_INSTALL = true;
    successCallback && successCallback instanceof Function && successCallback();
  });
  // 下载更新
  autoUpdater.downloadUpdate();
};

// 监听应用层发送来的进程消息，开始下载更新
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

// 监听用户点击更新窗口立即更新按钮进程消息
ipcMain('quitAndInstall', () => {
  NEED_INSTALL = false;
  autoUpdater.quitAndInstall(true, true);
})

// 用户点击稍后安装后程序退出时执行立即安装更新
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

全量更新方案的优缺点：

- 优点
  - 打包配置简单，只需添加 `publish` 即可
  - 代码逻辑简单，添加 `autoUpdater` 即可
- 缺点
  - 安装包体积过大时，浪费带宽，增加用户升级时间
  - 代码改动量小时，全量升级完全没有必要
  - 当我们的应用依赖一些第三方 SDK 时，当第三方 SDK 有打包问题时，可能会出现升级失败的情况

## 增量更新

应用在代码改动较少的情况下，用户体验好、比较优雅的更新方式就是增量更新了。增量更新的方案也有多种，具体的增量更新方案需要针对具体的业务需求进行定制。下边介绍两种常见的增量更新方案，我们仍然是基于 `electron-builder` 的打包方式来实现的。

### 固定模块升级

`asar` 是 Electron 提供的一种将多个文件合并成一个文件的类 `tar` 风格的归档格式，不仅可以缓解 Widows 下路径名过长的问题，还能略微加快 `require` 的速度，并且可以隐藏你的源代码（并非绝对隐藏，专业人士还是可以解压缩的），了解更多请查看官方文档 [asar](https://github.com/electron/asar)。

`asar` 方式下应用的启动流程：

要想完成 `asar` 方式下应用的更新，我们必须先了解 Electron 应用在这种模式下是如何启动。 其实在这种模式下 Electron 应用在启动时会读取 `app.asar.unpacked` 目录中的内容合并到 `app` 目录中进行加载,因此增量更新时我们只需要替换应用安装目录中的 `app.asar.unpacked` 目录，然后重启应用即可。

```jsx | inline
import React from 'react';
import img from '../../assets/electron/asar-workflow.jpg';

export default () => <img alt="ASAR工作流" src={img} width="72%" height="72%" />;
```

基于 `electron-builder` 的 `asar` 打包配置。

了解了 `asar` 方式下应用的启动流程之后，我们就可以对 `electron-builder` 的打包配置文件进行修改，设置成 `asar` 模式，具体配置文件如下。

```json
{
  "appId": "com.test.app",
  "productName": "APP",
  "files": ["./build/**/*"],
  "compression": "maximum",
  "asar": true,
  "asarUnpack": [
    "./build/src", // 不需要打包到 asar 中的文件，也就是有改动的代码
    "./build/sdk/one",
    "./build/sdk/two"
  ],
  "directories": {
    "output": "dist"
  },
  "mac": {
    "target": ["dmg", "zip"]
  }
}
```

打包生成的文件

根据配置文件打包后会生成安装包和增量包，其中 `app.asar` 压缩文件就是基本不要变动的代码，`app.asar.upacked` 目录就是我们的增量文件，也就是需要变更的代码。最后注意我们要对增量包进行压缩，减少更新包体积，然后上传文件到服务器。

```bash
├── mac
    ├── Contents
        ├── _CodeSignature
        ├── Frameworks
        ├── MacOS
        ├── Resources
            ├── app.asar  # 这就是我们的 asar 包,也就是不需要改动的代码
            ├── app.asar.unpacked # 这就是我们的增量包，修改更新的代码
├── app-0.0.2-mac.zip
├── app-0.0.2.dmg
├── app-0.0.2.dmg.blockmap
├── builder-effective-config.yaml
└── latest-mac.yml
```

更新过程：

1. 客户端通过定时检测、或者服务端推送方式检测是否有更新
2. 通过版本对比发现更新，并获取到需要更新的文件名称
3. 下载 `app.asar.upacked.xxx.tgz` 更新文件到应用的指定目录（路径因系统而异）
4. 解压覆盖原文件，重启应用

```jsx | inline
import React from 'react';
import img from '../../assets/electron/asar-update-flow.jpg';

export default () => <img alt="ASAR更新流程" src={img} width="72%" height="72%" />;
```

应用安装路径：

- MacOS：`/Applications/App.app/Contents/Resources`
- Windows：`D:/xxx/App/resources`

更新过程参考代码：

```js
// 下载更新文件
fetch(downloadURL)
  .then((res) => {
    let stream = fs.createWriteStream(unpackPath);
    res.body.pipe(stream);
    stream.on('close', () => {
      // 执行解压更新操作
      uncompressAndUpdate();
    });
  })
  .catch((err) => {
    logger.error(`download ${downloadURL}: ${err.toString()}`);
  });

function uncompressAndUpdate() {
  // 先备份当前的 app.asar.unpacked 目录
  fs.renameSync(untgzPath, `${untgzPath}.back`);

  compressing.tgz
    .uncompress(unpackPath, appPath)
    .then((res) => {
      logger.info(`uncompress ${asarName} success`);
      deleteDirSync(`${untgzPath}.back`);

      // 解压之后，重启应用即可
      app.relaunch();
      app.exit(0);
    })
    .catch((err) => {
      // 记录错误日志
      logger.error(`uncompress ${asarName} error: ${err.toString()}`);
      fs.renameSync(`${untgzPath}.back`, untgzPath);
    });
}
```

优缺点：

- 优点
  - 可以对代码进行压缩，在一定程度上隐藏源码、提高加载速度
  - `asar` 和 `asarunpacked` 分开，很方便的实现增量更新
- 缺点
  - 只能在一定程度上隐藏源码，使用 asar 可以方便地解压缩（可靠的方法还是对源码进行混淆压缩）
  - `asar` 压缩文件中存在 Node API 的局限性，无法实现非压缩下的所用功能，对于有 Node 执行有强需求的可能要仔细斟酌该方案，笔者就遇到了无法执行 `child_process.exec`，`child_process.spawn` 方法的问题

### 自定义模块升级

使用非 `asar` 方式，可以让我们的应用拥有更多的灵活性。下边我们介绍的方式，就是充分利用了 `electron-builder` 中两个常用的配置选项 `extraResources`（拷贝资源到打包目录 `resources` 中）、`extraFiles`（拷贝资源到打包目录的根路径），帮助我们轻松实现增量更新。

基于 electron-builder 的非 asar 打包配置文件：

```json
{
  "appId": "com.test.app",
  "productName": "App",
  "files": ["./build/**/*"],
  "asar": false,
  "asarUnpack": [],
  "compression": "maximum",
  "directories": {
    "output": "dist"
  },
  "mac": {
    "target": ["dmg", "zip"],
    "icon": "./build/src/icons/app.icns",
    "extendInfo": {
      "CFBundleURLSchemes": ["schema"]
    },
    // 拷贝需要的资源
    "extraResources": [
      {
        "from": "./SDK/",
        "to": "SDK"
      }
    ]
  }
}
```

打包完成生成如下的文件。

`app` 目录是我们的应用目录。SDK 目录就是我们额外拷贝过去的目录。SDK 都属于改动较小的部分，而我们的 `app` 是改动比较频繁的目录。因此增量更新其实就是替换 `app` 目录即可，而完全不需要重新下载 SDK，当然如果 SDK 也需要更新的话，更新逻辑中可以添加 SDK 的更新即可。在打包完成之后，使用压缩脚本自动将 `app` 目录压缩生成 `app_mac_0.0.2.tgz`，它就是我们的更新包，上传到更新服务器。

```bash
├── mac
    ├── Contents
        ├── _CodeSignature
        ├── Frameworks
        ├── MacOS
        ├── Resources
            ├── app  >> 应用代码
            ├── SDK  >> 应用代码中依赖的SDK,被拷贝过来的
├── app-0.0.2-mac.zip
├── app-0.0.2.dmg
├── app-0.0.2.dmg.blockmap
├── builder-effective-config.yaml
└── latest-mac.yml
```

非 `asar` 增量更新代码流程：

1. 客户端啊定时、轮询，或者服务端主动推送方式通知检测更新
2. 从服务器下载需要更新的文件
3. 解压并覆盖已有的文件
4. 退出重启应用

非 `asar` 方式的优缺点：

- 优点
  - 不用考虑 `asar` 的各种限制
  - 更新方式极其灵活，可根据业务方便的进行定制
- 缺点
  - 无法对代码进行归档压缩（其实对源码对混淆压缩才是真的隐藏）

## 参考资料

- [Electron - 自动升级总结](https://www.jianshu.com/p/b505adb91e11)
- [Electron 客户端场景化更新升级方案实践](https://zhuanlan.zhihu.com/p/94171579)
- [Electron 全量更新](https://segmentfault.com/a/1190000039683917)
- [Electron 增量更新（一）](https://segmentfault.com/a/1190000039747461)
- [Electron 增量更新（二）](https://segmentfault.com/a/1190000039872331)
- [Electron 增量更新和全量更新](https://www.cnblogs.com/mapleChain/p/13063954.html)
