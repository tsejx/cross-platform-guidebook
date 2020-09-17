---
nav:
  title: Hybrid
  order: 1
group:
  title: Native
  order: 2
title: iOS 通用链接
order: 10
---

# iOS 通用链接

iOS 9 之前，一直使用的是 URL Schemes 技术来从外部对 App 进行跳转，但是 iOS 系统中进行 URL Schemes 跳转的时候如果没有安装 App，会提示 `Cannot open Page` 的提示，而且当注册有多个 `scheme` 相同的时候，目前没有办法区分，但是从 iOS 9 起可以使用 Universal Links 技术进行跳转页面，这是一种体验更加完美的解决方案。

> 什么是 Universal Link（通用链接）

Universal Link 是 Apple 在 iOS 9 推出的一种能够方便的通过传统 HTTPS 链接来启动 APP 的功能。如果你的应用支持 Universal Link，当用户点击一个链接时可以跳转到你的网站并获得无缝重定向到对应的 APP，且不需要通过 Safari 浏览器。如果你的应用不支持的话，则会在 Safari 中打开该链接。

支持 Universal Link（通用链接） 先决条件：必须有一个支持 HTTPS 的域名，并且拥有该域名下上传到根目录的权限（为了上传 Apple 指定文件）。

## 技术优点

1. 之前的 Custom URL scheme 是 **自定义的协议**，因此在没有安装该 APP 的情况下是无法直接打开的。而 Universal Links 本身就是一个能够指向 Web 页面或者 APP 内容页的标准 Web Link，因此能够很好的兼容其他情况
2. Universal link 是 **从服务器上查询** 是哪个 APP 需要被打开，因此不存在 Custom URL scheme 那样名字被抢占、冲突的情况
3. Universal link 支持 **从其他 APP 中的 UIWebView 中跳转** 到目标 APP
4. 提供 Universal link 给别的 APP 进行 APP 间的交流时，对方并不能够用这个方法去检测你的 APP 是否被安装（之前的 `custom scheme URL` 的 `canOpenURL` 方法可以）

---

**参考资料：**

- [📖 Universal Links for Developers](https://developer.apple.com/ios/universal-links/)
- [📝 iOS 唤起 APP 之 Universal Link 通用链接（2019 年 11 月 06 日）](https://juejin.im/post/6844903988526055437)
- [Universal Link 前端部署采坑记](https://juejin.im/post/6844903493862588429)
- [Support Universal Links](https://developer.apple.com/library/archive/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW2)
