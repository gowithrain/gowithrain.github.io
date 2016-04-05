---
layout: post
tags: 翻译 Go 切片 slice
title: 数组、切片和字符串:append的机制
---
注：本文是Go官方Blog的中文翻译版本，原文链接请
[点击](https://blog.golang.org/slices)。转载本文请注明出处。

### 简介

对于计算机编程语言来说，最通用的特性之一，就是数组的概念。数组看起来是简单的，但当我们想把它加入到一个新的语言中，必须弄明白一些问题，比如：

* 固定长度还是可变长度？
* 长度是否是类型的一部分？
* 多维数组如何实现？
* 空数组的含义是什么？

这些问题的答案，决定了数组是这个语言的属性，还是一个核心的组成部分。

在Go开发的早期，我们花费了一年的时间讨论这些问题的答案，然后才让这个设计令我们满意。这个设计的关键步骤就是引入切片(slice)。切片是一个构建在固定长度的数组之上的，拥有灵活、可扩展特性的数据结构。但是，直到今天，一些新接触Go的程序员，依然对切片的工作方式感到困惑。也许，来自其它语言的经验，会让他们的想法先入为主。

在这篇文章中，我们会尝试着消除这些困惑。我们会一步一步地解释内置函数append如何工作，以及它为什么是按照这种方式工作。

### 数组

数组是Go语言大厦的一块重要的地基，但像任何地基一样，它们常常是隐藏在可见部分下面。在我们讨论更加有趣、强大并有着精彩设计的切片之前，我们必须仔细的讨论一下数组。

数组在Go语言的使用并不多见，那是因为数组的长度是它类型的一部分。这种特性限制了它的表现力。

下面的声明

	var buffer [256]byte

定义了一个有256个字节的变量*buffer*，这个*buffer*的类型包含它的长度，[256]byte。一个有着512个字节的数组有着唯一的类型[512]byte。

关联这个数组的数据是一列数组元素。这段buffer在内存中看起来像是这样

	buffer: byte byte byte ... 256次 ... byte byte byte

这个变量正好有256个字节的数据。我们能访问这些元素用我们熟悉的索引语法: buffer[0], buffer[1],一直到buffer[255]。（下标0到255正好覆盖256个元素）。*buffer*下标访问如果越界，将会引起程序崩溃。

内置的函数len能返回一个数组或切片的元素个数。对于数组，len的返回值是显而易见的。在我们的例子中，len(buffer)返回固定的长度256.

数组也有它的使用场景。例如，它们是转换矩阵一个很好的描述。但是，在Go语言中，大多数情况都是用切片存储数据。

### 切片：切片头

想要用好切片，必须要真正地明白它们是什么和它们能做什么。

切片是一个描述了数组中一段连续元素的数据结构。***切片不是数组***。一个切片描述了数组的一个片段。

现在看我们之前段落描述的*buffer*数组变量，我们可以通过切片这个数组来生成一个描述元素100到150（更准确地说，是100到149，包含）的切片：

	var slice []byte = buffer[100:150]

在上面的小片段中，为了更清晰，我们使用了最完整的变量定义。变量*slice*有着类型[]byte，读作“字节类型的切片”。它通过切片*buffer*数组的100(包含)到150(不包含)的元素进行初始化。更常用的语法是省去类型：

	var slice = buffer[100:150]

在函数里，我们可以使用更简短的声明：

	slice := buffer[100:150]

这个*slice*变量真正是什么？直到现在，我们可以认为切片是一个用着两个元素的数据结构：长度和一个指向数组元素的指针，尽管，这种认识并不完全。你可以认为它看起来像这个样子：
{% highlight go linenos %}
type sliceHeader struct {
	Length        int
	ZerothElement *byte
}

slice := sliceHeader{
	Length:        50,
	ZerothElement: &buffer[100],
}
{% endhighlight %}

当然，上面的代码只是一个示例。尽管，上面提到的sliceHeader数据结构对程序员并不可见，同时指针的类型取决于指向的元素的类型，但这至少对这个结构有了一个直观的感觉。

迄今，我们只在数组上使用了切片，事实上，我们也可以在切片上使用切片，像下面这样：

	slice2 := slice[5:10]

和之前的一样，这个操作创建了一个新的切片。在这个例子中，这个切片拥有*slice*的第5到第9(包含)的元素，也就是初始数组的105到109的元素。对于*slice2*变量, 相关的sliceHeader结构看起来像这样：

	slice2 := sliceHeader{
		Length:        5,
		ZerothElement: &buffer[105],
	}

注意，这个头信息依然指向相同的存储在*buffer*变量的数组。

我们也可以重新切片，也就是说对一个切片进行切片，然后将结果存回初始的切片结构。

	slice = slice[1:len(slice)-1]

在这之后，*slice*变量的sliceHeader结构看起来和*slice2*变量的一样了。重新切片使用得比较广泛，例如截短一个切片。下面的例子将切片的第一个和最后一个元素移除：

	slice = slice[1:len(slice)-1]

你常常听到有经验的Go程序员谈论切片的头，因为这个信息确实真正地存储在一个切片变量里。例如，当你调用一个需要切片参数的函数时，比如[bytes.IndexRune](http://golang.org/pkg/bytes/#IndexRune)，这个头部将会被传递给函数。在下面的调用中：

	slashPos := bytes.IndexRune(slice, '/')
	
被传递给IndexRune函数的*slice*变量，其实是一个“切片的头"。

在这个头部中，还存储着另一个数据元素，后面我们会讨论它。我们先看看当我们在程序中使用切片时，有了这个头部意味着什么。
### 传切片给函数
尽管切片包含着一个指针，但其实它本身是一个值类型，明白这一点非常重要。本质上，它是一个拥有一个指针和长度的结构体。它*不是*一个指向结构体的指针。

这很重要。

在之前的例子中，当我们调用IndexRune时，将切片头部的*拷贝*传递给函数。这样的行为有着重要的影响。

考虑下面的函数：

{% highlight go linenos %}
func AddOneToEachElement(slice []byte) {
	for i := range slice {
		slice[i]++
	}
}
{% endhighlight %}

正如函数名暗示的，它遍历切片的每一个元素，并将它们加一。

使用它：

{% highlight go linenos %}
func main() {
	slice := buffer[10:20]
	for i := 0; i < len(slice); i++ {
		slice[i] = byte(i)
	}
	fmt.Println("before", slice)
	AddOneToEachElement(slice)
	fmt.Println("after", slice)
}
{% endhighlight %}
	
尽管切片头按值传给了函数，但头包含了指向数组元素的指针，所以初始切片的头和传给函数的头的拷贝描述的是同一个数组。因此，当函数返回时，通过初始的*slice*变量，也能看到改变的元素。

传递给函数的参数真的只是一个拷贝，就像下面这样：

{% highlight go linenos %}
func SubtractOneFromLength(slice []byte) []byte {
	slice = slice[0 : len(slice)-1]
	return slice
}
func main() {
	fmt.Println("Before: len(slice) =", len(slice))
	newSlice := SubtractOneFromLength(slice)
	fmt.Println("After: len(slice) =", len(slice))
	fmt.Println("After: len(newSlice) =", len(newSlice))
}
{% endhighlight %}	

我们可以看到，一个切片参数的内容可以被一个函数改变，但它的头不能。存储在*slice*变量的长度不会被调用的函数改变，因为这个函数接收的是切片头的拷贝。因此，如果你想要写一个函数改变头部信息，我们必须将它做为一个结果参数返回，就像我们上面的例子描述的。*slice*变量是不会被改变的，但返回的值有一个新的长度，它存储在*newSlice*变量中。

### 切片指针：方法接收器

另一个让函数改变切片头信息的方式是传递指针给函数。下面是之前例子的变种，它可以实现我们的目的：

{% highlight go linenos %}
func PtrSubtractOneFromLength(slicePtr *[]byte) {
	slice := *slicePtr
	*slicePtr = slice[0 : len(slice-1)]
}
func main() {
	fmt.Println("Before: len(slice) =", len(slice))
	PtrSubtractOneFromLength(&slice)
	fmt.Println("After: len(slice) =", len(slice))
}
{% endhighlight %}	
	
在这个例子中，处理方式有些笨拙，特别是还需要一个临时变量辅助。但这是一个常见的使用切片指针的方式。针对一个改变切片的方法，使用指针接受器更加符合Go的语言习惯。

我们想给切片定义一个方法，用来截断切片元素最后的反斜杠(/)。我们这样实现：

{% highlight go linenos %}
type path []byte

func (p *path) TruncateAtFinalSlash() {
	i := bytes.LastIndex(*p, []byte("/"))
	if i >= 0 {
		*p = (*p)[0:i]
	}
}
{% endhighlight %}	

运行这个例子，你会看到方法正确地更新了切片。

另一方面，如果你想要实现一个方法，将*path*中的所有英文字母变成大写。这个方法可以作用在一个值上，因为值接收器指向的数组元素是一样的。

{% highlight go linenos %}
type path []byte
	
func (p path) ToUpper() {
	for i, b := range p {
		if 'a' <= b && b <= 'z' {
			p[i] = b + 'A' - 'a'
		}
	}
}
	
func main() {
	pathName := path("/user/bin/tso")
	pathName.ToUpper()
	fmt.Printf("%s\n", pathName)
}
{% endhighlight %}	
	
在ToUpper方法中，我们使用了两个变量用来存储for range语句返回的下标和切片元素。这种方式可以避免在for循环体中写p[i]多次。

### 容量

下面的函数，将传递过来的int类型的切片追加一个元素：

{% highlight go linenos %}
func Extend(slice []int, element int) []int {
	n := len(slice)
	slice = slice[0 : n+1]
	slice[n] = element
	return slice
}
{% endhighlight %}

(为什么函数需要返回改变的切片？)现在运行它：

{% highlight go linenos %}
func main() {
	var iBuffer [10]int
	slice := iBuffer[0:0]
	for i := 0; i < 20; i++ {
		slice = Extend(slice, i)
		fmt.Println(slice)
	}
}
{% endhighlight %}

你能看到这个切片是怎么增长的，直到...程序挂掉。

是时候讨论切片头部中的第三个元素了：切片的容量。除了数组的指针和长度，切片的头也存储了它的容量。

	type sliceHeader struct {
		Length        int
		Capacity      int
		ZerothElement *byte
	}

这个**Capacity**字段记录了原始数组有多大的空间，也就是**Length**可以达到的最大的值。增加切片的元素，直到超过这个值，将会触发一个panic。

在我们的例子中，切片被创建：

	slice := iBuffer[0:0]

它的头部看起来是这样的：

	slice := sliceHeader{
		Length:        0,
		Cacacity:      10,
		ZerothElement: &iBuffer[0],
	}
	
*Capacity*的值等于原始数组的长度减去切片第一个元素在原始数组中的下标（本例子中是0）。如果你想要查询一个切片的容量，用内置的函数cap:

	if cap(slice) == len(slice) {
		fmt.Println("slice is full!")
	}
	
### Make
我们能否往切片增加元素直到超过它的容量？答案是不能。直白地说，容量是增长的上限。不过，你可以通过分配一个新的数组、将原数组的数据拷贝过来，然后改变新数组的切片来达到这个目的。

首先，从分配说起。我们可以用内置函数new来分配一个更大的数组，然后用一个切片来指向它。但其实用内置函数make做这件事更简单。它分配一个新的数组并创建一个切片头来描述它，这些操作一次完成。make函数需要三个参数：切片类型、它的初始长度和它的容量。下面的调用创建了一个长度为10，并且还有5个空闲空间的切片。通过运行这个例子，你能看得更清楚：

	slice := make([]int, 10, 15)
	fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
	
下面的代码段将int切片的容量加倍，但长度保持不变：

{% highlight go linenos %}
slice := make([]int, 10, 15)
fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
newSlice := make([]int, len(slice), 2*cap(slice))
for i := range slice {
	newSlice[i] = slice[i]
}
slice = newSlice
fmt.Printf("len: %d, cap: %d\n", len(slice), cap(slice))
{% endhighlight %}

执行这个程序后，在重新分配内存之前，*slice*将有更多的空闲空间来容纳增加的元素。

当创建一个切片时，常常长度和容量是相同的。对于这种情况，make有一个更简洁的用法。长度参数默认也是容量参数，这样我们可以省略容量参数，使得它们俩都是一样的值。如：

	gophers := make([]Gopher, 10)
	
*gophers*切片的长度和容量都是10.

### 拷贝

在上面给切片加倍容量的例子中，我们写了一个循环拷贝老的数据到新的切片。Go语言有一个内置的函数copy，它可以让这个过程变得更容易。它的参数是两个切片，数据从右边的参数拷贝到左边的参数。下面重写前面的例子：

	newSlice := make([]int, len(slice), 2*cap(slice))
	copy(newSlice, slice)

copy函数是聪明的，它能注意到两个参数的长度。换句话说，它拷贝的元素个数是两个切片最少的。同时，copy函数会返回一个int值，它表明有多少元素被拷贝了。但是，通常我们不需要处理这个返回值。

如果源切片和目的切片有重合，copy函数也可以正确地工作。这个特性意味着我们可以用它来移动一个单独切片内的元素。下面的例子展示了如何用copy函数来在切片中间插入一个元素。
{% highlight go linenos %}
// Insert 在切片指定的合法的下标下插入一个值,
// 切片必须有空间容纳新的元素
func Insert(slice []int, index, value int) []int {
	// 切片长度加1
   	slice = slice[0 : len(slice)+1]
   	// 通过copy来移动数组
   	copy(slice[index+1:], slice[index:])
   	// 存储新的值
   	slice[index] = value
   	// 返回结果
   	return slice
}
{% endhighlight %}
在这个函数中，我们有两点需要关注。第一，函数必须返回更新后的*slice*，因为切片的长度已经改变了。第二，它用了一个方便的简洁表达式。如下：

	slice[i:]

它表达的和下面的表达式是完全一样的：

	slice[i:len(slice)]

当然，我们也可以省去切片表达式的第一个元素（它默认是0），例如：

	slice[:]

它表示的是切片自身。这个表示式是最短的表示“一个描述数组所有元素的切片”的方式：

	array[:]

现在，让我们执行Insert函数:

{% highlight go linenos %}
slice := make([]int, 10, 20) // 注意容量>长度，说明有空间增加元素
for i := range slice {
	slice[i] = i
}
fmt.Println(slice)
slice = Insert(slice, 5, 99)
fmt.Println(slice)
{% endhighlight %}

### Append：一个例子
在前面有一段，我们写了一个Extend函数。但其实它有bug，因为如果切片的容量太小，这个函数就会崩溃。（我们的Insert例子也有相同的问题）现在，为了修复它我们需要添加一些适当的代码，因此让我们写一个更加健壮的Extend实现。

{% highlight go linenos %}
func Extend(slice []int, element int) []int {
	n := len(slice)
	if n == cap(slice) {
		// 切片满了，需要增加容量
		// 我们将容量加倍然后再加1，这样即使容量是0也可以正常地增长
		newSlice := make([]int, len(slice), 2*len(slice)+1)
		copy(newSlice, slice)
		slice = newSlice
	}
	slice = slice[0:n+1]
	slice[n] = element
	return slice
}
{% endhighlight %}

在这个例子中，当重新分配了一个描述完全不同的数组的切片时，返回这个切片是特别重要的。下面的代码展示了当切片满了时，什么将发生：

{% highlight go linenos %}
slice := make([]int, 0, 5)
for i := 0; i < 10; i++ {
	slice = Extend(slice, i)
	fmt.Printf("len=%d cap=%d slice=%v\n", len(slice), cap(slice), slice)
	fmt.Println("address of 0th element:", &slice[0])
}
{% endhighlight %}

注意，当初始长度为5的数组被填满时，将会重新分配内存。此时，容量和第0个元素的地址都会改变。

参考这个更健壮的Extend函数实现，我们可以写出一个更好的函数，它可以用多个元素扩展切片。为了做到这点，我们用Go的可变函数参数。

我们叫这个函数为Append。在第一个版本，我们简单地多次调用Extend函数。Append的签名如下：

	func Append(slice []int, items ...int) []int

Append需要一个切片参数和0到多个int参数。这些参数事实上是一个int类型的切片，这关系到Append的实现。如下：

{% highlight go linenos %}
// Append追加一些元素到切片
// 第一版本：循环调用Extend
func Append(slice []int, items ...int) []int {
	for _, item := range items {
		slice = Extend(slice, item)
	}
	return slice
}
{% endhighlight %}

注意，for range循环遍历*items*参数的所有元素，*itmes*参数隐含的类型是[]int。同时，也需要注意到，我们用下划线_舍弃循环中的索引，在这个例子中，我们并不需要它。

使用它：

	slice := []int{0, 1, 2, 3, 4}
	fmt.Println(slice)
	slice = Append(slice, 5, 6, 7, 8)
	fmt.Println(slice)
	
在这个例子中，另一个我们学到的新技术是用一个混合标记初始化切片。它包含切片的类型和在大括号中的元素：

	slice := []int{0, 1, 2, 3, 4}

Append函数是有趣的。它不仅能追加多个元素，也能追加一整个切片，只需要用...标记将切片转换成多个元素参数:

{% highlight go linenos %}
slice1 := []int{0, 1, 2, 3, 4}
slice2 := ]int{55, 66, 77}
fmt.Println(slice1)
slice1 = Append(slice1, slice2...) // ...是必须要有的！
fmt.Println(slice1)
{% endhighlight %}

当然，我们可以一次性分配够内存来优化Append的性能，看下面的代码：

{% highlight go linenos %}
// Append追回多个元素到一个切片
// 性能优化版本
func Append(slice[]int, elements ...int) []int {
	n := len(slice)
	total := len(slice) + len(elements)
	if total > cap(slice) {
		// 重新分配内存，增长1.5倍。
		newSize := total*3/2 + 1
		newSlice = make([]int, total, newSize)
		copy(newSlice, slice)
		slice = newSlice
	}
	slice = slice[:total]
	copy(slice[n:], elements)
	return slice
}
{% endhighlight %}

我们用了两次copy函数，先是将切片数据移到新分配的内存中，然后再将要追加的元素拷贝到原有数据的后面。

使用它，效果和之前的程序是一样的：

	slice1 := []int{0, 1, 2, 3, 4}
	slice2 := ]int{55, 66, 77}
	fmt.Println(slice1)
	slice1 = Append(slice1, slice2...) // ...是必须要有的！
	fmt.Println(slice1)

