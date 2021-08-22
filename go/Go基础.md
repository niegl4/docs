写在前面：

短时间内上手一门编程语言，被认为是一个程序员的基本素质。回顾我学习Python，Golang的经历，发现Python的学习经历更坎坷一些，Go相对轻松一些。这一方面是因为Python是我接触的第一门编程语言，学习的时候抓不住重点，有些迷茫，另一方面是学习Go的时候，学习理念有了一些变化。此时此刻，我作为一个菜鸟程序员，对待一门新语言，已经不再要求自己掌握每一个细节。而是掌握主干，在实际项目中边学边熟悉，以追求快速上手。毕竟，熟悉一门语言，最好的方式是经常使用它，而不是死扣细枝末节。

当然，我的意思绝对不是忽略语言的细节。事实上，在一些主干知识点中，掌握细节就意味着将来少走弯路。考虑到现阶段我只掌握了Python和Go，马上会规划再学习Java和JavaScript。而Python由于最近一段时间没有使用，就有些生疏，所以我决定写下这一系列的总结，便于自己能够在生疏的时候快速重温语言的核心主干知识点。



[TOC]

## **变量**

三种声明方式：

1. var 变量名 变量类型

   这种方式声明后，并没有初始化，可以再进行赋值初始化。

2. var 变量名 = 表达式

   这种方式相对第一种省去了变量类型的描述，通过表达式来自动进行类型推断。

3. 变量名 := 表达式

   这种方式相对第二种又省去了var关键字，并且也是通过表达式来进行类型推断。注意：这种方式只能用来声明局部变量，而不能声明全局变量。前两种就没有该限制。

## **常量**

将变量声明的前两种方式中，var关键字替换为const关键字即可，规则不变。

------

- 整数类型值，整数常量之间的类型转换

  > 原则上，只要源值在目标类型的可表示范围内就是合法的。
  >
  > uint8(255)
  >
  > 源整数类型的范围大，目标类型的范围小，转换可能会发生溢出。
  >
  > var srcInt = int16(-255)
  >
  > dstInt := int8(srcInt)  

- 虽然直接把一个整数值转换为一个string类型的值是可行的，但值得关注的是，被转换的整数值应该可以代表一个有效的Unicode代码点，否则转换的结果将会是"?"（仅由高亮的问号组成的字符串值）。

  > 字符'?'（这里只是近似表示，实际是一个实心菱形，中间一个问号），它的Unicode代码点是U+FFFD。它是Unicode标准中定义的Replacement Character，专用于替换那些未知的，不被认可的以及无法展示的字符。
  >
  > string(-1)就会得到上文那个符号。

------



## **流程控制**

### 分支选择

- if

  ```go
  //完整的if; else if； else
  if condition1 {
      code block1
  } else if condition2 {
      code block2
  } else {
      code block3
  }
  //if与局部变量，以下例子中，err的作用域是if语句块
  //最常见的场景，json的反序列化，可以缩短代码行数
  obj ：= structA{}
  if err := json.Unmarshal([]byte("测试，不太规范"), &obj); err != nil {
      fmt.Println(err)
  }
  ```

- switch

  ```go
  //switch适合用于多分支判断的情况
  //switch与类型推断表达式联合使用
  var er interface{}
  switch tmp := er.(type) {
  case string:
      fmt.Println(tmp)
  case int:
      fmt.Println(tmp)
  }
  //switch不引用表达式，直接对值的情况进行判断。并结合fallthrough关键字
  var action string
  switch action {
  case "match":
      code block1
  case "export":
      code block2
  case "search":
      //表达：如果是search，则顺延至下一个case
      fallthrough
  default:
      code block3
  }
  ```

- select

  ```go
  //select又称之为go中的多路复用
  //它总是和并发编程的通道联系在一起，具体参见并发编程的部分
  ```

  

### 循环控制

- for循环

  ```go
  //go不支持while，仅支持for循环和for range循环
  for i := 0; i < n; i++ {
      code blok
  }
  //如果不通过计数结束循环，而想通过某个条件结束循环
  for {
      code blok1
      if condition {
          break
      }
  }
  
  //for range循环总是和go的容器数据类型(array, slice, map, channel)结合使用，参加下文
  ```

  

- 循环跳出

  go有强大的循环控制功能，有goto，contine，break，return，并且还可以和标签配合使用。我觉得这点比python强大。
  
  ```go
  //goto关键字
  //虽然有些人不推荐使用goto，而且它有一些限制，但我仍然觉得某些时候使用它还是很有用的
  func gotoTest() {
      for i := 0; i < 5; i++ {
          for i := 0; i < 5; i++ {
              if i == 4 {
                  goto GotoLabel
              }
          }
      }
  GotoLabel:
      //这里就是goto的限制，在goto关键字与标签之间，不能再声明变量，否则不能通过编译
      //如果把变量a移动到标签上方，则会报错
      a := "a"
      fmt.Println(a)
      fmt.Println("result")
  }
  
  //contine和break关键字
  //它们的作用和其他语言类似，只是go支持它们与标签配合使用，使得它们更加灵活。如果不和标签配合，它们的语义和python相同。
  //contine：结束当前的最内层遍历，开始下一次最内层遍历
  //break：跳出最内层遍历
  //以break为例，如果是多层循环，没有标签功能时，通常要设置flag来控制循环，比如python。和标签配合，会让代码更清晰明了。
  func breakTest() {
  BreakLabel:
      for i := 0; i < 5; i++ {
          for j := 0; j < 5; j++ {
              if j == 4 {
                  break BreakLabel
              }
          }
      }
  }
  //作为对比，不使用标签，提出外层循环
  func breakWithoutLabelTest() {
     for i := 0; i < 5; i++ {
         flag := false
         for j := 0; j < 5; j++ {
             if j == 4 {
                 flag = true
             }
         }
         if flag {
             break
         }
      }
  }
  
  //return关键字
  //有些时候，在函数中，直接使用return退出循环并退出函数，会更加简洁有力
  func checkStrInSlice(targetSli []string, str string) bool {
      for i, _ := range targetSli {
          if i == str {
              return true
          }
      }
      return false
  }
  //作为对比，不使用return退出循环
  func checkStrInSliceWithBreak(targetSli []string, str string) (res bool) {
      for i, _ := range targetSli {
          if i == str {
              res = true
              break
          }
      }
      return
  }
  ```


