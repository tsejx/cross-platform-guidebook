---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 硬件管理
order: 8
---

# 硬件管理

以前 Web 前端技术是没办法访问客户端的硬件设备的，后来 HTML5 提供了一系列的技术来支持这项不足，但是限制太多了，一旦尝试使用这些特性，浏览器就会弹窗，用户同意后才能访问硬件。Electron 可以自由的使用这些特性，并且默认拥有了这些硬件的访问权限，甚至还提供了额外的支持以帮助开发者使用硬件的能力。

## 屏幕

用户电脑可能外界了多个显示器，那么你的窗口想显示在哪个显示器？首先你得获取到这些显示器：

```js
const remote = require('electron').remote;

// 获取主显示器
let mainScreen = remote.screen.getPrimaryDisplay();

console.log(mainScreen);
```

`mainScreen` 是一个对象，包含字段：

- `id`：显示器 ID
- `rotation`：显示器是否选择，值可能是 0、90、180、270
- `touchSupport`：是否支持触屏
- `bounds`：绑定区域，可根据此判断是否为外界显示器
- `size`：显示器大小，与显示器分辨率有关，但不一定是显示器的分辨率

将窗口显示到外界显示器上去

```js
const { screen } = require('electron');
const displays = screen.getAllDisplays();
const externalDisplay = displays.find(
  (display) => display.bounds.x !== 0 || display.bounds.y !== 0
);

// 判断是否外接扩展示器，如果 bounds.x 和 bounds.y 不等于 0 则为外接显示器
// 然后将窗口显示在外接显示器的左上角
if (externalDisplay) {
  win = new BrowserWindow({
    x: externalDisplay.bounds.x + 50,
    y: externalDisplay.bounds.y + 50,
    webPreferences: {
      nodeIntegration: true,
    },
  });

  win.loadURL('https://www.baidu.com');
}
```

另外，显示器信息对象包含 `internal` 属性，官方说明此属性值为 `true` 是内置显示器，`false` 为外接显示器。但实际开发时发现这个目前有 bug，无论是内置还是外接都显示为 `false`，因此通过 `display.bounds` 来确定是否外界显示器更准确。

注意：`screen` 模块只有在 `app.ready` 事件之后才能使用。

## 音视频设备

在开发 Web 网页应用时，如果要使用用户的音视频设备，浏览器为了安全，会向用户发出提示。用户允许浏览器访问其音视频设备后，前端代码才有权访问这些设备。而 Electron 不必获得用户授权，直接具有访问用户音视频设备的能力。

打开摄像头和麦克风获取用户音视频流

```js
const options = {
  audio: true,
  video: true,
};

// 获取摄像头和麦克风产生的流数据
const mediaStream = await navigator.mediaDevices.getUserMedia(option);
const video = document.querySelector('video');

video.srcObject = mediaStream;
video.onloadedmetadata = function (e) {
  video.play();
};
```