### Append: 内置函数
现在，我们已经知道了设计append内置函数的动机。它其实和上面例子中的Append函数功能一样、性能也一样，只不过它可以用在任意切片类型。

Go的一个弱点就是在运行时必须明确类型。也许有一天这种情况会改变，但至少现在，为了更容易地操作切片，Go提供了一个内置的通用append函数。它可以正确地应用在int切片上，也可以应用在其它任意切片类型上。

切记，由于append函数会改变切片头，所以你需要保存append返回的切片值。事实上，如果你没有保存append返回的结果，编译器会报错。

下面一些混合着print语句的代码，试着运行它并自己探索新的用法：

{% highlight go linenos %}
// 创建一对初始的切片
slice := []int{1, 2, 3}
slice2 := []int{55, 66, 77}
fmt.Println("start slice: ", slice)
fmt.Println("start slice2: ", slice2)

// 加一个元素到切片
slice = append(slice, 4)
fmt.Println("Add one item:", slice)

// 加一个切片到另一个
slice = append(slice, slice2...)
fmt.Println("Add on slice:", slice)

// 创建一个切片的拷贝
slice3 := append([]int(nil), slice...)
fmt.Println("Copy a slice:", slice3)

// 拷贝一个切片到它自身的后面
fmt.Println("Before append to self:", slice)
slice = append(slice, slice...)
fmt.Println("After append to self:", slice)
{% endhighlight %}

