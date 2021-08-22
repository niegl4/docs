[TOC]

## 学习开源项目：gopkg.in/ini.v1(v1.62.0)，笔记

### 开始用起来

```go
package main

import (
    "gopkg.in/ini.v1"
)

func main() {
    basic()
}

func basic() {
    cfg, err := ini.Load("path-to-ini.ini")
    if err != nil {
        fmt.Printf("Fail to read file: %v", err)
        os.Exit(1)
    }
    
    //读取操作，默认分区可以使用空字符串表示
    fmt.Println("App Mode:", cfg.Section("").Key("app_mode").String())
    fmt.Println("Data Path:", cfg.Section("paths").Key("data").String())
    
    //对候选值做限制，读取的值不在候选列表中，则使用提供的默认值
    fmt.Println("Server Protocol:",
                cfg.Section("server").Key("protocol").In("http", []string{"http", "https"}))
    fmt.Println("Email Protocal:",
                cfg.Section("server").Key("protocol").In("smtp", []string{"imap", "smtp"}))
    
    //自动类型转换
    fmt.Printf("Port Number: (%[1]T) %[1]d\n", cfg.Section("server").Key("http_port").MustInt(9999))
    fmt.Printf("Enforce Domain: (%[1]T) %[1]v\n", cfg.Section("server").Key("enforce_domain").MustBool(false))
    
    //修改某个值然后保存到一个新的ini文件中
    cfg.Section("").Key("app_mode").SetValue("production")
    _ = cfg.SaveTo("path-to-new-ini.ini")
    
}
```

```
#possible values : production, development
app_mode = development

[paths]
# Path to where grafana can store temp files, sessions, and the sqlite3 db (if that is used)
data = /home/git/grafana

[server]
# Protocol (http or https)
protocol = http

# The http port to use
http_port = 9999

# Redirect to correct domain if host header does not match domain
# Prevents DNS rebinding attacks
enforce_domain = true
```



### 深入理解

根据给出的基本使用样例，翻看源码，可以发现比较重要的部分就是作者把ini文本转化为怎样的数据结构。

三个最为重要的类型：File，Section，Key。作者将多个数据源合并，都转化为一个File结构体。File内再包含多个Section，每个Section都对应ini文本中的一个“节”。Section内再包含多个Key，每个Key即ini文本中的一个k-v键值对。

仔细观察，可以发现作者的一些很好的构思。

1. 首先，每个Section都有一个\*File，每个Key都有一个\*Section，分别指向它们的上级类型。这样，当操作子类型时，就可以通过这个指针，快速回到上级。

2. 其次，File对象输出为ini文本时，需要按照它原本的顺序，这就要求数据结构能够按顺序定位Section和Key。作者这里的实现很有意思：File对象有一个[]string字段，有序地指明Section的名字顺序，但是map[string]\[\]\*Section中存储的是Section的指针切片，怎样定位一个具体的Section呢？它还有一个[]int字段，有序地指明Section在切片中的索引。如果拿横纵坐标来类比，[]string就是横坐标，[]int就是纵坐标。这样，按顺序遍历，同时取出横纵坐标，就可以按顺序操纵Section。

   Section的数据结构就比较简单了，不过值得学习的一个点是，对于一个Key，它提供了多个字段便于从多个角度来操作。比如，[]string字段，可以方便获取该Section有哪些Key；map[string]string字段，可以方便获取Key的k-v；map[string]\*Key，可以获取详细的Key对象。

   Key的数据结构就更简单了，基本只包括k，v，注释。

其实，只要debug几次，再对数据结构多熟悉一下，就会发现：基本使用中的大部分场景，都非常依赖数据结构的实现。换句话讲，数据结构理解清楚，它的基本使用场景，实现起来比较容易。

```go
//源码file.go文件
type File struct {
    ...
    sectionList []string
    sectionIndexes []int
    sections map[string][]*Section
    ...
}

//源码section.go文件
type Section struct {
    f *File
    Comment string
    name string
    Keys map[string]*Key
    KeyList []string
    KeysHash map[string]string
    ...
}

//源码key.go文件
type Key struct {
    s *Section
    Comment string
    name string
    value string
    ...
}
```



### 良好的编程范式

#### 多定制化入口与公共函数

加载操作，可能需要多种定制需求，ini提供多个定制化的函数，开箱即用。同时，又抽象出同一的入口方法，进行代码复用。

```go
//多种定制需求
func Load(source interface{}, others ...interface{}) (*File, error) {
    return LoadSource(LoadOptions{}, source, others...)
}
func LooseLoad(source interface{}, others ...interface{}) (*File, error) {
    return LoadSource(LoadOptions{Loose: true}, source, others...)
}
func InsensitiveLoad(source interface{}, others ...interface{}) (*File, error) {
    return LoadSource(LoadOptions{Insensitive: true}, source, others...)
}
func ShadowLoad(source interface{}, others ...interface{}) (*File, error) {
    return LoadSource(LoadOptions{AllowShadows: true}, source, others...)
}


//在LoadSource方法中，统一进行加载
func LoadSources(opts LoadOptions, source interface{}, others ...interface{}) (_ *File, err error){
    ...
    sources[0], err = parseDataSource(source)
    ...
    f := newFile(sources, opts)
    if err = f.Reload(); err != nil {
        return nil, err
    }
    return f, nil
}
```



#### 接口的良好实践--利用空接口统一解析多种数据类型，利用接口再把多种数据类型化归为一种情况

加载支持多种数据来源，作者使用interface{}来接收。然后在parseDataSource中，通过类型推断，分情况讨论，转换为不同的结构体。

