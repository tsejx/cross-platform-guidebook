---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 安全管理
order: 10
---

# 安全管理

## 保护源码

- 立即执行函数
- 禁用开发者调试工具
- 源码压缩与混淆

### 使用 ASAR 保护源码

`asar` 是一种将多个文件合并成一个文件的归档格式，且多文件打包后，还可以按照原来的路径难过读取打包后的内容。`asar` 格式的文档与 `tar` 格式的文档类似，但你不能使用解压缩文件将其解压。而 Electron 无需解压即可从其中读取任意文件内容。

### 使用 V8 字节码保护源码

[bytenode](https://github.com/bytenode/bytenode#readme)

## 保护客户

- 禁用 Node.js 集成
- 启用同源策略
- 启用沙箱隔离
- 禁用 webview 标签

## 保护网络

- 屏蔽虚书
- 关于防盗链

## 保护数据

### 使用 Node.js 加密解密数据

加密解密工作创建原始密钥和初始化向量：

```js
const crypto = require('crypto');
const key = crypto.scryptSync('这是我的密码', '生成密钥所需的盐', 32);

const iv = Buffer.alloc(16, 6);
```

`crypto.scryptSync` 方法会基于你的密码生成一个密钥，使用这种方法可以有效地防止密码被暴力破解。

`Buffer.alloc` 会初始化一个定长的缓冲区，这里我们生成的缓冲区长度为 16，填充值为 6。建议把 `key` 和 `iv` 保存成全局变量，避免每次加密、解密时执行重复工作。

对待加密的数据进行加密：

```js
const cipher = crypto.createCipheriv('aes-256-cbc', key, iv);
let encrypted = cipher.update('这是待加密的数据，这是待加密的数据');

encryptd = Buffer.concat([encrypted, cipher.final()]);
let result = encrypted.toString('hex');

console.log(result);
```

对密文解密：

```js
let encryptedText = Buffer.from(result, 'hex');
// 创建解密对象
let decipher = crypto.createDecipheriv('aes-256-cbc', Buffer.from(key), iv);
// 执行解密过程
let decrypted = decipher.update(encryptedText);
// 结束解密工作
decrypted = Buffer.concat([decryped, decipher.final()]);

result = decrypted.toString();

console.log(result);
```

注意，加密、解密过程必须使用一致的密钥、初始化向量和加密、解密算法。

如果每次加密完之后都会在可预期的未来执行解密过程，那么你也可以用下面这种方式生成随机的密钥和初始化向量。

```js
const key = crypto.randomBytes(32);
const iv = crypto.randomBytes(16);
```

这种方法会随机生成 `key` 和 `iv`，加密、解密过程都应使用它们。如果这两个值可能会在解密前被丢弃，那么就不应该使用这种方法。

对敏感数据进行加密可以保证你的数据不会被恶意用户窃取或篡改，你可以放心地把加密后的数据保存在客户端本地电脑上或在网络上传输（服务端使用对应的解密算法对密文进行解密）。

如果需要把密钥保存在客户电脑上，可以考虑使用 [node-keytar](https://github.com/atom/node-keytar) 库。这是一个原生 Node.js 库，它帮助你使用本地操作系统密钥管理工具来管理你的密钥，Mac 系统上使用的是钥匙串工具，Windows 系统上使用的是用户凭证工具。