对于最后一段代码，可以思考下如何设计才能让这个简单的调用正确地工作。

在社区共建的["Slice Tricks" Wiki page](https://golang.org/wiki/SliceTricks)，有大量的append, copy等使用切片的例子。大家可以参考。

### Nil

通过我们新学到的知识，我们能知道nil切片是什么。显然，它的切片头是零值。

	sliceHeader{
		Length:         0,
		Capacity:       0,
		ZerothElement:  nil,
	}
	
或者：

	sliceHeader{}

这个关键点是元素指针也是nil。下面例子创建的切片:

	array[0:0]

长度是0（可能容量也是0），但它的指针不是nil，所以它不是nil切片。

一个空的切片是可以增长的（假设它的容量不为0），但nil切片是不能增长的，因为没有数组存放元素。

但是，nil切片在功能上等价于空切片，即使它的元素指针是nil。它的长度是0，可以被append。前面的例子中，就有通过向nil切片追回切片来达到拷贝切片的功能。

### 字符串
本节我们讨论一下在切片场景下的字符串用法。

字符串实际很简单：它是一个只读的byte类型切片，并在语言层面有一些额外的语法支持。

由于是只读的，所以不需要容量参数（你不能增长它）。绝大多数情况下，你都可以把它看作一个只读的byte类型切片。

首先，我们通过下标来访问单独的字符：

	slash := "/usr/ken"[0] // 返回字符'/'。

我们也能通过切片一个字符串来截取一个子字符串:

	usr := "/usr/ken"[0:4] // 返回字符串"/usr"

我们也通过一个简单地转换，将一个byte类型切片转换成字符串:

	str := string(slice)

反过来，转换也可以进行：

	slice := []byte(usr)
	
一个数组隐藏在字符串下面。只有通过字符串操作，才能访问它的内容。这就意味着当我们进行这两个中任意一个转换时，必须生成一个数组的拷贝。当然，Go会处理这些。在这两个中任意一个转换完成后，改变切片指向的数组不影响对应的字符串。

将字符串设计的像切片一样，其中一个重要的好处就是能高效地创建一个子串。这个操作只需要创建一个两字节字符串的头。因为字符串是只读的，所以初始字符串以及通过切片生成的字符串能安全地共享相同的数组。

字符串最早期的实现总是要分配内存的，不过当切片被加到语言中时，他们提供了一个有效操作字符串的模型。一些基准测试也表明这样做的巨大性能提升。

当然，关于字符串还有很多知识，一个单独的[blog post](http://blog.golang.org/strings)更深入地讨论了这些。

### 结论
明白切片如何工作，能帮助我们理解它们是怎么实现的。有一个小的数据结构：关联切片的切片头，这个头描述了单独分配的数组的一段。当我们传递切片值时，头将会被拷贝，但指向的数组总是共享的。

一旦我们明白了切片怎么工作，它们就会变得不仅易用，而且强大和有表现力。特别有copy和append这些内置函数的帮忙。

### 进一步阅读
有许多围绕切片的文章。早前提到的["Slice Tricks" Wiki page](https://golang.org/wiki/SliceTricks)有许多例子。[Go Slices](http://blog.golang.org/go-slices-usage-and-internals)用清晰的图表描述了切片的内存布局。Russ Cox写的[Go Data Structures](http://research.swtch.com/godata)文章包含了一些切片和Go内置的一些其它数据结构的讨论。

尽管有很多的资料可以参考，但学习切片最好的办法就是使用它们。
