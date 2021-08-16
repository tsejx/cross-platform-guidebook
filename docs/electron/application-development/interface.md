---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 界面管理
order: 4
---

# 界面管理

## 页面内容

主进程中获取窗体对象

```js
const { BrowserWindow } = require('electron');

const win = new BrowserWindow({ width: 800, height: 1500 });
win.loadURL('http://github.com');

const contents = win.webContents;
console.log(contents);
```

主进程中获取处于激活状态下的窗体实例

```js
const webContent = win.webContents.getFocusedWebContents();
```

### 渲染进程中的事件

| 顺序 | 事件                    | 说明                                                                                                                                  |
| :--- | :---------------------- | :------------------------------------------------------------------------------------------------------------------------------------ |
| 1    | `did-start-loading`     | 页面加载过程中的第一个事件。如果该事件在浏览器中发生，那么意味着此时浏览器 Tab 页的旋转图标开始旋转，如果页面发生跳转，也会触发该事件 |
| 2    | `page-title-updated`    | 页面标题更新事件，事件处理函数的第二个参数为当前的页面标题                                                                            |
| 3    | `dom-ready`             | 页面中的 DOM 加载完成时触发                                                                                                           |
| 4    | `did-frame-finish-load` | 框架加载完成时触发，页面中可能会有多个 `iframe`，所以该事件可能会被触发多次，当前页面为 `mainFrame`                                   |
| 5    | `did-finish-load`       | 当前页面加载完成时触发。注意，此事件在 `did-frame-finish-load` 之后触发                                                               |
| 6    | `page-favicon-updated`  | 页面 Icon 图标更新时触发                                                                                                              |
| 7    | `did-stop-loading`      | 所有内容加载完成时触发，如果该事件在浏览器中发生，那么意味着此时浏览器 Tab 页的旋转图标停止旋转                                       |

`dom-ready` 事件其实就是网页的 DOMContentLoaded 事件

- 如果当前页面没有 `<script>` 标签，那么当前页面中的文本加载完成后立马触发 `dom-ready` 事件，而不会去管引入的外部文件
- 如果页面中有 `<script>`，则会等待 `<script>` 标签加载并解析完成后才触发 `dom-ready` 事件
- 如果 `<script>` 前面还有 CSS 资源，那么会等 CSS 加载成功后再加载 JS，JS 成功后再触发 `dom-ready` 事件
- 如果页面中还存在 `iframe`，`dom-ready` 会无视 `iframe` 的存在

### 页面跳转事件

| 事件                      | 说明                                                                                                                                                                                                                                                                    |
| :------------------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `will-redirect`           | 当服务端返回一个 301 或 302 跳转后，浏览器正准备跳转时，触发该事件。Electron 可以通过 `event.preventDefault` 取消此事件，禁止跳转继续执行                                                                                                                               |
| `did-redirect-navigation` | 当服务端返回一个 301 或 302 跳转后，浏览器开始跳转时，触发该事件。Electron 不能取消此事件，此事件一般发生在 `will-redirect` 之后                                                                                                                                        |
| `did-start-navigation`    | 用户点击了某个跳转链接或者 JavaScript 设置了 `window.location.href` 属性，页面（包含页面内任何一个 `frame` 子页面）发生页面跳转之时，会触发该事件，此事件一般发生在 `will-navigate` 之后                                                                                |
| `will-navigate`           | 用户点击了某个跳转链接或者 JavaScript 设置了 `window.location.href` 属性，页面发生跳转之前，触发该事件。然而当调用 `webContents.loadURL` 和 `webContents.back` 时并不会触发该事件。此外，当更新 `window.location.hash` 或者用户点击了一个锚点链接时，也并不会触发该事件 |
| `did-navigate-in-page`    | 当更新 `window.location.hash` 或者用户点击了一个锚点链接时，触发该事件                                                                                                                                                                                                  |
| `did-frame-navigate`      | 主页面（主页面 main frame 也是一个 frame）和子页面跳转完成时触发。当更新 `window.location.hash` 或者用户点击一个锚点链接时，不会触发该事件                                                                                                                              |
| `did-navigate`            | 主页面跳转完成时触发该事件（子页面不会）。当更新 `window.location.hash` 或者用户点击了一个锚点链接时，并不会出发该事件                                                                                                                                                  |

### 页面缩放

默认，用户可以通过 `Ctrl + Shift + 加号` 快捷键放大网页，`Ctrl + 减号` 缩小网页。

```js
const { BrowserWindow } = require('electron');

const win = new BrowserWindow({ width: 800, height: 1500 });
win.loadURL('http://github.com');

const contents = win.webContents;

// 缩小到 30%
contents.setZoomFactor(0.3);

// 获取当前缩放比例
const zoomFactor = contents.getZoomFactor();

console.log(zoomFactor);
```

通过 `setZoomFactor` 设置缩放比例，默认为 1（100%），如果参数值大于 1，则放大，小于 1 则缩小。