## **数组**

数组是**值类型**：声明后可以直接对它操作。

- 声明

  数组元素的个数和元素的类型，一起构成数组的类型。一旦声明，不能改变。

  ```go
  var array1 [3]int
  var array2 = [3]float32{1.1, 2.2, 3.3}
  //数组个数可以自动推断
  var array3 = [...]string{"ab", "AB"}
  //初始化时，可以指定索引进行初始化，未指定索引的元素，为元素类型的零值。
  var array4 = [...]int32{0:0, 3:3}
  ```

- 遍历

  ```go
  //for遍历
  var arr [...]{1, 2, 3, 4, 5}
  for i := 0; i < 5; i++ {
      fmt.Println(i)
  }
  
  //for range遍历(:=左边一个元素，则它为数组索引)
  for i := range arr {
      fmt.Println(i)
  }
  //for range遍历(:=左边两个元素，则第一个元素是数组的索引，第二个元素是遍历时的临时容器，类型为数组元素类型。对它进行的修改，不会影响数组。map，slice的遍历都有类似属性。这一点和Python很不一样。)
  for _, item := range arr {
      fmt.Println(i)
  }
  ```

## **切片**

切片是**引用类型**：声明后，还需进行初始化。

相对于数组，切片具有更大的灵活性，使用的更频繁。

- 声明

  元素类型，长度，容量，通常从这三个方面来考量切片。

  元素类型：与数组一样，需要明确元素类型。

  长度：肉眼可见的切片元素的个数。

  容量：切片底层是数组，从当前切片的起点索引，到底层数组的末尾索引，它们之间的元素个数。

  ```go
  //在实际使用中，更多的是使用make来声明并初始化切片。
  //make([]itemType, len, cap)
  
  //此时，len和cap都是5，并且它们都是int的零值，即0。
  //注意，这时候再对它进行append，会触发切片自动扩容，而初始的5个元素都还是0。
  //可以通过list1[index] = n，来修改初始的几个零值。
  list1 := make([]int, 5)  
  
  //也可以指定len为0，cap为5。这样就既可以append，也可以通过索引修改元素值了。
  list2 := make([]int, 0, 5)
  //值得注意的是，虽然切片可以自动扩容，但是却要消耗性能，所以make([]type, 0)是不太负责任的写法。即使无法推断容量，也可以大概指定一个值。
  ```

- 遍历

  与数组的遍历完全相同。

- append函数

  与Python提供很多关于列表的方法不同，Go关于切片可用的函数很少，最常用的就是append函数。

  在使用append函数前，只需对切片声明即可，不强求做初始化，这一点很方便。

  当切片底层的数组容量不够时，append函数会触发切片自动扩容。

  ```go
  var list1 []int
  var list2 = []int{1, 2, 3}
  list1 = append(list1, 1)
  //使用...进行“拆包”
  list1 = append(list1, list2...)
  ```

------

- 值类型与引用类型

  > 值类型：基础数据类型，数组类型，结构体类型。
  >
  > 引用类型：切片类型，字典类型，通道类型，函数类型。
  >
  > 从传递成本的角度讲，引用类型的值往往要比值类型的值低很多。

- string与[]byte，[]rune类型之间的互转

  > - 一个值在从string类型向[]byte转换时，代表着以UTF-8编码的字符串会被拆分成零散的，独立的字节。除了与ASCII编码兼容的那部分字符集，以UTF-8编码的某个单一字节是无法代表一个字符的。
  > - 一个值再从string类型向[]rune转换时，代表着字符串会被拆分成一个个Unicode字符。

- 切片的底层数组什么时候会被替换？

  > 确切地说，一个切片的底层数组永远不会被替。因为，虽然在扩容的时候Go语言一定会生成新的底层数组，但是它也同时生成了新的切片。
  >
  > 在无需扩容时，append函数返回的是指向原底层数组的新切片，而在需要扩容时，append函数返回的是指向新底层数组的新切片。
  >
  > 只要新长度不会超过切片的原容量，那么使用append函数对其追加元素的时候就不会引起扩容。这只会使紧邻切片窗口右边的（底层数组中的）元素，被新的元素替换掉。也就是，通过切片来影响底层数组的元素。
  >
  > ```go
  > //下面的例子用来演示，通过切片来影响底层数组
  > func main() {
  >     type Map map[string][]int
  >     m := make(Map)
  >     s := []int{1, 2}
  >     s = append(s, 3)
  >     m["a"] = s
  >     
  >     //切片不扩容时，新的s和map的值，都指向同一底层数组。
  >     //所以这里其实就是，操作s影响底层数组，进而影响所有指向该数组的切片。
  >     s = append(s[:1], s[2:]...)
  >     
  >     //作为对比，切片扩容时，新切片指向别的数组，map的值就不受影响了。
  >     s = append(s, 3, 3, 3, 3)
  >     
  >     fmt.Println(s)
  >     fmt.Println(m["a"])
  > }
  > ```
  >
  > 

