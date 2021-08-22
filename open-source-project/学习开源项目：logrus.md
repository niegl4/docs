[TOC]

## 学习开源项目：github.com/sirupsen/logrus(v1.5.0)，笔记

### 开始用起来

```go
package main

import (
    log "github.com/sirupsen/logrus"
)

func init() {
    log.SetFormatter(&log.JSONFormatter{})
    log.SetOutput(os.Stdout)
    log.SetLevel(log.DebugLevel)
}

func main() {
    log.WithFields(log.Fields{
        "fieldA": "A",
        "fieldB": 10,
    }).Info("这是一条info级别的日志")
    //如果fields可以复用
    logEntry := log.WithFields({
        "common": "common field",
    })
    logEntry.Info("复用logentry，记录日志")
    logEntry.Warn("复用logentry")
}
```

翻看源码，不难发现：exported.go文件中暴露的众多函数，内部都是std调用它的对应方法。std在init函数中被创建，它就是该开源项目内部的一个私有全局变量，类型是*Logger。可以把这里理解成：从面向过程，到面向对象吗？面向过程的函数式编程，提供开箱即用的功能函数。但其实内部，是一个全局私有变量在调用方法，即代码组织形式还是面向对象。

```go
//源码exported.go文件
var (
    std = New()
)

func WithFields(fields Fields) {
    return std.WithFields(fields)
}
...

//源码logger.go文件
func New() *Logger {
    return &Logger{
        Out: os.Stderr,
        Formatter: new(TextFormatter),
        Hooks: make(LevelHooks),
        Level: InfoLevel,
        ExitFunc: os.Exit,
        ReportCaller: false,
    }
}

func (logger *Logger) WithFields(fields Fields) *Entry {
    entry := logger.newEntry()
    defer logger.releaseEntry(entry)
    return entry.WithFields(fields)
}
```

### 深入理解

整个logrus项目，最重要的两个对象分别是：Logger和Entry。

Entry对象，有一个字段Logger，*Logger类型，指明它是哪个Logger对象。

Logger对象，有一个字段entryPool，sync.Pool类型，可以托管Entry对象。

*Logger的系列方法，如上面提到的WithFields方法，都是先试图从entryPool中获取一个entry对象（有则获取，没有则新建），然后根据这个基础版的entry，生成一个新的复杂版的entry，以便后续调用Info等方法。这样，只要Logger对象不被gc回收，那么这个基础版的entry就可以复用。

（logrus中，buffer_pool.go中，还有一个关于sync.Pool的例子）

```go
//根据newEntry和releaseEntry方法，可以参考学习sync.Pool的用法
//源码logger.go文件
func (logger *Logger) newEntry() *Entry {
    //先试图从entryPool中获取一个entry对象，有则获取
    entry, ok := logger.entryPool.Get().(*Entry)
    if ok {
        return entry
    }
    //没有则新建
    return NewEntry(logger)
}
func (logger *Logger) releaseEntry(entry *Entry) {
    entry.Data = map[string]interface{}{}
    logger.entryPool.Put(entry)
}

//源码entry.go文件
func NewEntry(logger *Logger) *Entry {
    //返回的是一个基础版的entry
    return &Entry{
        Logger: logger,
        Data: make(Fields, 6)
    }
}
```

关于sync.Pool的特点，目前只知道它是为了缓解gc的压力，能够复用对象。Get方法获取，Put方法释放。当指定New字段时，它的类型是 func() interface{}，pool中没有对象，Get方法获取会触发调用New字段函数。下面是关于sync.Pool的演示。

```go
type stuPool struct {
    pool sync.Pool
}

func stuSyncPool() {
    p := stuPool{}
    //pool中没有对象时，返回nil
    getObj := p.pool.Get()
    fmt.Printf("%v", getObj)
    
    //可以向pool中放入多个对象
    p.pool.Put(1)
    p.pool.Put(2)
    
    //pool中有对象时，get就可以获取到，貌似是其中随机一个
    getObj = p.pool.Get()
    fmt.Printf("%v", getObj)
}
func stuSyncPoolWithNewFunc() {
    p := stuPool{
        pool: sync.Pool{
            New: func() interface{} {
            	return 1  
            },
        }，
    }
    //如果指定了New函数，即使pool中没有对象，get时会触发调用New函数，返回一个interface{}类型的对象
    getObj = p.pool.Get()
    fmt.Printf("%v", getObj)
}

func main() {
    stuSyncPool()
    stuSyncPoolWithNewFunc()
}
```

### 良好的编程范式

#### 多入口与公共函数

日志有多个级别，很多逻辑都是重复的，logrus开放出所有日志级别对应的方法，方便使用。同时，又抽象出同一的入口方法，进行代码复用。