还有另外一个方法 `setZoomLevel(0)`，按等级缩放，0 为不缩放，传入 `int` 数字代表缩放等级，缩放比例等于缩放等级乘以 1.2，例如传入 1，则相当于放大到 120%，如果要缩小，则传负数。

如果需要控制用户缩放网页等级范围，可以通过 `setVisualZoomLevelLimits()` 方法设置，第一个参数为最小缩放等级，第二个参数为最大缩放等级。设置为一样就不能缩放了。

### 渲染海量数据

单页面渲染十万行数据，如果靠 Web 浏览器渲染是很卡的。

解决方案，分屏渲染：

当界面加载完成后只渲染一屏数据，但为这一屏数据加一个足够长的滚动条，接着监听滚动条的滚动事件。当滚动条向下滚动时，更新这一屏的数据，把头部几行内容丢弃掉，尾部创建几行新内容；当向上滚动时，把尾部的丢弃为头部找回那几行数据，这样做出来就和传统桌面开发的性能一样了。这个方案有几个细节：

1. 让容器出现一个滚动条的方法就是给容器加一个额外的 DOM 元素，高度设置为足够高就出现滚动条了，具体多高根据你数据的总行数与没行设置的高度计算得出。
2. 因为滚动条是一个额外的 DOM 元素，首屏数据是没用滚动条的，所以当用户在数据区域滚动鼠标轮时，要控制容器的滚动条，使数据区域与滚动条看起来像一个整体。
3. 当滚动条滚动时到底要增删几条数据合适呢？这需要先获取用户滚动距离，然后用这个距离除以滚动条的总高度得到滚动距离在总高度中的占比，然后让数据总行数乘以上这个占比，就得到了需要增删的行数了。但是这个计算是有误差，不过没关系，能滚就行，这个误差只会在滚动到顶部或者最底部才会产生影响，只需要在这个时候修正误差就行，即这时候把剩余的全部数据都显示出来。
4. 如果窗口放大或者缩小会导致首屏数据区域发生变化，如果发生了变化就需要重绘数据。

监听用户滚动条代码：

```js
let scrollDom = document.querySelector("#scrollDom"); // 此元素负责创建滚动条
let dataDom = document.querySelector(:#dataDom:); // 此元素放一次渲染的数据
scrollDom.onscroll = () => {
// 滚动条滚动触发的逻辑
};

dataDom.onwheel = (e) => {
scrollDom.scrollTop = scrollDom.scrollTop + e.deltaY;
}
```

当用户在数据区域滚动鼠标滑轮也会触发 `scrollDom` 的 `onscroll` 事件的。对应 HTML 代码如下：

```html
<div class="tableBox">
  <div class="dataTable">
    <table>
      <thead>
        <tr>
          <td>Your Column1</td>
          <td>Your Column2</td>
          <td>Your Column3</td>
        </tr>
      </thead>
      <tbody id="dataDom"></tbody>
    </table>
  </div>
  <div class="rightScroller" id="scrollDom">
    <div>
      <!-- 此元素高度动态计算 -->
    </div>
  </div>
</div>
```

