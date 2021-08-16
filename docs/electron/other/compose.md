---
nav:
  title: Electron
  order: 5
group:
  title: 拓展资料
  order: 6
title: 桌面端技术选型分析
order: 2
---

# 桌面端技术选型分析

Node.js 和 Chromiums 整合

- 难点：Node.js 事件循环基于 libuv，但 Chromium 基于 message pump
  - Chromium 集成到 Node.js：用 libuv 实现 message pump（nw）
  - Node.js 集成到 Chromium

延伸资料：

- https://electronjs.org/blog/electron-internals-node-integration
- https://www.youtube.com/watch?v=OPhb5GoV8Xk
- https://github.com/electron/electron/blob/master/shell/common/node_bindings.cc

为什么要开发桌面端：

- 更便捷的入口
- 离线可用
- 调用系统能力（通信、硬件。。。）
- 安全需求

Native（C++/C#/Objective-C）

- 高性能
- 原生体验
- 包体积小
- 门槛高
- 迭代速度慢

QT（DropBox、WPS）

- 基于 C++
- 跨平台（Mac、Windows、iOS、Android、Linux、嵌入式）
- 高性能
- 媲美原生的体验
- 门槛高
- 迭代速度一般

Flutter

- 跨端（iOS、Android、Mac、Windows、Linux、Web）
- PC 端在发展中（Mac > Linux、Windows）
- 基建少

NW.js

- 跨平台（Mac、Windows、Linux）
- 迭代快，Web 技术构建
- 源码加密、支持 Chrome 扩展
- 不错的社区
- 包体积大
- 性能一般

Electron

- 跨平台（Mac、Windows、Linux、不支持 XP）
- Web 技术构建
- 活跃的社区
- 大型应用案例
- 包体积大
- 性能一般

其他：

- Carlo
- WPF
- Chromium Embedded Framework（CEF）
- PWA

|            | Electron                               | Native                | QT                      | NW                                 |
| :--------- | :------------------------------------- | :-------------------- | :---------------------- | :--------------------------------- |
| 性能       | 1                                      | 3                     | 2                       | 1                                  |
| 安装包大小 | 1                                      | 3                     | 1                       | 1                                  |
| 原生体验   | 1                                      | 3                     | 2                       | 1                                  |
| 跨平台     | 3                                      | x                     | 3                       | 3                                  |
| 开发效率   | 3                                      | 1                     | 2                       | 3                                  |
| 人才储备   | 3                                      | 2                     | 2                       | 3                                  |
| 社区       | 3                                      | 2                     | 1                       | 2                                  |
| 适用场景   | 跨平台应用<br/>快速交付<br/>前端技术栈 | 专业应用<br/>性能最佳 | 跨平台应用<br/>性能最佳 | 跨平台<br/>快速交付<br/>前端技术栈 |

NVM 安装