```go
//源码entry.go文件
func (entry *Entry) Info(args ...interface{}) {
    entry.Log(InfoLevel, args...)
}
func (entry *Entry) Warn(args ...interface{}) {
    entry.Log(WarnLevel, args...)
}
func (entry *Entry) Warning(args ...interface{}) {
    entry.Warn(args...)
}
...
//在Log方法中，统一进行级别过滤判断
func (entry *Entry) Log(level Level, args ...interface{}) {
    if entry.Logger.IsLevelEnabled(level) {
        entry.log(level, fmt.Sprint(args...))
    }
}
//私有log方法，才是真正的记录日志的入口
func (entry *Entry) log(level Level, msg string) {
    ...
    if entry.Time.IsZero() {
        entry.Time = time.Now()
    }
    ...
    entry.write()
    ...
    if level <= PanicLevel {
        panic(&entry)
    }
}
```



#### 接口的良好实践

logrus自带两种fomat规则，分别是JSONFormatter和TextFormatter，它们都实现了Formatter接口，该接口制定了格式化输出的规范。

Logger对象的一个字段是Formatter接口类型，写日志的时候，就是拿这个字段调用接口中的方法。

所以，这种把接口暴露出去，可以很方便的实现用户自定义格式化输出。这算是接口类型的一个很好的实践样例。

```go
//源码formatter.go文件
type Formatter interface{
    Format(*Entry) ([]byte, error)
}

//源码json_formatter.go文件
type JSONFormatter struct {
    ...
}
func (f *JSONFormatter) Format(entry *Entry) ([]byte, error) {
    //data是一个map
    data := make(Fields, len(entry.Datt)+4)
    ...
    var b *bytes.Buffer
    ...
    encoder := json.NewEncoder(b)
    encoder.SetEscapeHTML(!f.DisableHTMLEscape)
    if f.PrettyPrint {
        encoder.SetIndent("", "    ")
    }
    if err := encoder.Encode(data); err != nil {
        return nil, fmt.Errorf("failed to marshal fields to JSON, %v", err)
    }
    return b.Bytes(), nil
}
```

上面Format方法中，最值得学习的就是json包中的相关方法。以前只用过Marshal，Unmarshal。下面是关于json包的一些方法的演示。

```go
func stuJsonEnDe() {
    jsonStr1 := `
    {"Name": "Alice", "Age": 25}`
    var m map[string]interface{}
    //Unmarshal功能比较单一，它只能对标准json进行反序列化
    if err := json.Unmarshal([]byte(jsonStr1), &m); err != nil {
        fmt.Println(err)
        return
    } else {
        fmt.Println(m)
    }
    
    //注意：这里字符串并不是json，而是包含了多个json的字符串
    //Decoder依然可以解析，这是它比Unmarshal强大的地方
    jsonStr := `
    {"Name": "Alice", "Age": 25}
    {"Name": "Bob", "Age": 22}`
    reader := strings.NewReader(jsonStr)
    writer := os.Stdout
    
    //形参指定输入的对象，解码对应输入
    de := json.NewDecoder(reader)
    //形参指定输出的对象，编码对应输出
    en := json.NewEncoder(writer)
    
    for {
        var m map[string]interface{}
        //输入对象 ==> m
        if err := de.Decode(&m); err == io.EOF {
            break
        } else if err != nil {
            fmt.Println(err)
        }
        
        //输出对象 <== m
        if err := en.Encode(&m); err != nil {
            fmt.Println(err)
        }
    }
}

type Track struct {
    XmlRequest string `json:"xmlRequest"`
}
func (t *Track) JSON() ([]byte, error) {
    buffer := &bytes.Buffer{}
    encoder := json.NewEncoder(buffer)
    //false可以禁止对html中出现的<等符号进行转义
    encoder.SetEscapeHTML(false)
    err := encoder.Encode(t)
    return buffer.Bytes(), err
}
func stuSetEscapeHTML() {
    message := Track{}
    message.XmlRequest = "<car><mirror>XML</mirror></car>"
    fmt.Println("before call JSON", message)
    messageJSON, _ := message.JSON()
    fmt.Println("after call JSON", string(messageJSON))
}

func main() {
    stuJsonEnDe()
    
    stuSetEscapeHTML()
}
```



### 项目中的实际使用

每记录一次日志，就生成一个Entry对象，向带缓冲的通道发送。通道另一端，接收后向列表添加。当检测到结束标志时，再一次性输出。这样，单次http请求的日志内容，就可以集中打印，方便观看，方便定位问题。

但是这样有一个小的问题，那就是一次性打印的日志，时间并不是记录日志的时间，而是一次性打印的时间。

翻看源码，在func (entry *Entry) log(level Level, msg string)中，发现如下内容，它有做判断。所以稍加修改：在生成Entry对象时，就对它的Time字段赋予当前时间。这样，这个问题就解决了。

```go
...
if entry.Time.IsZero() {
  entry.Time = time.Now()
}
...
```



### 彩蛋

在学习logrus项目时，意外发现go语言中改变字符串颜色的方法。以下是演示内容。

```go
func main() {
    //\x1b是开始的标志，[数字m 表示颜色
    //下面的例子，会把整个字符串都打印为蓝色
    fmt.Printf("\x1b[%dm%s abc", 36, "INFO")
    fmt.Println()
    //\x1b[0m 表示恢复正常
    //下面的例子，只会把INFO打印为蓝色，abc还是正常
    fmt.Printf("\x1b[%dm%s \x1b[0mabc", 36, "INFO")
    fmt.Println()
}
```