------



## **映射**

map是**引用类型**：声明后，还需进行初始化。

- 声明

  键-值类型，容量，通常从这两个方面来考量map。

  与切片类似，最好在make时指定容量。

  ```go
  var map1 = map[string]int{"a": 1}
  map2 := make(map[string]int, 3)
  ```

- 遍历

  ```go
  //map无序，所以没有for遍历，只有for range遍历
  //只对key遍历
  for k := range map1 {
      fmt.Println(k)
  }
  
  //对key, value进行遍历
  for k, v := range map2 {
      fmt.Println(k, v)
  }
  ```

- 推断键是否存在

  ```go
  // :=左边只有一个变量，则如果存在，该变量为对应val的拷贝。如果不存在，该变量为对应类型的零值。
  val := map1["b"]
  fmt.Println(val)
  // :=左边有两个变量，第一个变量的规则同上。第二个变量为bool类型，如果存在，为true。如果不存在，为false。
  if val, ok := map1["c"]; !ok {
      fmt.Println(val)
  }
  ```

------

- 字典的键类型不能是哪些类型

  > - 键类型的值，必须要支持判等操作。
  >
  >   函数类型，字典类型，切片类型的值并不支持判等操作，所以字典的键类型不能是这些类型。
  >
  >   如果键类型是接口类型，那么接口类型的“实际类型”也不能是上述三种类型。
  >
  >   如果键类型是数组类型，那么该类型的元素类型，也不能是上述三种类型。
  >
  >   如果键类型是结构体类型，那么还要保证其中字段的类型的合法性。
  >
  > - 为什么呢？
  >
  >   每个哈希桶都会把自己包含的所有键的哈希值存起来。Go语言会用被查找键的哈希值与这些哈希值逐个对比，看看是否有相等的。如果一个相等的都没有，那么就说明这个桶中没有要查找的键值。如果有相等的，那就再用键值本身去对比一次。
  >
  >   为什么还要对比？不同值的哈希值是可能相同的，术语叫做“哈希碰撞”。所以，即使哈希值一样，键值也不一定一样。如果键类型的值之间无法判等，那么此时这个映射的过程就没办法继续下去了。
  >
  >   最后，只有键的哈希值和键值都相等，才能说明查找到了匹配的键-元素对。

------




## **函数**

- 定义

  函数是一种类型。

  函数传参是值拷贝，如果希望修改生效，需要传入指针。

  ```go
  //可变长形参只能出现一次，且在形参末尾。在函数内部，该参数会成为对应类型的切片
  func func1(arg1, arg2 int, arg3 ...bool) (bool, error) {
      code block1
  }
  
  //返回值命名，则表示已声明但未初始化，而且函数返回时可以只写return
  //返回值不命名，则函数返回时必须在return后面指明返回的变量
  func func2() (res bool, err error) {
      return
  }
  
  func func3() (bool, error) {
      return true, nil
  }
  ```

  

## **结构体**

- 声明与初始化

  结构体是值类型

  ```go
  type person struct {
      name, address string
      age uint8
  }
  //声明与初始化一起
  p1 := person{
      name: "a",
      address: "1",
  }
  //声明与初始化分离
  var p2 person
  p2.name = "b"
  p2.address = "2"
  
  //结构体指针，可以有三种方式
  p3 := &person{
      name: "c",
  }
  //
  var p4 *person
  p4.name = "d"
  //new
  p5 := new(person)
  p5.name = "e"
  ```

  

- 接受者，即结构体方法

  值接收者，指针接收者，都可以分别调用值接收者实现的方法，指针接收者实现的方法。

  但是一旦把方法抽象为接口类型，情况就会变化，参见接口部分的说明。

  ```go
  //同一个结构体，通常，要么都是值接收者方法，要么都是指针接收者方法
  //如果不需要修改结构体的字段值，可以使用值接收者，比如单次http请求中的请求参数。
  //如果需要修改结构体的字段值，可以使用指针接收者。
  //如果接收者是拷贝代价比较大的大对象，可以使用指针接收者。
  //值接收者
  func (p person) func1() {
      
  }
  
  //指针接收者
  func (p *person) func1() {
      
  }
  ```



## **并发**

工作中常用到的并发编程模型

### 不使用通道，使用锁或者并发安全map，来获取数据

优点：省去建立通道；可以通过闭包修改不同的值，进一步简化代码，但也要小心冲突，尤其是err。

缺点：使用了锁。

```go
type targetData struct {
    data string
    Err error
}

func task(arg1 int, res []*targetData, lock *sync.Mutex, wg *sync.WaitGroup) {
    defer wg.Done()
    if arg1 < 4 {
        //slice不是并发安全的，读写要加锁
        lock.Lock()
        res = append(res, &targetData{data: "a"})
        lock.Unlock()
    } esle {
        lock.Lock()
        res = append(res, &targetData{Err: errors.New("error")})
        lock.Unlock()
    }
    return
}

func main() {
    var (
        wg sync.WaitGroup
        lock sync.Mutex
        res = make([]*targetData, 0 , 10)
    )
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go task(i, res, &lock, &wg)
    }
    wg.Wait()
}
```

### 使用同步通道（缓冲值为0）获取数据

优点：不使用锁，适用范围更广，最常用也最简洁

缺点：需要创建通道元素类型，需要初始化通道

```go
type targetData struct {
    data string
    Err error
}

func task(arg1 int, resChan chan *targetData) {
    if arg1 < 4 {
        resChan <- &targetData{data: "a"}
    } esle {
        resChan <- &targetData{Err: errors.New("error")}
    }
    return
}

func main() {
    resChan := make(chan *targetData, 0)
    for i := 0; i < 10; i++ {
        go task(i, resChan)
    }
    //分发了几次任务，就接收几次任务的结果。不用使用wg
    for i := 0; i < 10; i++ {
        //当任务没有完成时，这里会阻塞
        res := <- resChan
        fmt.Println(res)
    }
}
```

