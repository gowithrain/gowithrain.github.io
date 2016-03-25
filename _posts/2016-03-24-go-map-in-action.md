---
layout: post
title: Go语言中的map实践
---
{{ page.title }}
---
注：本文是Go官方Blog的中文翻译版本，原文链接请[点击](http://blog.golang.org/go-maps-in-action)。转载本文请注册出处。

### 简介
---
hash表是计算机科学中最有用的语言之一。一些hash表的实现有着非常丰富的属性，但通常都会提供快速查找、增加和删除操作。Go语言提供了一个实现hash表的内置类型--map。

### 声明和初始化
---
一个Go的map类型看起来是这样的：

	map[KeyType]ValueType

*KeyType*是任意[可比较的](http://golang.org/ref/spec#Comparison_operators)数据类型。*ValueType*的数据类型则没有限制，甚至可以包含另一个map。

下面的*m*变量是一个key为string类型、值为int类型的map

	var m map[string]int

map类型和指针、切片（slice）一样，都是引用类型，因此上面的m的值是*nil*（注：引用类型的0值是nil）；它并没有指向一个初始化的map。一个值为*nil*的map，当对它读取时，行为和空的map一致；但对它写，会导致致命的错误。***不要使用一个没有初始化的map***。为了初始化一个map，用Go语言内置的make函数：

	m = make(map[string]int)

make函数分配和初始化一个map数据结构并且返回一个指向这个结构的指针。在这篇文章中，我们主要聚焦在如何***使用***map，而不关心它的实现细节。
### map的使用
---
Go提供了一些非常通用的语法来操作map。下面的等式将键值为*route*的值设置为*66*:


	m["route"] = 66

下面的等式获取键值为*route*的值，并将它存储在一个新的变量*i*上：

	i := m["route"]

如果请求的键值并不存在，我们将获取到值类型的零值。在我们的例子中，值类型是*int*，因此零值是*0*：

	j := m["root"]
	// j == 0

内置的函数*len*返回一个map中所有元素的个数：

	n := len(m)

内置的函数*delete*将移除map中的一个元素:

	delete(m, "route")

*delete*函数没有返回值，如果删除的键值不存在，则什么也不做。

一个二值等式检测一个键值是否存在：

	i, ok := m["route"]

在这个等式中，第一个值(*i*)赋值为键值*route*的值。如果这个键值并不存在，i的值是零值(0)。第二个值(*ok*)是一个*bool*类型，如果这个键值存在，则是*true*；否则，是*false*。
如果只是想知道一个键值是否存在，用*下划线*取代第一个值:

	_, ok := m["route"]

为了遍历一个map的所有元素，用*range*关键字:

	for key, value := range m {
		fmt.Println("Key:", key, "Value:", value)
	}

为了用一些数据初始化一个map，用map标签:

	commits := map[string]int{
		"rsc": 3711,
		"r":   2138,
		"gri": 1908,
		"adg": 912,
	}

同样的语法也可以用来初始化一个空的map，它和用*make*函数有着一样的效果

	m = map[string]int{}

### 深入研究零值
---
当一个键值不存在时，会返回值类型的零值，这样的设计在实际使用中很方便。
例如，一个boolean值类型的map可以当一个set使用。在这个例子中，遍历一个链表的节点并打印它们的值。通过使用一个键值为Node类型的指针的map，我们可以检测是否这个链表存在一个环。

	{% highlight go linenos %}
	type Node struct {
		Next *Node
		Value interface{}
	}
	var first *Node
	
	visited := make(map[*Node]bool)
	for n := first; n != nil; n = n.Next {
		if visited[n] {
			fmt.Println("cycle detected")
			break
		}
		visited[n] = true
		fmt.Println(n.Value)
	}
	{% endhighlight %}

当n已经访问过了，则visited[n]的值为*true*，否则如果n不存在，返回*false*。我们不需要用二值等式来检测n是否存在；零值默认为我们做了这件事。

另一个场景是值类型为slice的map。向一个值为*nil*的切片添加元素，会分配一个新的切片，因此往一个slice的map添加值只需要一行代码。这样，对于一个值类型为slice的map，我们在向它添加元素时，就不需要检测键值是否存在。下一个例子中，切片*people*是由*Person*构成的。每一个*Person*都有一个*Name*和一个*Likes*的切片。这个例子创建了一个map，这个map将*like*(*注：这个人喜欢的东西*)和喜欢它的*people*关联起来。

	{% highlight go linenos %}
	type Person struct {
		Name string
		Likes []string
	}
	var people []*Person
	
	likes := make(map[string][]*Person)
	for _, p := range people {
		for _, l := range p.Likes {
			likes[l] = append(likes[l], p)
		}
	}
	{% endhighlight %}

打印所有喜欢*cheese*的人的列表：

	for _, p := range likes["cheese"] {
		fmt.Println(p.Name, "likes cheese.")
	}

打印喜欢*bacon*的人的数目:

	fmt.Println(len(likes["bacon"]), "people like bacon.")

注意，由于*range*和*len*将*nil*值的切片看作长度为0的切片，所以在上面的两个例子中，即使没有人喜欢*cheese*和*bacon*，也可以正常的工作。
### 键类型
---
正如之前提到的，map的键可以是任意*可比较的*类型。简单地说，在[Go的语言标准](http://golang.org/ref/spec#Comparison_operators)中，精确地定义了*boolean*, *numeric*, *string*, *pointer*, *channel*和*interface*，以及仅包含上述类型的*struct*、*array*为*可比较*类型。slice、map和function不是*可比较的*，这些类型不能用==比较，当然也不能用作map的键。

显然，*string*、*int*和其它基本类型都可以用作map的键。但*struct*能用作map的键有点出乎意料。如果一个map的键有多个维度，那么可以定义成一个*struct*。例如，map的map可以用来统计每个国家的网页点击量。

	hits := make(map[string]map[string]int)

这是一个*string*到(*string到int的map*)的map。每个外部map的键表示一个网站的路径，每个内部map的键表示2个字符的国家代码。下面的表达式获取澳大利亚一个网页的点击量:

	n := hits["/doc/"]["au"]

不幸的是，这样的实现使得增加数据时非常的麻烦。对于一个指定的键，你必须检测内部的map是否存在，如果不存在，你需要新建一个：

	{% highlight go linenos %}
	func add(m map[string]map[string]int, path, country string) {
		mm, ok := m[path]
		if !ok {
			mm = make(map[string]int)
			m[path] = mm
		}
		mm[country]++
	}
	add(hits, "/doc/", "au")
	{% endhighlight %}

另一方面，如果把键变成*struct*，这些复杂性都消失了。

	{% highlight go linenos %}
	type Key struct {
		Path, Country string
	}
	hist := make(map[Key]int)
	{% endhighlight %}

当一个越南人访问主页，增加（可能需要新建）合适的计数只需要一行代码:

	hits[Key{"/", "vn"]++

同样，查看有多少瑞士人读了这个说明书也非常简单

	n := hits[Key{"/ref/spec", "ch"}]

### 并发
---
[并发访问map并不安全](http://golang.org/doc/faq#atomic_maps):当你并发读或写一个map时，结果是不可预测的。如果需要在多个并发的*goroutine*中对一个map进行读写，访问必须要加入锁机制。一个普遍的保护map的做法是[sync.RWMutex](http://golang.org/pkg/sync/#RWMutex)。

这个语句声明了一个*counter*变量，它是一个匿名的*struct*类型，包含一个map和一个sync.RWMutex。

	{% highlight go linenos %}
	var counter = struct{
		sync.RWMutex,
		m map[string]int
	}{m: make(map[string]int)}
	{% endhighlight %}

当读这个*counter*时，用读锁:

	{% highlight go linenos %}
	counter.RLock()
	n := counter.m["some_key"]
	counter.RUnlock()
	fmt.Println("some_key:", n)
	{% endhighlight %}

当写这个*counter*时，用写锁:

	{% highlight go linenos %}
	counter.Lock()
	counter.m["some_key"]++
	counter.Unlock()
	{% endhighlight %}

### 遍历顺序
---	
当通过*range*遍历一个map时，遍历顺序是不确定的，同时也不保证两次的遍历顺序是一致的。自从Go 1后，map的遍历顺序都是随机的，而之前是一个固定的顺序。如果你非常需要一个固定的遍历顺序，那你只能额外操作一个数据结构，这个数据结构存储确定顺序的数据。下面的例子用一个额外的slice,存储排好序的键，从而打印map[int]string.

	{% highlight go linenos %}
	import "sort"
	
	var m map[int]string
	var keys []int
	for k :=range m {
		keys = append(keys, k)
	}
	sort.Ints(keys)
	for _, k := range keys {
		fmt.Println("Key:", k, "Value:", m[k])
	}
	{% endhighlight %}
