## golang源码sort包解读

golang的sort包对外主要暴露了两个排序函数，一个是基于快排骨架的Sort函数，一个是基于归并的Stable函数。前者面对各种极端情况可能效率不稳定但是整体期望效率较高，是非常常用的排序算法实现；后者虽然期望效率不如前者，但是能保证对各种极端情况排序的效率趋于稳定。介于流行程度和分析价值，这里我们主要讲解Sort函数的实现，它的实现也更有意思。

必要的，要调用sort包里暴露的方法，需要先实现sort.Interface接口，该接口定义如下,我仅将注释稍作翻译：

```go
// 集合的元素要求能够被int索引
type Interface interface {
	// 返回集合元素的个数
	Len() int
	// 若索引i对应的元素应该排在索引j之前，返回true；反之返回false
	Less(i, j int) bool
	// 交换集合中i，j元素的位置
	Swap(i, j int)
}
```

对实现了上述接口的某个对象实例 instance，可以直接调用sort.Sort(instance)，运行完成后的instance已排好序。

sort.Sort()函数直接调用了```quickSort(data Interface, a, b, maxDepth int)```函数，这里我们注意到函数的入参与教科书里的快排相比多了一个maxDepth。该值由data的长度data.Len()作为入参调用maxDepth函数计算而来：
```go
// maxDepth returns a threshold at which quicksort should switch
// to heapsort. It returns 2*ceil(lg(n+1)).
func maxDepth(n int) int {
	var depth int
	for i := n; i > 0; i >>= 1 {
		depth++
	}
	return depth * 2
}
```
从注释可以看出，该maxDepth值的作用是观测到快排递归出现了明显的退化现象时，即递归深度超过了maxDepth阈值的时候，将快排切换成堆排，防止进一步的递归调用。这样做可以防止人为构造的数据对快排算法形成攻击，使算法始终运行在极端的最坏情况，从而降低计算机单位时间的响应能力来达到攻击目的。

下面我们来看看quickSort函数的具体实现，先直接贴出源码：
```go
func quickSort(data Interface, a, b, maxDepth int) {
	for b-a > 12 { // Use ShellSort for slices <= 12 elements
		if maxDepth == 0 {
			heapSort(data, a, b)
			return
		}
		maxDepth--
		mlo, mhi := doPivot(data, a, b)
		// Avoiding recursion on the larger subproblem guarantees
		// a stack depth of at most lg(b-a).
		if mlo-a < b-mhi {
			quickSort(data, a, mlo, maxDepth)
			a = mhi // i.e., quickSort(data, mhi, b)
		} else {
			quickSort(data, mhi, b, maxDepth)
			b = mlo // i.e., quickSort(data, a, mlo)
		}
	}
	if b-a > 1 {
		// Do ShellSort pass with gap 6
		// It could be written in this simplified form cause b-a <= 12
		for i := a + 6; i < b; i++ {
			if data.Less(i, i-6) {
				data.Swap(i, i-6)
			}
		}
		insertionSort(data, a, b)
	}
}
```
该快排的实现代码在不同情况下分别调用了插入、堆、希尔排序。这几种排序和快排一样都是原址排序，所以他们能够无缝互相配合，只需由快排骨架根据情况判断并指定哪种排序方法去排序哪一段。

第一层的for循环和if语句比较好懂：排序总长度b-a小于等于12的时候使用希尔排序，具体的，使用间距为6的希尔排序完成初排，再用插入排序完成最终的排序。

我们再回过头来看第一个for循环内部：
首先是刚才提到的退化情形：当递归深度值maxDepth降低到0的时候，直接使用堆排序完成本轮排序，不再递归；接着便是快排的主元选取，一般的教材中，该过程选取当前排序区间的第一位或者最后一位元素作为主元，通过一次遍历将区间分成两部分，左边的部分均小于主元，右边的部分均大于主元。这部分的操作这里由doPivot()函数实现，但是这里注意到该函数返回了两个参数：mlo，mhi，我们现在只需要记住mlo左边的元素均小于mlo对应的元素，mhi右边的元素均大于mhi对应的元素，我们等会儿再来分析这个函数。