### 带有context的并发模型

虽然第二种模型最常用，最通用，但是如果要精益求精，更加注重性能，那么推荐使用这种带有context的模型。

适用场景：多个子任务中，只要能获取到一个有效结果，那么其他子任务可以立即结束返回，而不必等到它们自然结束。

优点：耗时更短，尤其是子任务失败时耗时更长的情况。

缺点：适用场景较少，如果必须获取所有子任务的结果，那就不适用。会让代码看起来比较复杂，难以理解。

```go
type targetData struct {
    data string
    Err error
}

//这里模拟的是一个耗时任务，实际项目中，可能是一次http请求
func task(arg1 int, resChan chan *targetData) {
    if arg1 < 4 {
        resChan <- &targetData{data: "a"}
    } esle {
        resChan <- &targetData{Err: errors.New("error")}
    }
    return
}

func callTask(arg1 int, resChan chan *targetData, ctx context.Context, 
              wg *sync.WaitGroup) {
    //为了形成select多路复用，必须多创建一个本地通道
    localChan := make(chan *targetData, 0)
    //为了能够及时获取上层的cancel信息，必须形成select多路复用的模型，所以必须异步调用子任务
    go task(arg1, localChan)
    select {
        //最多四种情况：
        //1.无cancel,无结果
        //select阻塞
        //2.有cancel，无结果
        //此时本次任务产生的结果已经没有意义了，及时结束本次任务
        //3.无cancel，有结果
        //本次任务的结果要发送给调用者
        //4.同时有cancel，有结果
        //select会随机选择一个分支执行
        //本次任务的结果同样没有意义，无论是发送还是丢弃，都可以
    case res := <-localChan:
        //如果选择发送结果，无论本次结果是否还有意义，都可以发送，上层负责接收所有任务的结果
        resChan <- res
    case <- ctx.Done():
        //如果上层cancel，本次的结果丢弃，但不能不管已经启动的任务，异步从本地通道获取一次结果，让任务结束
        go func() {
            <- localChan
        }()
    }
    wg.Done()
    return
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    resChan := make(chan *targetData, 0)
    var wg sync.WaitGroup
    
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go calltask(i, resChan, ctx, &wg)
    }
    
    var result *targetData
    for i := 0; i < 10; i++ {
        res := <- resChan
        if res.Err != nil {
            continue
        }
        //所有的结果都不满足要求，那么不会修改result
        //任一子任务的结果满足要求，修改result，通知其余任务及时结束
        if res.data == "4" {
            result = res
            cancel()
            break
        }
    }
    //主线任务不能不管正在进行中的任务，要能够让他们正常结束
    //如果结果全都不满足要求，也就是上面已经取出所有子任务的结果时，这个匿名函数没有什么用处
    //如果结果有满足要求的，cancel通知时他们的任务也刚好结束（也就是出现情况四），匿名函数负责让select随机选择了发送结果分支的情况能够善终
    go func() {
        for {
            _, ok := <- resChan
            //通道被关闭，ok为false，那么这个善后的匿名函数，也就可以结束了
            if !ok {
                break
            }
        }
    }
    //到达这里，子任务仍然可能正在运行（当然，由于cancel，它们很快就会结束，而不是必须等待耗时调用完成）,所以需要等待
    wg.Wait()
    //所有子任务已经结束，关闭通道，让善后匿名函数结束它本身
    close(resChan)
    return
}
```

### 带有优先级的并发模型

对于同一问题，有方案A和方案B，方案A的优先级高于B。

具体体现为：

- A有结果且不报错，则用A的结果，立即结束方案B；

- A有结果但是报错，再分析B的结果；
  - B有结果且不报错，则用B的结果
  - B有结果但报错，则接口返回错误

