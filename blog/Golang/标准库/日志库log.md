## Go语言标准库log

2021年9月29日

>  今天阅读了下Go语言官方的日志包源码，此处整理一下，并记录一下问题。

log包的代码量比较少，是学习Go编程风格的好素材。



### Logger

log包主要围绕`Logger`结构体，其提供打印日志的各种方法。

`Logger`主要有两种实现方式。

1. 包内默认实现的`std`
2. 用户调用`New()`自定义

#### **使用默认Logger**

```go
package main

import "log"

func main() {
    // Println函数，实际上在内部调用了名为std的Logger
	log.Println("这是一条很普通的日志。")
}
```

输出：

```
2021/09/29 16:32:27 这是一条很普通的日志。
```

#### **使用自定义Logger**

```go
package main

import "log"

func main() {
    // 通过New生成新的logger
    logger := log.New(os.Stdout, "<prefix>", log.Lshortfile|log.Ldate|log.Ltime)
	logger.Println("这是自定义的logger记录的日志。")
}
```

输出：

```
2021/09/29 16:35:35 这是自定义的logger记录的日志。
```



源码

Logger结构体

```go
type Logger struct {
	mu     sync.Mutex // 确保原子写
	prefix string     // 日志前缀，查询更方便
	flag   int        // 日志包含的参数，比如日期|时间等
	out    io.Writer  // 日志的输出者，实现了io.Writer的都行
	buf    []byte     // 需要输出的内容
}
```

包内标准Logger实现

```go
func New(out io.Writer, prefix string, flag int) *Logger { // 自定义Logger也是调用New函数
	return &Logger{out: out, prefix: prefix, flag: flag}
}
var std = New(os.Stderr, "", LstdFlags) // 这里调用New()使用了os.Stderr（标准错误输出）作为输出位置
```

两者调用方法的区别

```go
func (l *Logger) Println(v ...interface{}) {  // 自定义
    l.Output(2, fmt.Sprintln(v...)) 
}
func Println(v ...interface{}) { // std
	std.Output(2, fmt.Sprintln(v...))
}
```

可以看出，std实际上就是封装了一下Logger方法。

### 配置Logger

Logger提供了一些方法，可以配置日志。包括：

1. 日志输出位置。文件/标准输出等。
2. 日志前缀
3. 日志目录/日期/时间等。

#### flag选项

```go
const (
    // 控制输出日志信息的细节，不能控制输出的顺序和格式。
    // 输出的日志在每一项后会有一个冒号分隔：例如2009/01/23 01:23:23.123123 /a/b/c/d.go:23: message
    Ldate         = 1 << iota     // 日期：2009/01/23
    Ltime                         // 时间：01:23:23
    Lmicroseconds                 // 微秒级别的时间：01:23:23.123123（用于增强Ltime位）
    Llongfile                     // 文件全路径名+行号： /a/b/c/d.go:23
    Lshortfile                    // 文件名+行号：d.go:23（会覆盖掉Llongfile）
    LUTC                          // 使用UTC时间
    LstdFlags     = Ldate | Ltime // 标准logger的初始值
)
```

通过配置这些常量，实现改变日志信息细节的目的。

#### 配置前缀&详细信息

```go
func main() {
	log.SetFlags(log.Llongfile | log.Lmicroseconds | log.Ldate) // 设置细节
	log.Println("这是一条很普通的日志。")
	log.SetPrefix("[小王子]") // 设置前缀
	log.Println("这是一条很普通的日志。")
}
```

输出：

```
2021/09/29 17:42:03.109910 e:/public/.../index.go:10: 这是一条很普通的日志。
[小王子]2021/09/29 17:42:03.123410 e:/public/.../index.go:12: 这是一条很普通的日志。
```

源码

```go
func (l *Logger) SetFlags(flag int) {
	l.mu.Lock()
	defer l.mu.Unlock()
	l.flag = flag
}

func (l *Logger) SetPrefix(prefix string) {
	l.mu.Lock()
	defer l.mu.Unlock()
	l.prefix = prefix
}
```

可以看出是给Logger的字段赋值。

那么数据是如何组装出来的呢？上面可以看到，`Printxx`方法都是调用的`Output`方法，这里看一下源码：

```go
func (l *Logger) Output(calldepth int, s string) error {
	// ...
    if l.flag&(Lshortfile|Llongfile) != 0 {
        // ...
		_, file, line, ok = runtime.Caller(calldepth) // 获取日志所在文件名以及行
	}
	l.buf = l.buf[:0] // 最终的数据[]byte
	l.formatHeader(&l.buf, now, file, line) // l.buf组装刚才setFlags的各种信息
	l.buf = append(l.buf, s...) // append将要打印的数据
	_, err := l.out.Write(l.buf) // 调用io.Writer的Write方法，输出数据
	return err
}
```

上面可以看出，先组装传入参数对应的值，然后再拼接想要打印的数据。

再来看看`l.formatHeader`是如何实现的

```go
func (l *Logger) formatHeader(buf *[]byte, t time.Time, file string, line int) {
	if l.flag&Lmsgprefix == 0 {
		*buf = append(*buf, l.prefix...)
	}
	if l.flag&(Ldate|Ltime|Lmicroseconds) != 0 {
		// ...
	}
	if l.flag&(Lshortfile|Llongfile) != 0 {
		// ...
		*buf = append(*buf, file...)
		// ...
	}
	if l.flag&Lmsgprefix != 0 {
		*buf = append(*buf, l.prefix...)
	}
}
```

`formatHeader`主要作用就是根据配置，拼接数据。

这里以及其它很多地方，作者利用了位运算操作，提升性能减少代码量，后面可以学习一下，再输出一篇文章。

### 参考

[Go语言标准库log介绍]([Go语言标准库log介绍 | 李文周的博客 (liwenzhou.com)](https://www.liwenzhou.com/posts/Go/go_log/))

[Golang官网]([log package - log - pkg.go.dev](https://pkg.go.dev/log))
