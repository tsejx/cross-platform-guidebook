---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 调试测试
order: 9
---

# 调试测试

## 调试

### 自动追踪性能问题

在 `app` 的 `ready` 事件的回调函数中，通过 `contentTracing.startRecoding` 方法来启动性能监控。

```js
(async () => {
  const { contentTracing } = require('electron');

  await contentTracing.startRecording({
    include_categories: ['*'],
  });

  await new Promise((resolve) => setTimeout(resolve, 6000));

  const path = await contentTracing.stopRecording();

  console.log(`性能追踪日志地址：${path}`);

  createWindow();
})();
```

在 Chrome 浏览器中输入 `chrome://tracing`，加载 `contentTracing` 保存的日志文件，即可进行分析。

### 开发环境调试工具

devtron

### 生产环境调试工具

debugtron

## 日志

### 业务日志

electron-log

### 网络日志

netLog

### 崩溃日志

crashReporter