在示例代码中，我们使用 `navigator.mediaDevices.getUserMedia` 来获取用户的音视频流，该方法需要一个配置参数，包含两个属性 `audio` 和 `video`。详细的参数请查看 [MediaStreamConstraints](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamConstraints) 和 [MediaTrackConstraints](https://developer.mozilla.org/en-US/docs/Web/API/MediaTrackConstraints)。

如果设备有多个摄像头并且不区分前后，可以通过以下方法获取所有与摄像头基本信息：

```js
const devices = await navigator.mediaDevices.enumerateDevices();
```

## 录屏

通过 [desktopCapturer](https://www.electronjs.org/docs/api/desktop-capturer) 模块提供的 API 可用获取桌面的屏幕视频流，并且可以指定只获取某个应用的录屏，例如获取微信窗口的视频流显示在窗体内：

```js
const { desktopCapture } = require('electron');

// 获取桌面所有应用
const sources = await desktopCapture.getSources({
  types: ['window', 'screen'],
});

// 找到微信
let target = sources.find((v) => v.name === '微信');
let mediaStream = await navigator.mediaDevices.getUserMedia({
  audio: false,
  video: {
    mandatory: {
      chromeMediaSource: 'desktop',
      chromeMediaSourceId: target.id,
    },
  },
});

const video = document.querySelector('video');
video.srcObject = mediaStream;
video.onloadedmetadata = function (e) {
  vido.play();
};
```

其中 `desktopCapture.getSources` 获取所有显示在桌面上的应用信息，获取到指定应用后，我们把应用的 ID 传递给 `video.mandatory.chromeMediaSourceId`，同时设置了 `video.mandatory.chromeMediaSource` 的值为 `desktop`。此后我们得到的视频流对象就与从摄像头里得到的视频流对象基本一致了。

## 电源

### 电源的基本状态和事件

获取电源状态

```js
const battery = await navigator.getBattery();
```

| 属性名称                  | 说明                                                                     |
| :------------------------ | :----------------------------------------------------------------------- |
| `battery.charging`        | 是否正在充电，只要在充电，哪怕是满电状态也返回 `true`                    |
| `battery.chargingTime`    | 距离电池充满还剩多少时间，单位秒，如果为 0 则代表充满了                  |
| `battery.dischargingTime` | 电池电量还能用多久，单位秒，如果电池满电并且还在冲，值位 infinity 无穷大 |
| `battery.level`           | 充电水平，值范围在 0 到 1 之间                                           |

Battery 实例的事件：

| 事件名称                          | 说明                               |
| :-------------------------------- | :--------------------------------- |
| `battery.onchargingchange`        | 接入和断开交流电时触发该事件       |
| `battery.onchargingtimechange`    | 当 chargingTime 属性发生变化时触发 |
| `battery.ondischargingtimechange` | 当 dischargingTime 属性变化时触发  |
| `battery.onlevelchange`           | 当 level 属性变化时触发            |

### 监控系统挂起与锁屏事件

上面的电源 API 是 HTML5 中为网页提供的，在 Electron 中不需要用户授权就能访问。除了 HTML5 API 的能力外，Electron 自己也封装了对电源的 [powerMonitor](https://www.electronjs.org/docs/api/power-monitor) 模块，并且把监控系统是否挂起和恢复事件、系统空闲状态获取的能力也放在了这个模块中。

#### 监视系统挂起和恢复事件

```js
const { powerMonitor } = require('electron').remote;

powerMonitor.on('suspend', () => {
  console.log('系统睡眠了');
});

powerMonitor.on('resume', () => {
  console.log('系统唤醒了');
});
```

#### 监听系统锁屏和解锁事件

目前只适用于 win 和 mac，linux 下不行：

```js
powerMonitor.on('lock-screen', () => {
  console.log('锁屏了');
});

powerMonitor.on('unlock-screen', () => {
  console.log('解锁了');
});
```

系统挂起（睡眠）后，系统内一些应用会切换到挂起状态，不再提供服务，如果 Electron 应用依赖这些服务就会出现问题，因此可以通过这四个事件做一些挂起前后的处理防止业务异常。

### 阻止系统锁屏

操作系统在一段事件没有收到用户操作时会进入省电模式，关闭显示器，把内存中的内容转存到磁盘，进入睡眠模式。如果做的是个播放器这种应用，在播放视频的时候就需要防止系统息屏、睡眠。操作系统提供了[powerSaveBlocker](https://www.electronjs.org/docs/api/power-save-blocker) API，Electron 对其有实现：

```js
const { powerSaveBlocker } = require('electron');

const id = powerSaveBlocker.start('prevent-display-sleep');

// 判断阻止行为是否已启动
console.log(powerSaveBlocker.isStarted(id));

powerSaveBlocker.stop(id);
```

| 事件名称                 | 说明                                                       |
| :----------------------- | :--------------------------------------------------------- |
| `prevent-display-sleep`  | 防止系统锁屏                                               |
| `prevent-app-suspension` | 防止程序挂起（程序在下载文件或播放音乐时需要阻止音乐挂起） |

`start()` 方法返回一个整型 `id`，如果要解除阻止可以通过 `stop()` 传入 `id` 来解除：

## 打印机

打印当前窗体中的页面

```js
const { remote } = require('electron');

const webContents = remote.getCureentWebContents();

webContents.print(
  {
    slient: false,
    printBackground: true,
    deviceName: '',
  },
  // 打印完成后的回调函数
  // 打印完成则 success 为 true，如果打印失败或者用户取消则返回 false
  (success, error) => {
    if (!success) {
      console.log(error);
    }
  }
);
```

执行如上代码会弹出打印机选择框，选择打印机才能打印，如果要设置指定打印机，那么应该先获得当前电脑所有打印机，然后用户选择一个默认打印机。

```js
const { remote } = require('electron');

let content = remote.getCurrentWebContents();
let printers = content.getPrinters();

printers.forEach((element) => {
  // 输出打印机名称
  console.log(element.name);
  // 那这个打印机名称传递到上面 print() 方法的 deviceName 即可
});
```

### 导出 PDF

可以通过打印能力把页面导出为 PDF 文件

```js
const { remote } = require('electron');
const path = requrie('path');
const fs = require('fs');

const webContents = remote.getCurrentWebContents();

const data = await webContents.printToPDF({});

let filePath = path.join(__static, 'xxx.pdf');

fs.writeFile(filePath, data, (error) => {
  if (error) throw error;
  console.log('PDF导出成功');
});
```

`webContents.printToPDF()` 方法可接受的参数配置项与前面 `print()` 方法是一样的，这里不再重复演示。`printToPDF()` 方法返回一个 Promise 对象，内容是 Buffer 缓存，可以把这个 Buffer 保存到指定文件路径，也可以打开保存文件对话框让用户选择保存文件的路径，现在我们把上面 `fs.writeFile()` 方法改造一下：

```js
const savePath = await remote.dialog.showSaveDialog({
  title: '保存PDF',
  filters: [
    {
      name: 'xxx',
      extensions: ['pdf'],
    },
  ],
});

if (savePath.canceled) {
  return; // 用户在对话框点击取消了，则不保存
}

// 进行保存
fs.writeFile(savePath.filePath, data, (error) => {
  if (error) throw error;
});
```

## 硬件信息

[Electron Process API](https://www.electronjs.org/docs/api/process)

获取内存使用情况

```js
const memoryUsage = process.getSystemMemoryInfo();

console.log(memoryUsage);
```

| 属性名称    | 说明                               |
| :---------- | :--------------------------------- |
| `total`     | 当前系统可用的物理内存总量         |
| `free`      | 应用程序或磁盘缓存未使用的内存总量 |
| `swapTotal` | 系统交换内存总量                   |
| `swapFree`  | 系统可用交换内存总量               |

获取 CPU 使用情况

```js
setInterval(() => {
  const cpuUsage = process.getCPUUsage();

  console.log(cpuUsage);
}, 1600);
```

打印出来 [CPUUsage](https://www.electronjs.org/docs/api/structures/cpu-usage) 对象，有两个属性：

- `percentCPUUsage`：某个时间段内 CPU 使用率
- `idleWakeupsPerSecond`：某个时间段内每秒唤醒空闲 CPU 的平均次数，此值在 Windows 下永远返回 0

第一次调用 `process.getCPUUsage()` 时这两个值都是 0，后面每次调用获得值为本次调用与上次调用之间这段时间内相应 CPU 使用率和唤醒空闲 CPU 的平均次数，所以上面代码要用定时器来不断请求 CPUUsage 对象。

Electron 提供的硬件信息 API 太简单了，如果需要获得更详细的硬件信息可以使用 [systeminformantion](https://github.com/sebhildebrandt/systeminformation) 这个库。

### 使用硬件串号控制应用分发

开发商业桌面 GUI 应用，有时候需要控制应用分发的范围，比如用户购买某软件的使用权后，应用开发商只允许用户在某一台固定的物理设备上使用该软件，不允许用户随意地在其他设备上安装并使用。如果用户希望在另一台设备上可以使用，则需要另外购买使用权。

如果开发者想让自己开发的软件具备这种能力，一种常见的办法是获取用户设备的专有硬件信息，并把这个信息与当前使用该软件的用户信息、用户的付费情况信息绑定，保存在一个服务器上。当用户打开软件时，软件获取这个物理设备的专有硬件信息，并把这个信息连同当前用户信息发送到服务端，由服务端确认该用户是否已为该设备购买了授权，如果没有则通知应用，要求用户付费。

这种方式有两个弊端，一是由于验证过程强依赖于服务端，所以这个应用必须联网，对于一些有离线使用需求的应用来说，这个方案显然行不通。除此之外，一些恶意用户完全可以自己开发一个简单的服务，代理这个验证授权的请求，这个简单的服务只要永远返回验证授权通过的结果就能让恶意用户免费使用该软件。

对于离线应用来说，开发者可以通过一个安全的算法来保证应用只被安装在一台设备上，具体实现过程为：当应用第一次启动时，应用获取到这个物理设备的专有硬件信息，并把这个信息发送到服务端，用户付费后，服务端通过算法生成一个与该硬件信息匹配的激活码，并把这个激活码发送给用户，由用户把激活码保存在应用内。以后用户每次启动时，应用通过同样算法验证激活码是否与当前设备的硬件信息匹配，如果匹配则授权成功，反之授权失败。

每次执行逻辑的关键点是服务端根据用户设备的专有硬件信息生成激活码的过程，和应用每次启动时检验激活码与硬件信息是否匹配的过程。这两个过程内的算法是要严格保密的，一旦被恶意用户窃取（或通过逆向工程分析出了算法逻辑），那么恶意用户就可以开发一个注册机来无限生成激活码，无限制地分发你的软件。

使用 [systeminformation](https://github.com/sebhildebrandt/systeminformation) 获取当前硬件中各个组件的串号，并依据这些串号的组合来保证硬件专有信息唯一：

```js
// 返回当前物理设备的所有组件的静态信息，包括内存、CPU、磁盘网卡等组件的生产厂家、型号、硬件串号等信息
const staticData = await si.getStaticData();

const serial = {
  // 系统串号
  systemSerial: staticData.system.serial,
  // 主板串号
  baseboardSerial: staticData.baseboard.serial,
  // 基座串号
  chassisSerial: staticData.chassis.serial,
  // 第一块磁盘的串号
  diskSerial: staticData.diskLayout[0].serialNum,
  // 第一个内存的串号
  memSerial: staticData.memLayout[0].serialNum,
};

const arr = await si.networkInterfaces();
const networkInterfaceDefault = await si.networkInterfaceDefault();
const [item] = arr.flter((v) => v.iface === networkInterfaceDefault);
// 默认网卡的 MAC 地址
serial.mac = item.mac;

// 开发者可以依据此字符串，保证当前设备是唯一的
let serialNumStr = JSON.stringify(serial);

console.log(srialNumStr);
```

通过上述代码得到的硬件串号字符串最好不要直接使用，而是通过哈希运算获得哈希值。

```js
const crypto = require('crypto');

const serialHash = crypto.createHash('sha256').update(serialNumStr).digest('hex');

console.log(serialHash);
```

## 参考资料

- [Electron Docs: screen API](https://www.electronjs.org/docs/api/screen)
- [Electron Docs: desktopCapturer API](https://www.electronjs.org/docs/api/desktop-capturer)
- [Electron Docs: powerMonitor API](https://www.electronjs.org/docs/api/power-monitor)
- [Electron Docs: powerSaveBlocker API](https://www.electronjs.org/docs/api/power-save-blocker)
- [W3C: Media Capture and Streams](https://w3c.github.io/mediacapture-main/#dom-mediatrackconstraints)