```go
type result struct {
    planAResult string
    planBResult string
    err			error
}

func planA(err error, resChan chan *result, wg *sync.WaitGroup) {
    //defer wg.Done()
    res ：= &result{}
    if err != nil {
        res.err = err
    } else {
        res.planAResult = "A result" 
    }
    resChan <- res
}

func planB(err error, resChan chan *result, ctx context.Context, wg *sync.WaitGroup) {
    //defer wg.Done()
    resChanLocal := make(chan *result)
    getPlanB := func() {
        time.Sleep(5 * time.Second)
        res := &result{}
        if err != nil {
            res.err = err
        } else {
            res.planBResult = "B result"
        }
        resChanLocal <- res
    }
    go getPlanB()
    
    select {
    case <-ctx.Done():
        log.Println("B was canceled")
        //模型四与模型三最大的不同，就在于这里。无论是自己运算出结果，还是被cancel，都要向结果通道传值，保证行为一致。这样便于上层调用函数，处理本子任务的善后工作。
        resChan <- &result{}
        go func() {
            <-resChanLocal
            log.Println("B ended by cancel")
        }()
    case res := <-resChanLocal:
        log.Println("B ended by normal")
        resChan <- res
    }
}

func genTask() (interface{}, error) {
    //这里即使子任务的结果是同一类型，也强烈建议创建多个结果通道。有几种方案，就创建几个通道。方便后面进行逻辑判断。
    resChan1 := make(chan *result)
    resChan2 := make(chan *result)
    ctx, cancel := context.WithCancel(context.Background())
    var wg sync.WaitGroup
    
    //wg.Add(2)
    // 1. a right, b right
    go planA(nil, resChan1, &wg)
    go planB(nil, resChan2, ctx, &wg)
    
    // 2. a right, b wrong
    //go planA(nil, resChan1, &wg)
    //go planB(errors.New("B is set wrong"), bResChan, ctx, &wg)
    
    // 3. a wrong, b right
    //go planA(errors.New("A is set wrong"), resChan1, &wg)
    //go planB(nil, bResChan, ctx, &wg)
    
    // 4. a wrong, b wrong
    //go planA(errors.New("A is set wrong"), resChan1, &wg)
    //go planB(errors.New("B is set wrong"), bResChan, ctx, &wg)
    
    aRes := <-resChan1
    if aRes.err == nil {
        log.Println("we get planA result, finish the task")
        cancel()
        //方案A有结果且不报错，需要对方案B做善后工作。固定地从通道2中取一次结果。
        //cancel时，方案B既有可能select cancel，也有可能select发送结果。所以它们的行为要一致，方便我这里统一处理善后。
        go func() {
            <-resChan2  
        }()
        
        //只有在需要关闭通道时，才需要wg变量。它的作用就是通知调用者，不再使用通道了。但其实，通道不关闭也是可以的。
        //wg.Wait()
        //close(resChan1)
        //close(resChan2)
        return aRes, nil
    }
    
    log.Println("planA wrong. wait for planB")
    bRes := <-resChan2
    if bRes.err == nil {
        //wg.Wait()
        //close(resChan1)
        //close(resChan2)
        return bRes, nil
    }
    return nil, errors.New(aRes.err.Error() + " " + bRes.err.Error())
}

func main() {
    _, err := genTask()
    if err != nil {
        log.Println(err)
    }
    log.Println("genTask end")
    time.Sleep(20 * time.Second)
    log.Println("all tasks end.")
}
```



## **接口**