一般的教材里，接下来便是分两段递归，然后函数结束，但是这里却稍有不同：
```go
// Avoiding recursion on the larger subproblem guarantees
// a stack depth of at most lg(b-a).
if mlo-a < b-mhi {
	quickSort(data, a, mlo, maxDepth)
    a = mhi // i.e., quickSort(data, mhi, b)	
} else {
	quickSort(data, mhi, b, maxDepth)
	b = mlo // i.e., quickSort(data, a, mlo)
}
```

为了避免对大规模集合排序递归深度过大导致的栈空间的消耗，这里的实现方法是：短的一半进入下次递归，长的一半由for循环代替，不再进入递归，从而节省了栈空间的消耗。

至此，Sort函数的基本脉络我们已经理清，我们最后来看看doPivot函数在干什么？为什么它要返回两个值呢？

```go
func doPivot(data Interface, lo, hi int) (midlo, midhi int) {
	m := lo + (hi-lo)/2 // Written like this to avoid integer overflow.
	if hi-lo > 40 {
		// Tukey's ``Ninther,'' median of three medians of three.
		s := (hi - lo) / 8
		medianOfThree(data, lo, lo+s, lo+2*s)
		medianOfThree(data, m, m-s, m+s)
		medianOfThree(data, hi-1, hi-1-s, hi-1-2*s)
	}
	medianOfThree(data, lo, m, hi-1)
	...
}
```
开头的主元选取也很有意思，通过采样的方法使得主元在概率上小一点，这样分出来的区间更可能一段大一段小。接下来的代码便是正常的与主元比较、元素交换的操作，下面这段之后，区间便已经分成两段，但是主元还未换到中间。
```go
	pivot := lo
	a, c := lo+1, hi-1

	for ; a < c && data.Less(a, pivot); a++ {
	}
	b := a
	for {
		for ; b < c && !data.Less(pivot, b); b++ { // data[b] <= pivot
		}
		for ; b < c && data.Less(pivot, c-1); c-- { // data[c-1] > pivot
		}
		if b >= c {
			break
		}
		// data[b] > pivot; data[c-1] <= pivot
		data.Swap(b, c-1)
		b++
		c--
	}
```
然而接下来这么大一段又是在干什么呢？
```go
// If hi-c<3 then there are duplicates (by property of median of nine).
// Let be a bit more conservative, and set border to 5.
protect := hi-c < 5
fmt.Println(protect, hi, "", c)
if !protect && hi-c < (hi-lo)/4 {
    // Lets test some points for equality to pivot
    dups := 0
    if !data.Less(pivot, hi-1) { // data[hi-1] = pivot
        data.Swap(c, hi-1)
        c++
        dups++
    }
    if !data.Less(b-1, pivot) { // data[b-1] = pivot
        b--
        dups++
    }
    // m-lo = (hi-lo)/2 > 6
    // b-lo > (hi-lo)*3/4-1 > 8
    // ==> m < b ==> data[m] <= pivot
    if !data.Less(m, pivot) { // data[m] = pivot
        data.Swap(m, b-1)
        b--
        dups++
    }
    // if at least 2 points are equal to pivot, assume skewed distribution
    protect = dups > 1
}
if protect {
    // Protect against a lot of duplicates
    // Add invariant:
    //	data[a <= i < b] unexamined
    //	data[b <= i < c] = pivot
    for {
        for ; a < b && !data.Less(b-1, pivot); b-- { // data[b] == pivot
        }
        for ; a < b && data.Less(a, pivot); a++ { // data[a] < pivot
        }
        if a >= b {
            break
        }
        // data[a] == pivot; data[b-1] < pivot
        data.Swap(a, b-1)
        a++
        b--
    }
}
// Swap pivot into middle
data.Swap(pivot, b-1)   
return b - 1, c
```
仔细阅读注释和代码可以发现，这段代码是为了处理重复元素过多而存在的，返回的两个索引值，对应的区间里的元素值都是主元，如果这里把重复的主元全部找到，那么这段元素就不用进入下次的递归，这在重复元素很多的情况下能起到非常大的优化效果，目的是减少递归或循环的次数。

这就是一个工业级Sort函数的实现，是不是和你在教材里看过的十几行写完的快速排序差别很大，仔细体会，你能学到更多。

（完）
