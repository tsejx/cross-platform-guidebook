---
nav:
  title: Hybrid
  order: 1
group:
  title: Native
  order: 2
title: JavaScriptCore
order: 5
---

# JavaScriptCore

JavaScript 引擎 / 虚拟机

JavaScriptCore 是一个 C++实现的开源项目。使用 Apple 提供的 JavaScriptCore 框架，你可以在 Objective-C 或者基于 C 的程序中执行 Javascript 代码，也可以向 JavaScript 环境中插入一些自定义的对象。JavaScriptCore 从 iOS 7.0 之后可以直接使用。

## 模块组成

JavaScriptCore 主要由以下模块组成：

- Lexer 词法分析器，将脚本源码分解成一系列的 Token
- Parser 语法分析器，处理 Token 并生成相应的语法树
- LLInt 低级解释器，执行 Parser 生成的二进制代码
- Baseline JIT 基线 JIT（just in time 实施编译）
- DFG 低延迟优化的 JIT
- FTL 高通量优化的 JIT

关于更多 JavaScriptCore 的实现细节，参考 [Webkit - JavaScriptCore](https://trac.webkit.org/wiki/JavaScriptCore)

## 内核组成

在 `JavaScriptCore.h` 中，我们可以看到这个：

```cpp
#ifndef JavaScriptCore_h
#define JavaScriptCore_h

#include <JavaScriptCore/JavaScript.h>
#include <JavaScriptCore/JSStringRefCF.h>

#if defined(__OBJC__) && JSC_OBJC_API_ENABLED

#import "JSContext.h"
#import "JSValue.h"
#import "JSManagedValue.h"
#import "JSVirtualMachine.h"
#import "JSExport.h"

#endif

#endif /* JavaScriptCore_h */
```

这里已经很清晰地列出了 JavaScriptCore 的主要几个类：

- JSContext
- JSValue
- JSManagedValue
- JSVirtualMachine
- JSExport

### JSVirtualMachine

### JSContext

### JSValue

---

**参考资料：**

- [深入剖析 JavaScriptCore](https://www.jianshu.com/p/e220e1f34a0b)
- [JavaScriptCore 全面解析](https://www.cnblogs.com/qcloud1001/p/10305293.html)
