---
nav:
  title: Electron
  order: 5
group:
  title: 应用开发
  order: 1
title: 通信管理
order: 7
---

# 通信管理

## 与 Web 服务器通信

### 禁用通源策略以实现跨域

禁用 Electron 的同源策略

```js
const win = new BrowserWindow({
  width: 800,
  height: 600,
  webPreferences: {
    nodeIntegration: true,
    // 此参数禁用当前窗口的同源策略
    webSecurity: false,
  },
});
```

相似地，`webPreferences` 配置项下还有一个 `allowRunningInsecureContent` 参数，如果把它设置为 `true`，那么你就可以在一个 HTTPS 页面内访问 HTTP 协议提供的服务了，这在默认情况下也是不被允许的。当开发者把 `webSecurity` 设置为 `false` 时，`allowRunningInsecureContent` 也会被自动设置为 `true`。

### Node.js 访问 HTTP 服务的不足

Electron 为我们提供了一个 [net](https://www.electronjs.org/docs/api/net) 模块，允许我们使用 Chromium 的原生网络库发出 HTTP 或 HTTPS 请求，它内部会自动判断请求地址是基于什么协议的。

```js
const { app } = require('electron');

app.whenReady().then(() => {
  const { net } = require('electron');
  const request = net.request('https://github.com');

  request.on('response', (response) => {
    console.log(`STATUS: ${response.statusCode}`);

    console.log(`HEADERS: ${JSON.stringify(response.headers)}`);

    response.on('data', (chunk) => {
      console.log(`BODY: ${chunk}`);
    });

    response.on('end', () => {
      console.log('No more data in response.');
    });
  });
  request.end();
});
```

### 截获并修改网络请求

如果只是要拦截 AJAX 请求，那么为第三方网页注入一个脚本，让这个脚本修改 XMLHttpRequest 对象以获取第三方网页 AJAX 请求后的数据即可。

```js
const open = window.XMLHttpRequest.prototype.open;

window.XMLHttpRequest.prototype.open = function (method, url, async, user, pass) {
  this.addEventListener(
    'readystatechange',
    function () {
      if (this.readyState === 4 && this.status === 200) {
        // 服务端响应的数据
        console.log(this.responseText);
      }
    },
    false
  );
  open.apply(this, arguments);
};
```

在网页加载之初，注入并执行上面的代码，就为接下来的每个 AJAX 请求都注册了一个监听器，开发者可以通过 `open` 方法的 `method`、`url` 等参数过滤具体的请求，以实现在不影响第三方网页原有逻辑的前提下，精准截获 AJAX 响应数据的目的。

这种方法虽然可以正确截获 AJAX 请求的响应数据，但对结果静态文件请求及修改响应数据无能为力。如果开发者想获得这方面的能力，可以使用 Electron 提供的 [WebRequest](https://www.electronjs.org/docs/api/web-request) 对象的方法：

```js
const { BrowserWindow } = require('electron');

const win = new BrowserWindow({ width: 800, height: 600 });

win.webContents.session.webRequest.onBeforeRequest(
  {
    urls: ['https://*/*'],
  },
  (details, cb) => {
    if (details.url === 'https://targetDomain.com/vendors.js') {
      cb({
        redirectURL: 'http://yourDomain.com/vendors.js',
      });
    } else {
      cb({});
    }
  }
);

win.loadURL('http://github.com');
```

在上面的代码中，我们使用了 WebRequest 的 `onBeforeRequest` 方法监听第三方网页的请求，当请求发生时，判断请求的路径是否为我们关注的路径，如果是，则把请求转发到另一个地址，新的地址往往是我们服务器的一个地址。

### 修改 UserAgent

在加载 URL 的时候更改 User-Agent：

```js
win.webContents.loadURL('https://www.baidu.com', {
  userAgent: 'Mozilla/5.0(Windows NT 10.0; Win64; x64; rv:69.8) Gecko/20200112 Firefox/69.8',
});
```

除了修改请求头，你的脚本源码最好也加密，免得留下蛛丝马迹。

可以修改 `app.userAgentFallback` 属性的值来全局设置 User-Agent，修改后你的 Electron 程序内的所有请求的会使用此 UA。

## 与系统内其他应用通信

### Electron 应用于其他应用通信

系统内进程间通信一般会使用 IPC 命名管道技术实现（此类技术在 UNIX 系统下被称为域套接字），Electron 并没有提供相应的 API。

ICP 命名管道区分客户端和服务端，其中服务端用于监听和接收数据，客户端主要用于连接和发送数据。服务端和客户端时可以做到持久双向通信的。

假设有一个第三方的程序，需要发送数据给我们的程序，我们可以在 Electron 应用中创建一个命名管道服务以接收数据，代码如下：

```js
const net = require('net');

const PIPE_PATH = '\\\\.\\ pipe\\ mypipe';

const server = net.createServer(function (conn) {
  conn.on('data', (data) => console.log(`接收到数据：${data.toString()}`));
  conn.on('end', () => console.log('客户端已关闭连接'));
  conn.write('当客户端建立连接后，发送此字符串数据给客户端');
});

server.on('close', () => console.log('服务关闭'));
server.listen(PIPE_PATH, () => console.log('服务启动，正在监听'));
```

假设第三方程序开启了命名管道服务，需要我们的程序给它发送数据，那么可以在我们的应用中创建一个命名管道客户端：

```js
const net = require('net');

const PIPE_PATH = '\\\\.\\ pipe\\ mypipe';

const client = net.connect(PIPE_PATH, () => {
  console.log('连接建立成功');
  client.write('这是我发送的第一个数据包');
});

client.on('data', (data) => {
  console.log(`接收到的数据：${data.toString()}`);
  client.end('这是我发送的第二个数据包，发送完之后我将关闭');
});

client.on('end', () => console.log('服务端关闭了连接'));
```

除了命名管道外，你还可以通过 socket、剪切板、共享文件（通过监控文件的变化来实现应用间通信）等方法来与其他进程通信，但最常见的还是命名管道。

### 网页与 Electron 应用通信

在 Electron 应用程序内启动一个 HTTP 服务，然后再在网页内跨域访问这个 HTTP 服务，即可完成网页与 Electron 应用的通信。

```js
const http = require('http');
const server = http.createServer((request, response) => {
  if (request.url === '/helloElectron') {
    let jsonString = '';
    request.on('data', (data) => jsonString);
    request.on('end', () => {
      let post = JSON.parse(jsonString);

      // 处理业务逻辑
      response.writeHead(200, {
        'Content-Type': 'text/html',
        'Access-Control-Allow-Origin': '*',
      });

      const result = JSON.stringify({
        ok: true,
        msg: '请求成功',
      });
      response.end(result);
    });
    return;
  }
});

server.on('error', (err) => {
  // 此处可把错误信息发送给渲染进程，由渲染进程显示错误信息给用户
});

server.listen(9416);
```

程序通过 `http.createServer` 创建了一个 Web 服务，该服务监听本机的 9416 端口。注意，`createServer` 方法返回的 `server` 实例应该是全局对象或者挂在在全局对象，目的是避免被垃圾回收器回收。

当网页请求 `http://localhost:9416/helloElectron` 时，Electron 应用会接收到该请求，并获取到网页发送的数据，同时通知 `response` 的 `writeHead` 和 `end` 方法给网页响应数据。

在实际应用中，端口很可能被系统其他程序占用，所以如果商用产品一定要先确认端口可用，再进行监听。

你可以给 `server.listen` 方法传入 0 或什么都不传，让 Node.js 自动给你选一个可用的端口进行监听。一旦 Node.js 找到可用的端口，开始监听后，会触发 `listening` 事件。在此事件中你可以获取 Node.js 分配的端口号。

```js
server.on('listening', () => {
  console.log(server.address().port);
});

server.listen(0);
```

你可以把端口上报网页服务端，然后再由服务端下发给网页前端应用，此时网页与 Electron 应用通信，就知道要请求什么地址了。

## 自定义协议

Electron 允许开发者自定义类似 HTTP 或 File 的协议，当应用内出现此类协议的请求时，开发者可以定义拦截函数，处理并响应请求。

在自定义协议前，我们需要告知 Electron 我们打算声明一个怎样的协议。这个协议具备一定的特权，这里的特权时指该协议下的内容不受 CSP 限制，可以使用内置的 Fetch API 等。

```js
const { protocol } = require('electron');
const options = [{ shceme: 'app', privileges: { secure: true, standard: true } }];
```

此代码务必在程序启动之初，`app` 的 `ready` 事件触发之前执行，且只能执行一次。代码中，通过 `shceme` 属性指定了协议名称为 `app`，与 `http://` 类似。

注册这个自定义协议：

```js
const { protocol } = require('electron');
const path = require('path');
const { readFile } = require('fs');
const { URL } = require('url');

protocol.registerBufferProtocol(
  'app',
  (request, respond) => {
    let pathName = new URL(request.url).pathname;
    pathName = decodeURI(pathName);
    const fullName = path.join(__dirname, pathName);

    readFile(fullName, (err, data) => {
      if (error) console.log(error);

      const extension = path.extname(pathName).toLowerCase();
      let mimeType = '';

      switch (extension) {
        case '.js':
          mimeType = 'text/javascript';
          break;
        case '.html':
          mimeType = 'text/html';
          break;
        case '.css':
          mimeType = 'text/css';
          break;
        case '.json':
          mimeType = 'application/json';
          break;
      }

      respond({ mimeType, data });
    });
  },
  (err) => {
    if (err) {
      console.log(err);
    }
  }
);
```

通过 [protocol](https://www.electronjs.org/docs/api/protocol) 的 registerBufferProtocol 方法，注册了一个基于缓冲区的协议，`reigsterBufferProtocol` 方法接收一个回调函数，当用户发起基于 `app://` 协议的请求时，此回调函数会截获用户的请求。

在回调函数内，先获取到用户请求的文件的绝对路径，然后读取文件内容，接着按指定的 `mimeType` 响应请求。这其实就是一个简单的静态文件服务。

自定义协议注册完成之后，啾可以通过如下方式使用此自定义协议了：

```js
win.loadURL('app://./index.html');
```

## 使用 Socks5 代理

使用 Electron 内置的 socks 代理访问网络服务。

```js
const result = await win.webContents.session.setProxy({
  proxyRules: 'socks5://58.218.200.249:2071',
});

win.loadURL('http://www.ipip.net');
```

以上仅为单个渲染进程设置代理，如果你需要给整个应用程序设置代理，可以使用如下代码完成：

```js
app.commandLine.appendSwitch('proxy-server', 'socks5://58.218.200.249:2071');
```
