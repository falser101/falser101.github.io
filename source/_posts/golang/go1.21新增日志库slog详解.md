---
title: go1.21新增日志库slog详解
author: falser101
date: 2023-10-07
category:
  - golang
tag:
  - golang
---

> slog 结构化日志记录

Go 1.21 中的新 log/slog 软件包为标准库带来了结构化日志记录。结构化日志使用键值对，因此可以快速可靠地解析、过滤、搜索和分析它们。对于服务器来说，日志记录是开发人员观察系统详细行为的重要方式，通常是他们调试系统的第一个地方。因此，日志往往数量庞大，快速搜索和过滤它们的能力至关重要

# 快速使用
```golang
package main

import "log/slog"

func main() {
    slog.Info("hello, world")
}

# 输出内容如下
2023/010/07 16:09:19 INFO hello, world
```

除Info之外，还有用于其他三个级别的函数 Debug 、 Warn 和 Error以及一个将级别作为参数的更通用 Log 的函数。在slog中，级别只是整数，因此不限于四个命名级别。例如，Info为0且Warn为4，因此，如果您的日志记录系统具有介于两者之间的级别，则可以使用2。

与 log 包不同，我们可以轻松地将键值对添加到我们的输出中，方法是将它们写入消息之后：
```golang
slog.Info("hello, world", "user", os.Getenv("USER"))

# 输出
2023/10/07 16:27:19 INFO hello, world user=falser
```

slog 顶级函数使用默认logger。我们可以显式获取此logger，并调用其方法

```golang
logger := slog.Default()
logger.Info("hello, world", "user", os.Getenv("USER"))
```

我们可以通过更改logger使用的处理程序来更改输出。 slog 带有两个内置处理程序。TextHandler以key=value的形式发出所有日志信息。此程序使用创建一个 TextHandler新的logger，并对Info该方法进行相同的调用：
```go
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))
logger.Info("hello, world", "user", os.Getenv("USER"))

# 输出
time=2023-10-07T16:56:03.786-04:00 level=INFO msg="hello, world" user=falser
```

JsonHandler:

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("hello, world", "user", os.Getenv("USER"))

# 输出
{"time":"2023-10-07T16:58:02.939245411-04:00","level":"INFO","msg":"hello, world","user":"falser"}
```

也可以自己实现slog.Handler接口

对于频繁执行的日志语句，使用该 Attr 类型和调用 LogAttrs 方法可能更有效。
```go
slog.LogAttrs(context.Background(), slog.LevelInfo, "hello, world", slog.String("user", os.Getenv("USER")))
```

```
slog.Info("message", "k1", v1, "k2", v2)
# 这种方式可读性更高不易出错
slog.Info("message", slog.Int("k1", v1), slog.String("k2", v2))
```