---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 发布管理
order: 11
---

# 发布管理

## 生成图标

在编译打包前，将应用程序的图标存放在 `[your_project_path]/public` 目录下，建议采用 1024 \* 1024 尺寸的 PNG 格式的图片。

使用 [electron-icon-builder](https://github.com/safu9/electron-icon-builder) 自动生成各种大小的图标文件。

```json
{
  "script": {
    "build-icon": "electron-icon-builder --input=./public/icon.png --output=build --flatten"
  }
}
```

## 生成安装包

常用打包构建工具：

- [electron-packager](https://github.com/electron/electron-packager)
- [electron-builder](https://github.com/electron-userland/electron-builder)

`electron-builder` 内置自动升级机制，把打包出来文件随意存储到 Web 服务器上即可完成自动升级。

## 代码签名

## 自动升级