这一部分，自己的感悟比较少，很多内容参考了[网上的信息](http://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-interface)。

- 接口的广义概念

  在计算机科学中，接口是计算机系统中多个组件共享的边界，不同的组件能够在边界上交换信息。接口的本质就是引入一个新的中间层，调用方可以通过接口与具体实现分离，解除上下游的耦合，上层的模块不再需要依赖下层的具体模块，只需要依赖一个约定好的接口。

  举例1

  面向接口的编程方式有着非常强大的生命力，无论是在框架还是在操作系统中我们都能够找到接口的身影。可移植操作系统接口（Portable Operating System Interface，POSIX）就是一个典型的例子，它定义了应用程序接口和命令行等标准，为计算机软件带来了可移植性——只要操作系统实现了POSIX，计算机软件就可以直接在不同操作上运行。

  举例2

  除了解耦有依赖关系的上下游，接口还能够帮助我们隐藏底层实现，减少关注点。人能够同时处理的信息非常有限，定义良好的接口能够隔离底层的实现，让我们将重点放在当前的代码片段中。SQL就是接口的一个例子，当我们使用SQL语句查询数据时，其实不需要关心底层数据库的具体实现，我们只在乎SQL返回的结果是否符合预期。

- Go语言中的接口

  Go语言中的接口就是一组方法的签名，它是一种类型，一种只关心是否实现指定方法集的类型。

  - 当结构体实现接口时：

    初始化变量为结构体，可以调用接口内的方法；

    <u>*初始化变量为结构体指针时，可以调用接口内的方法。*</u>

  - 当结构体指针实现接口时：

    *<u>初始化变量为结构体，**不能**调用接口内的方法；</u>*

    初始化变量为结构体指针时，可以调用接口内的方法。

  对于上面这个差异的解释：Go语言在传递参数时都是传值的。无论初始化变量是结构体，还是结构体指针，调用方法时都会发生值拷贝。

  - 对于结构体实现接口，初始化变量为结构体指针并调用方法这种情况，会拷贝一个新的指针，这个指针（方法内）与原来的指针（调用的实参）指向一个相同并且唯一的结构体，所以编译器可以隐式的对变量解引用（dereference）获取指针指向的结构体；

  - 对于结构体指针实现接口，初始化变量为结构体并调用方法这种情况，会拷贝一个新的结构体，方法的形参是指针类型，编译器不会无中生有创建一个指针；即使编译器可以根据结构体创建新指针，这个指针指向的也不是最初调用该方法的结构体。

  当我们使用指针实现接口时，只有指针类型的变量才会实现该接口；当我们使用结构体实现接口时，指针类型和结构体类型都会实现该接口。当然这并不意味着我们应该一律使用结构体实现接口，这个问题在实际工程中也没那么重要，这里只是解释现象。

- 定义

  ```go
  //一个常见的Go语言接口是这样定义的
  type error interface {
      Error() string
  }
  ```

- Go语言的接口类型不是任意类型

  Go语言只会在传递参数，返回参数以及变量赋值时，才会对某个类型是否实现了接口进行检查。

  ```go
  package main
  type TestStruct struct{}
  func NilOrNot(v interface{}) bool {
      return v == nil
  }
  func main() {
      var s *TestStruct
      fmt.Println(s == nil)  // return true
      fmt.Println(NilOrNot(s))  // return false
  }
  ```

  调用NilOrNot函数时发生了隐式的类型转换——除了向方法传入参数之外，变量的赋值也会触发隐式类型转换。在类型转换时，*TestStruct类型会转换成interface{}类型，转换后的变量不仅包含转换前的变量的值，还包含变量的类型信息 *TestStruct，所以转换后的变量与nil不相等。

- **类型断言**

  1. 直接对确定类型发起推断

     - 完整形式，等号左边有两个变量。第一个变量推断成功时，是变量的副本；第二个为bool类型。

       ```go
       var a interface{} = 10
       tmp, ok := a.(int)
       ```

     - 简写形式，等号左边有一个变量。第一个变量推断成功时，是变量的副本。

       注意：十分有把握再采用这种形式，如果推断错误，会引发panic。

       ```go
       var a interface{} = 10
       tmp := a.(int)
       ```

  2. 对type发起推断，一般结合switch进行判断

     ```go
     var er interface{}
     switch tmp := er.(type) {
     case string:
         fmt.Println(tmp)
     case int:
         fmt.Println(tmp)
     }
     ```



## **反射**

这一部分，自己的感悟比较少，很多内容参考了[网上的信息](http://draveness.me/golang/docs/part2-foundation/ch04-basic/golang-reflect)。

- 运行时反射是程序在运行期间检查其自身结构的一种方式。Go语言的反射涉及三大法则：

  1. 从`interface{}`变量可以反射出反射对象；

     > 如果说Go语言的类型和反射类型处于两个不同的世界，那么reflect.TypeOf和reflect.ValueOf就是连接这两个世界的桥梁。
     >
     > 它们的入参类型都是interface{}，所以在函数执行的过程中发生了类型转换。所以，reflect包是通过interface{}类型与外界交互的。

  2. 从反射对象可以获取`interface{}`变量；

     > 既然能够将interface{}类型的变量转换成反射对象，那么一定需要其他方法将反射对象还原成interface{}类型的变量。reflect包中的reflect.Value.Interface()方法就能完成这项工作。不过这个方法只能获取interface{}类型的变量，如果想要将其还原成最原始的状态，还需要进行显示类型推断。

     ![go-basic基本类型与反射类型的转换](/Users/nieguanglin/pics/go/go-basic基本类型与反射类型的转换.png)

     当然不是所有的基本类型变量都需要隐式类型转换这一过程。如果变量本身就是interface{}类型，那么它就不需要类型转换。因为类型转换这一过程一般都是隐式的，所以不用太关心它。

     只有在我们需要将反射对象转换回基本类型时，才需要显示的类型推断。

  3. 要修改反射对象，其值必须是可设置的。

     > 这条法则与值是否可以被更改有关。
     >
     > Go语言的函数调用传参都是值拷贝传递，所以我们得到的反射对象跟最开始的变量没有任何关系，除非它是指针类型。

     ```go
     func main() {
         i := 1
         v := reflect.ValueOf(&i)  //如果要修改，就传指针
         v.Elem().SetInt(10)  //reflect.Value.Elem方法获取指针指向的变量
         fmt.Println(i)
     }
     ```

  所谓“反射对象”，就是reflect包提供的两个函数的返回值。reflect.TypeOf，reflect.ValueOf可以将一个普通的变量转换成反射包中提供的Type（interface类型）和Value（struct类型）。

- 使用反射来调用方法非常复杂，原本只需要一行代码就能完成的工作，现在需要十几行代码才能完成，但这也是在静态语言中使用动态特性需要付出的代价。

- 我的实际项目中，使用到的反射。

  - 利用反射获取结构体的元数据信息

    ```go
    //利用反射获取结构体的元数据信息
    func main() {
        structAobj := structA{
            fieldA: "A",
            fieldB: 1,
        }
        tag2Type := JsonTag2Type(structAobj)
        fmt.Println(tag2Type)
    }
    
    type structA struct {
        FieldA string `json:"fieldA"`
        FieldB int `json:"fieldB"`
    }
    
    func JsonTag2Type(structObj interface{}) map[string]string {
        tag2Type := make(map[string]string, 8)
        t := reflect.TypeOf(structObj)
        //Kind方法 returns the specific kind of this type.
        if t.Kind() == reflect.Ptr {
            //Go语言中对指针获取反射对象时，可以通过reflect.Type.Elem()方法获取这个指针指向的元素类型。这个获取过程被称为取元素，等效于对指针类型变量做了一个*操作。
            t = t.Elem()
        }
        //reflect.Type.Numfield returns a struct type's field count. It panics if the type's Kind is not struct.
        fieldNum := t.NumField()
        for i := 0; i < fieldNum; i++ {
            //reflect.Type.Field(i int)StructField returns a struct type's i'th field. It panics if the type's Kind is not struct. It panics if i is not in the range[0, Numfield()).
            field := t.Field(i)
            //Get returns the value associated with key in the tag string. If there is no such key in the tag, Get returns the empty string.
            //Tag是StructField的一个字段，StructTag类型。Type是它的一个字段，reflect.Type类型。
            tag2Type[field.Tag.Get("json")] = strings.ToUpper(field.Type.String())
        }
        return tag2Type
    }
    ```

  - 利用反射进行函数调用

    ```go
    //利用反射进行函数调用
    func main() {
        job := NewJob(map[string]interface{}{
            "funcA": funcA,
            "funcB": funcB,
        })
        job.Run("funcA", 1)
        job.Run("funcB", 1, "B")
        job.Wait()
        if job.Err != nil {
            fmt.Println(job.Err.Error())
        }
        for _, task := range job.Tasks {
            for _, res := range task.Result {
                switch r := res.(type) {
                case string:
                    fmt.Println(r)
                case nil:
                    fmt.Println("nil")
                case int:
                    fmt.Println(r)
                //如果Wait方法中，对结果遍历时，不执行res.Interface().(error)，那它还是interface{}，不是error
                case error:
                    fmt.Println(r.Error())
                }
            }
        }
    }
    
    func funcA(a int) (b string, err error) {
        returnt fmt.Sprintf("%d", a), nil
    }
    
    //todo: 如果目标函数带有可边长参数，那么这套方法就不再使用。
    //因为reflect.Type.NumIn固定返回3，但是目标函数却可以接收参数个数>=2的参数。
    //func funcB(a int, b string, tmp ...interface{}) (c int, err error) {
    func funcB(a int, b string) (c int, err error) {
        return a, fmt.Errorf("%s", b)
    }
    
    type Job struct {
        WorkFlow map[string]interface{}
        Tasks []*Task
        Err error
    }
    
    type Task struct {
        FuncName string
        Args []interface{}
        Result []interface{}
    }
    
    func NewJob(funcMap map[string]interface{}) *Job {
        return &Job{WorkFlow: funcMap}
    }
    
    func (j *Job) Run(funcName string, args ...interface{}) {
        j.Tasks = append(j.Tasks, &Task{FuncName: funcName, Args: args})
        return
    }
    
    func (j *Job) Wait() *Job {
        for _, task := range J.Tasks {
            result, err := j.callMapFunc(task.FuncName, task.Args)
            if err != nil {
                j.Err = err
                return j
            }
            for _, res := range result {
                //reflect.Type.String()方法，返回它持有的类型。
                if res.Type().String() == "error" && !res.IsNil() {
                    //不执行res.Interface().(error)，那它还是interface{}，不是error
                    task.Result = append(task.Result, res.Interface().(error))
                } else if res.Type().String() == "error" && res.IsNil() {
                    task.Result = append(task.Result, nil)
                } else {
                    //reflect.Value.Interface()方法，是reflect.ValueOf的逆过程。
                    task.Result = append(task.Result, res.Interface())
                }
            }
        }
        return j
    }
    
    func (j *Job) callMapFunc(funcName string, args []interface{}) ([]reflect.Value, error) {
        f := reflect.ValueOf(j.WorkFlow[funcName])
        //reflect.ValueOf(变量).Type() 返回变量的反射Type对象。等价于reflect.TypeOf(变量)
        //reflect.Type.NumIn() returns a function type's input parameter count.It panics if the type's Kind is not Func.可边长参数，它按1计算。
        if len(args) != f.Type().NumIn() {
            return nil, errors.New("para number is wrong")
        }
        in := make([]reflect.Value, len(args))
        for k, arg := range args {
            in[k] = reflect.ValueOf(arg)
        }
        //reflect.Value.Call([]reflect.Value)[]reflect.Value
        //通过反射进行函数调用
        return f.Call(in), nil
    }
    ```

    


## **代码心得**

- 如果函数会被其他接口多次调用，也就是说他可能是一个小粒度模块，那么它就应该尽量减少定制化的设计，成为一个基础模块。

- 如果函数很确定只被本接口多次调用，那就应该尽量设计为本接口的临时函数。

  - 使用临时函数做并发编程时，最好不要再用匿名函数包裹，否则不方便调试。
  - 使用临时函数时，明确闭包和传参的区别，可以让函数的形参变少。
  - 使用闭包减少传参时，需要考虑可能发生的冲突，比如多个goroutine并发修改err。

- 如果结构体不具有通用性，那也应该设计为本接口的临时类型。

- 关于chan

  - value, ok := <-chan 如果通道关闭，则ok为false。这一点也许会用得上。
  - 通过for range遍历chan，如果通道关闭，则跳出循环。这一点要考虑时序，有可能出现异常。目前来看，对通道进行遍历并不好用，多数都是固定次数地从通道中获取值。

- 并发编程，目前来看就是在抽象子任务，当然也可以在子任务中再把耗时任务进行抽象，进行goroutine的嵌套。通道的类型不应该成为限制，重新构造chanItem{result, error}即可。

- json与struct/[]struct可以互相转化

  json与map/[]map也可以互相转化！只是这种用法，只能直接使用“第一层”，而使用内层嵌套的数据不太方便

- []struct 与 []\*struct

  对[]struct进行遍历，目标是[]\*struct。非常不推荐这种用法。

  因为for i, item := range []struct{}中，item是一个固定的容器，它的地址是固定的。

  所以，初始和目标的类型最好一致，如果是值，就都是值；如果是指针，就都是指针。

- break continue goto都可以指定标签，结合return，可以使flow control更加灵活，简洁。

- 直接指定合适的容量，比后期动态扩容更有效率

  - make([]type, len, cap)
  - make(map[key]value, cap)

- swagger

  swag init -g pkg/route/route.go //自动根据controller函数的注释，以及route信息，生成swag文件。

- Go mod

  - go mod init xxx/projectName
  
  - go mod vendor //go mod加入vendor，试成功了，这一招真牛！
  
  - go build -mod=vendor -o projectName cmd/main.go //编译生成二进制文件
  
  - go run mod=vendor cmd/main.go //运行
  
  - go mod edit replace=github.com/chartmuseum/storage@v0.8.0=github.com/choujimmy/storage@xxxx //go.mod文件部分依赖项通过命令行替换
  
  - 还有一种解决go依赖的命令行。这种情况下，每个branch下没有go.mod文件，vendor目录为空但它@某个commitID。
  
    项目根目录下执行：git submodule update --init；cd vendor；git submodule update --init
  
- 几个常用，但是由于不规范会导致严重后果的做法

  - 类型推断，不加ok。
  - 切片取值，不判断索引，导致索引越界。
  - map不初始化，直接赋值键值对。

- 项目作为服务端
  - 接收文件类型的请求参数

    ```go
    c.PostForm("fieldName")
    ```

  - 接口返回文件

    ```go
    file, err := yaml.JSONToYAML([]byte(str1))
    c.Header("Content-Type", "application/octet-stream")
    c.Header("Content-Disposition", fmt.Sprintf("attachment; filename=%s_%s.yaml", fileName, version))
    c.Header("Content-Transfer-Encoding", "binary")
    c.Writer.Write(file)
    ```

- 项目作为客户端

  - 请求服务端时，文件作为请求体

    ```go
    bodyBuf := &bytes.Buffer{}
    bodyWriter := multipart.NewWriter(bodyBuf)
    //请求体是一个multipart，不仅仅有文件，还可以有别的键值对。所以fieldName是文件字段的名字，fileName是文件的名字
    fileWriter, err := bodyWriter.CreateFormFile("fieldName", "fileName")
    if err != nil {
      ...
    }
    
    //源数据
    fh, err := os.Open(filePath)
    if err != nil {
      ...
    }
    defer func() { _ = fh.Close() }()
    //源数据写入fileWriter
    if _, err = io.Copy(fileWriter, fh); err != nil {
      ...
    }
    
    //multipart再写入新的键值对
    if err = bodyWriter.WriteField("type", "application/gzip"); err != nil {
      ...
    }
    //必须close，不能defer
    _ = bodyWriter.Close()
    
    //获取请求头，请求头中有一串随机字符串，用来分隔multipart的不同部分。请求头必须获取。
    contentType := bodyWriter.FormDataContentType()
    
    //bodyBuf作为请求体
    //Content-Type: contentType作为请求头
    //接下来就是一个普通的post请求
    ```

  - 接收服务端返回的文件

    接口的响应体（[]byte）就是文件内容，没什么特殊的。

- docker run

  - 不开启network namespace

    这种情况下，容器的端口直接由配置文件指定，它不能和宿主机上其他端口冲突。

    ```bash
    $ docker run --net host --name appName -d -v 宿主机目录:容器rootfs目录 imageName:tag
    ```

  - 开启network namespace

    这种情况下，命令行指定的容器port，必须与容器实际listen的端口一致，否则访问不到。

    ```bash
    $ docker run            --name appName -d -v 宿主机目录:容器rootfs目录 -p 宿主机port:容器port imageName:tag
    ```

- 



## **项目优化**

- 批量导入接口需要解析xlsx文件，然后向etcd中多次写入数据

  对xlsx解析时，使用goroutine创建子任务，在子任务中，调用其他方法进行写入操作，进行并发写入。

  但是仍然会遇到瓶颈，本次批量导入的请求，压力全都落在同一个pod上，没有充分发挥k8s负载均衡的能力。所以，修改子任务：仍然是并发解析，但是写入时，再发起一次完整的http请求。这个http请求会被ingress负载均衡到不同的pod上，然后多个pod一起执行写入。

  

## **读书笔记**

- 虚拟化
  - qemu：完全虚拟化
  - qemu_kvm：硬件辅助虚拟化，qemu集成kvm模块，对cpu的操作进行优化
  - virtio_net，virtio_blk：类虚拟化，进行网络和存储访问，则通过类虚拟化或者直通Pass through的方式，通过加载特殊的驱动，加速访问网络和存储的资源
  - virsh：大多数还是通过virsh启动，virsh属于libvirt工具，libvirt是目前使用最为广泛的对KVM虚拟机进行管理的工具和API，可不止关闭KVM。
- IaaS，PaaS，SaaS
  - 虚拟化：闭源软件-VMware ---- 开源技术-Xen，KVM
  - 私有云：VMware
  - 公共云：AWS
  - 公有云：闭源-AWS ---- 开源-OpenStack（Iaas)  Docker+Kubernetes（PaaS）
  - 计算，网络，存储我们常称为基础设施Infranstracture，因而这个阶段的弹性称之为资源层面的弹性。管理资源的云平台，我们称为基础设施服务，就是我们常听到的**IaaS，Infranstracture as a Service**。
  - 人们在IaaS平台上又加了一层，用于管理资源以上的应用弹性的问题，这一层通常称为**PaaS（Platform as a Service）**。这一层往往比较难理解，其实大致分两部分，一部分我称之为你自己的应用自动安装，一部分我称为通用的应用不用安装。
  - 云计算与大数据。只有云计算，可以为大数据的运算提供资源层的灵活性。而云计算也会部署大数据放到它的PaaS平台上，作为一个非常非常重要的通用应用。
  - 大数据与人工智能。然而神经网络包含这么多的节点，每个节点包含非常多的参数，整个数据量实在是太大了，需要的计算量实在太大，但是没有关系啊，我们有大数据平台，可以汇聚多台机器的力量一起来计算，才能在有限的时间内得到想要的结果。
  - SaaS/人工智能与云计算。由于人工智能算法多是依赖大量的数据的，这些数据往往需要面向某个特定的领域（例如电商，邮箱）进行长期的积累，如果没有数据，就算有人工智能算法也白搭，所以人工智能程序很少像前面的IaaS和PaaS一样，将人工智能程序给某个客户安装一套让客户去用，因为给某个客户单独安装一套，客户没有相关的数据做训练，结果往往是很差的。但是云计算厂商往往是积累了大量数据的，于是就在云计算厂商里面安装一套，暴露一个服务接口，比如您想鉴别一个文本是不是涉及黄色和暴力，直接用这个在线服务就可以了。这种形式的服务，在云计算里面称为软件即服务，**SaaS（Software as a Service）**。
  - 终于云计算的三兄弟凑齐了，分别是IaaS，PaaS和SaaS，所以一般在一个云计算平台上，云，大数据，人工智能都能找得到。对一个大数据公司，积累了大量的数据，也会使用一些人工智能的算法提供一些服务。对于一个人工智能公司，也不可能没有大数据平台支撑。所以云计算，大数据，人工智能就这样整合起来，完成了相遇，相识，相知。