如果对个性化定制要求不高，那么可以使用 [cheetah-grid](https://github.com/future-architect/cheetah-grid) 这个组件，这个开源项目的解决思路与本文介绍的一致，首屏是由 Canvas 技术渲染。

另外其他大数据渲染库，[PixiJS](https://github.com/pixijs/pixi.js) 这个库对 WebGL API 进行了二次封装，并与 Canvas 兼容，拥有强大硬件加速能力，谷歌、YouTube、Adobe 等都有在用这个库。

## 页面容器

### webFrame

`webFrame` 相当于浏览器的 `iframe`，用作嵌套页面。Electron 中即使一个页面中没有嵌套任何子页面他也是存在 `webFrame` 的，因为它本身就是一个 `webFrame` 实例，即主 `webFrame`（mainFrame）。`webFrame` 类型和实例只能在渲染进程中使用：

```js
const { webFrame } = require('electron');

// 以下两个方法和两个属性都能找到当前页面嵌套的 webFrame
webFrame.findFrameByName(name);
// routingId = webFrame.routingId
webFrame.findFrameByRoutingId(routingId);
webFrame.firstChild;
webFrame.nextSlibing;
```

前面说的 `did-frame-navigate` 事件就是页面中任何一个 `webFrame` 实例跳转完毕后触发的，可以通过 `webContents.isLoadingMainFrame()` 方法来判断 `mainFramne` 是否加载完毕。

在子 `webFrame` 中通过 `require` 函数引入其他 JS 库，会报错：`Uncaught ReferenceError: require is nodt defined`。

解决方法 1，在父 `webFrame` 中加入以下代码：

```js
let iframe = document.querySelector('#frameId');
// 子页面加载完后把 require 方法注入给他
iframe.onload = function () {
  let iframeWin = iframe.contentWindow;
  iframeWin.require = window.require;
};
```

解决方法 2：在创建窗口的时候把 `nodeIntegrationInSubFrames` 属性设置为 `true` 即可：

```js
let win = new BrowserWindow({
  webPreferences: {
    nodeIntegration: true,
    nodeIntegrationInSubFrames: true,
  },
});
```

### BrowserView

`webFrame` 是 BUG 挺多，窗体自用，`webview` 是官方不推荐，如果窗体要嵌入子页面，推荐的是使用 BrowserView。

BrowserView 被设计成子窗口的形式，依托 BrowserWindow 存在，可以绑定到 BrowserWindow 的一个具体区域，可随 BrowserWindow 一起缩放，随 BrowserWindow 移动而移动，看起来就像是 BrowserWindow 里面的一个 DIV 区块一样。

```js
// Main Process
let view = new BrowserView({
    webPreferences : {
        nodeIntegration: true,
    }
});
win.setBrowserView(view); // 给渲染进程设置一个BrowserView容器
let size = win.getSize();
// 绑定BrowserView到具体位置
view.setBounds({
    x: 0,
    y: 80, // 窗口顶部留出80px高度给父窗体用来做做tab选项卡栏
    width: size[0],
    height: size[1] - 80 // 所以browserView的高度要减去这80px
})；
// 设置BrowserView跟随父窗口自动缩放
view.setAutoResize({
    width: true,
    height: true
});

view.webContents.loadURL(url);
```

如果需要支持多标签页，需要在程序运行过程中动态创建多个 BrowserView 来缓存和展现用户打开的多个标签页。用户切换标签页时通过控制对应 BrowserView 容器对象的显示和隐藏来实现。这很像安卓的 Activity，添加了多个 BrowserView 就相当于添加了多个 Activity，后添加的显示在最上面。

为了实现这个需求，我们不应该用 `win.setBrowserView` 来创建 BrowserView，应该改用 `win.addBrowserView()` 方法为窗口设置添加 BrowserView。因为 `setBrowserView` 会判断当前窗口是否已经有 BrowserView 对象，如果有则会替换掉原来的对象，当再切换回来的时候用需要重新加载重新渲染，而 addBrowserView 可以为窗口设置多个 BrowserView 对象。然后再来显示隐藏的时候就不用重新渲染。

```js
// 隐藏
win.removeBrowserView(viw);
// 移回来，不会重新渲染
win.addBrowserView(view);
```

还可以通过 CSS 代码来显示隐藏 BrowserView：

```js
// 显示
view.webContents.insertCSS('html{display:block}');
// 隐藏
view.webContents.insertCSS('html{display:none}');
```

BrowserView 对象也拥有 `webContents` 属性，因此可以像操作 BrowserWindow 一样操作 BrowserView。

> 用 BrowserView 做选项卡切换比直接用 `div` 做选项卡切换页面有什么优势？

因为 `div` 方式你是所有逻辑在一个页面中的，如果页面内容较多，比较复杂，比如要请求网络发送好几个请求，有好几个图表要绘制，那么打开这个页面速度就很慢了，如果修改 DOM 之后重新渲染这个庞大页面也慢。但是如果把它拆分成多个 BrowserView，写多个 `.html` 文件用多个 BrowserView 装起来，那么每个 BrowserView 只负责一小部分请求和页面渲染那就很快，就能做到秒开，这才像个桌面端应该有的性能，不然做出来是个鸡肋，效果比浏览器没什么提升。

## 脚本注入

### 通过 preload 参数注入脚本

可以把一段 JS 代码注入到目标网页中，而这个代码好像就是那个网页自带的一样，哪怕那个网页时第三方网页。

```js
let win = new BrowserWindow({
  webPreferences: {
    // 通过 preload 参数注入 JavaScript 代码，脚本文件使用绝对路径
    preload: yourJsFilePath,
    // 注入进去就算了，还开启 node 支持
    nodeIntegration: true,
  },
});
```

由于你不知道你开发的程序被用户安装到哪个路径下了，所以不能写死注入脚本的绝对路径，要动态获取，可以通过在主进程中的 `app` 对象获取应用程序所在目录：

```js
const { app } = require{'electron'};
let path = require('path');
let appPath = app.getAppPath();
let jsPath = path.join(appPath, 'preload.js');
```

其实通过全局变量 `__dirname` 就能得到 APP 所在安装路径。所以在整个 Electron 程序中，任何要使用到本地安装目录下的文件都需要通过全局变量获取 APP 所在路径然后拼接，用相对路径是不靠谱的。

### 通过 executeJavaScript 注入脚本

```js
window.webContents.on('dom-ready', function () {
  window.webContents.executeJavaScript(`
   var electron = require('electron')
   var rootView = require(${JSON.stringify(path)})
   var h = require('mutant/h')

   electron.webFrame.setVisualZoomLevelLimits(1, 1)

   var config = ${JSON.stringify(config)}
   var data = ${JSON.stringify(opts.data)}
   var title = ${JSON.stringify(opts.title || 'Patchwork')}

   document.documentElement.querySelector('head').appendChild(
    h('title', title)
   )

   var shouldShow = ${opts.show !== false}
   var shouldConnect = ${opts.connect !== false}

   document.documentElement.replaceChild(h('body', [
    rootView(config, data)
   ]), document.body)
  `);
});
```
