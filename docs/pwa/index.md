---
nav:
  title: PWA
  order: 6
group:
  title: PWA
  order: 1
title: PWA
order: 1
---

# PWA

PWA 的中文名叫做渐进式网页应用，早在  2014年， W3C 公布过 Service Worker 的相关草案，但是其在生产环境被 Chrome 支持是在 2015 年。因此，如果我们把 PWA 的关键技术之一 Service Worker 的出现作为 PWA 的诞生时间，那就应该是 2015 年。

自 2015 年以来，PWA 相关的技术不断升级优化，在用户体验和用户留存两方面都提供了非常好的解决方案。PWA 可以将 Web 和 App 各自的优势融合在一起：渐进式、可响应、可离线、实现类似 App 的交互、即时更新、安全、可以被搜索引擎检索、可推送、可安装、可链接。

需要特别说明的是，PWA 不是特指某一项技术，而是应用了多项技术的 Web App。其核心技术包括 `App Manifest`、`Service Worker`、`Web Push` 等等。

## 技术特点

- **渐进式**：适用于选用任何浏览器的所有用户，因为它是渐进式增强作为核心宗旨来开发的
- **自适应**：适合任何机型，包括桌面设备、移动设备、平板电脑或任何未来设备
- **连接无关性**：能够借助于服务工作线程在离线或低质量网络状况下工作
- **类似应用**：由于是在 App Shell 模型基础上开发，因此具有应用风格的交互和导航，给用户以应用般的熟悉感
- **持续更新**：在服务工作线程更新进程的作用下时刻保持最新状态
- **安全**：通过 HTTPS 提供，以防止窥探和确保内容不被篡改
- **可发现**：W3C 清单和服务工作线程注册作用域能够让搜索引擎找到它们，从而将其识别为应用
- **可再互动**：通过推送通知之类的功能简化了再互动
- **可安装**：用户可免去使用应用商店的麻烦，直接将对其最有用的应用保留在主屏幕上
- **可链接**：可通过网址轻松分享，无需复杂的安装

### 离线使用

PWA 另一项令人兴奋的特性就是可以离线使用,其背后用到的技术是 Service worker。

Service Worker 实际上是一段脚本，在后台运行。作为一个独立的线程，运行环境与普通脚本不同，所以不能直接参与 Web 交互行为。Service Worker 的出现是正是为了使得 Web App 也可以做到像 Native App 那样可以离线使用、消息推送的功能。

我们可以把 Service worker 当做是一种客户端代理。

## 实现原理

上面所说 PWA 可实现 Web App 添加至主屏、可实现离线缓存，在断网或弱网状态下依然可以使用一些离线功能，不影响 Web App 体验以及可实现用户在不打开浏览器情况下实现类似于原生 App 离线消息推送功能。     

> 那么对于这些实现，PWA 依赖于什么？     

主要依赖于 `manifest.json` 和 Service wWrker（在项目中可写为一个名为 `SW.js` 的文件并引入项目）     

`manifest.json` 以一个 JSON 格式的文件被引入项目，它主要用来实现 PWA 页面的添加至主屏、定义 App 启动时的 URL（因为 PWA App 本质上还是一个 Web）等。

---

**参考资料：**

- [📖  MDN：渐进式 Web 应用 PWA](https://developer.mozilla.org/zh-CN/docs/Web/Progressive_web_apps)
- [📝  PWA 学习与实践系列](https://juejin.im/post/6844903588267835406)
- [第一本 PWA 中文书](https://juejin.im/entry/6844903517103063053)
- [简单介绍 Progressive Web App（PWA）](https://juejin.im/post/6844903556470816781)
- [浅谈 PWA](https://juejin.im/post/6844903714960965645)
- [深入理解 PWA](https://juejin.im/post/6844903731461357582)
- [PWA 初探](https://juejin.im/post/6844903657775857672)
- [React 同构应用 PWA 改造实践](https://juejin.im/entry/6844903609046401032)