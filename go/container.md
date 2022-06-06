[TOC]

# List

## 结构

<img src="https://github.com/NieGuanglin/docs/blob/main/pics/go/go-container.list的实现.png">

<img src="/Users/nieguanglin/docs/pics/go/go-container.list的实现.png" alt="go-container.list的实现.png" style="zoom:100%;" />

List在内部就是一个循环链表，它的root永远不会持有任何实际的元素值，而该元素的存在就是为了连接这个循环链表的首尾两端。

```go
type List struct {
	root Element 
	len  int     
}
type Element struct {
	next, prev *Element

	list *List

	Value interface{}
}
```

## 延迟初始化

Go标准库中很多结构体类型的程序实体都做到了开箱即用，链表List就是其中之一。var l list.List声明的链表l可以直接使用，关键就在于它的延迟初始化机制。

延迟初始化，就是把初始化操作延后，仅在实际需要的时候才进行。

- 优点：在于延后，它可以分散初始化操作带来的计算量和存储空间消耗。计算量和存储空间的压力可以被分散到实际使用它们的时候，实际使用的时间越分散，延迟初始化带来的优势就会越明显。
- 缺点：也在于延后，在使用对象的时候，如果每个方法都需要判断它是否已经被初始化，这也会是计算力的浪费。当方法被频繁调用的时候，这种浪费的影响就开始显现了。

## List平衡延迟初始化的优点和缺点

当移动链表中的Element时，如果我们自己生成一个Element并传给诸如func (l *List) MoveToFront(e *Element)等方法，这些方法不会对链表做出任何改动。因为Element的list字段私有，我们无法获取，或者说我们自己生成的Element不在链表中，谈不上移动元素。

- List的入口方法：PushFront，PushBack，PushBackList，PushFrontList总会先判断链表状态，必要时进行初始化。注意：只有这四个入口方法才判断，弥补了延迟初始化的缺点。

  为什么说这四个方法是入口？因为其他方法都需要Element，而它又不允许自己生成的Element插入链表。所以只能依赖这四个不需要Element的方法。

  ```go
  func (l *List) PushFront(v interface{}) *Element {
  	l.lazyInit()
  	return l.insertValue(v, &l.root)
  }
  ```

- 一些方法是无需对是否初始化做判断的，一旦发现链表的长度为0，直接返回nil。比如：Front，Back。

  ```go
  func (l *List) Front() *Element {
  	if l.len == 0 {
  		return nil
  	}
  	return l.root.next
  }
  ```

- 一些方法只要判断传入的元素中指向所属链表的指针，是否与当前链表的指针相等就可以了。比如：删除元素，移动元素，插入元素。

  ```go
  func (l *List) Remove(e *Element) interface{} {
  	if e.list == l {
  		l.remove(e)
  	}
  	return e.Value
  }
  ```

指针相等是链表已经初始化的充要条件。List和Element在结构上的特点，巧妙地平衡了延迟初始化的优缺点，既使链表可以开箱即用，又在性能上可以达到最优。

# Ring

## 结构

Ring的结构相对简单，就是一个首尾相连的双向链表。一个Ring类型的值只代表了其所属循环链表中的一个元素，而一个List类型的值代表了一个完整的链表。

## 巧妙的抽象

两个Ring的link操作，可以用下图表示。

<img src="https://github.com/NieGuanglin/docs/blob/main/pics/go/go-container.ring的link实现.png">

<img src="/Users/nieguanglin/docs/pics/go/go-container.ring的link实现.png" alt="go-container.ring的link实现.png" style="zoom:100%;" />

```go
func (r *Ring) Link(s *Ring) *Ring {
	n := r.Next()
	if s != nil {
		p := s.Prev()
		// Note: Cannot use multiple assignment because
		// evaluation order of LHS is not specified.
		r.next = s
		s.prev = r
		n.prev = p
		p.next = n
	}
	return n
}
```

而把一个Ring的UnLink操作，源码的作者把它抽象为：与Link是本质上相同的问题。这种巧妙的抽象，很大程度上减少了代码量。

<img src="https://github.com/NieGuanglin/docs/blob/main/pics/go/go-container.ring的unlink实现.png">

<img src="/Users/nieguanglin/docs/pics/go/go-container.ring的unlink实现.png" alt="go-container.ring的unlink实现.png" style="zoom:100%;" />

```go
func (r *Ring) Unlink(n int) *Ring {
	if n <= 0 {
		return nil
	}
	return r.Link(r.Move(n + 1))
}
```