这些结构体的作用，就是为后续操作提供数据源，所以作者让他们都实现了一个接口。这样，多种结构体又可以被看成是一种情况。

```go
//源码data_source.go文件
type dataSource interface{
    ReadCloser() (io.ReadCloser, error)
}

//对应数据源为文件的情况
type sourceFile struct {
    name string
}
func (s sourceFile) ReadCloser() (_ io.ReadCloser, err error) {
    return os.Open(s.name)
}

//对应数据源为[]byte的情况
type sourceData struct {
    data []byte
}
func (s *sourceData) ReadCloser() (io.ReadCloser, error) {
    return ioutil.NopCloser(bytes.NewReader(s.data)), nil
}

//对应数据源本身就是io.ReadCloser类型的情况
type sourceReadCloser struct {
    reader io.ReadCloser
}
func (s *sourceReadCloser) ReadCloser() (io.ReadCloser, error) {
    return s.reader, nil
}

//通过类型推断，分情况讨论，转换为不同的结构体
func parseDataSource(source interface{}) (dataSource, error) {
    switch s := source.(type) {
    case string:
        return sourceFile{s}, nil
    case []byte:
        return &sourceData{s}, nil
    case io.ReadCloser:
        return &sourceReadCloser{s}, nil
    case io.Reader:
        return *sourceReadCloser{ioutil.NopCloser(s)}, nil
    default:
        return nil, fmt.Errorf("error parsing data source: unknown type %q", s)
    }
}
```



#### 利用反射对结构体的字段进行赋值

ini项目的高阶用法中，包含了大量反射的使用。

```go
//利用反射对结构体的字段进行赋值
type StuReflectPerson struct {
    //基本数据类型
    Name string
    NamePtr *string
    
    Age int `ini:"age,omitempty",json:"age,omitempty",stu:"study-reflect-in-ini"`
    AgePrt *int
    
    Male bool
    MalePtr *bool
    
    //基本复合类型
    Names []string
    Ages []int
    Males []bool
    
    Note
    *NotePtr `ini:"-"`
}
type Note struct {
    Content string
    Cities []string
}
type NotePtr struct {
    Content string
    Cities []string
}
func StuReflectInIniProject() {
    p := &StuReflectPerson{}
    typ := reflect.TypeOf(p)
    val := reflect.ValueOf(p)
    if typ.Kind() == reflect.Ptr {
        //type接口和value结构体的基本类型是指针时，通常需要通过Elem转换为指针指向的基本类型
        typ = typ.Elem()
        val = val.Elem()
    }
    mapToField(val)
    
    fmt.Printf("%v\n", p)
}
func mapToField(val reflect.Value) {
    if val.Kind == reflect.Ptr {
        val = val.Elem()
    }
    //可以从value到type，但不能从type到value
    typ := val.Type()
    
    for i := 0; i < typ.NumField(); i++ {
        //第i个字段的详细type信息--structField，该字段的type接口是它的一个字段
        structField := typ.Field(i)
        //第i个字段的详细value信息
        valField := val.Field(i)
        
        typField := structField.Type
        isPtr := structField.Type.Kind() == reflect.Ptr
        if isPtr {
            typField := structField.Type.Elem()
        }
        
        //通过structField获取tag
        if tag := structField.Tag; tag != "" {
            fmt.Println(tag)
            fmt.Println(tag.Get("ini"))
        }
        
        isStruct := structField.Type.Kind() == reflect.Struct
        isStructPtr := structField.Type.Kind() == reflect.Ptr && structField.Type.Elem().Kind() == reflect.Struct
        if isStruct {
            fmt.Println("is a struct, do not need to New")
        }
        //注意：如果字段是结构体指针类型，需要此New操作
        if isStructPtr {
            valField.Set(reflect.New(structField.Type.Elem()))
        }
        
        switch typField.Kind() {
        case reflect.String:
            name := "nie"
            //指针字段和非指针字段的对比
            if isPtr {
                valField.Set(reflect.ValueOf(&name))
                //注意：指针类型字段，不能SetString...
                //panic: reflect: call of reflect.Value.SetString on ptr value
            } else {
                valField.SetString(name)
            }
            
        case reflect.Int:
        	ageInt64 := int64(28)
            ageInt := 28
            if isPtr {
                valField.Set(reflect.ValueOf(&ageInt))
            } else {
                valField.SetInt(ageInt64)
            }
            
        case reflect.Bool:
            male := true
            if isPtr {
                valField.Set(reflect.ValueOf(&male))
            } else {
                valField.SetBool(true)
            }
            
        case reflect.Struct:
            //结构体字段进行赋值时，必须要类型对应
            //在ini项目中，作者使用了一种巧妙的方法：结构体在section中找不到，那就把它当做time.Time；能找到，那就切换section，进行一个递归调用
            //因为section一定是与具体的struct对应的，通过这种方式，来确保结构体字段的类型正确。因为无论哪种结构体，kind返回的都是reflect.Struct。
            mapToField(valField)
            
        case reflect.Slice:
            //切片字段赋值，也比较麻烦
            slice := reflect.MakeSlice(valField.Type(), 2, 2)
            sliceOf := valField.Type().Elem().Kind()
            for j := 0; j < 2; j++ {
                switch sliceOf {
                case reflect.String:
                    slice.Index(j).Set(reflect.ValueOf("a"))
                case reflect.Int:
                    slice.Index(j).Set(reflect.ValueOf(1))
                case reflect.Bool:
                    slice.Index(j).Set(reflect.ValueOf(true))
                }
            }
            valField.Set(slice)
        } 	
    }
}

func main() {
    StuReflectInIniProject()
}

```

