

# heap源码解析

## heap.Interface接口

heap是一个小顶堆的实现，若想将集合变成一个小顶堆，需要实现`heap.Interface`。

```go
// The Interface type describes the requirements
// for a type using the routines in this package.
// Any type that implements it may be used as a
// min-heap with the following invariants (established after
// Init has been called or if the data is empty or sorted):
//
// !h.Less(j, i) for 0 <= i < h.Len() and 2*i+1 <= j <= 2*i+2 and j < h.Len()
//
// Note that Push and Pop in this interface are for package heap's
// implementation to call. To add and remove things from the heap,
// use heap.Push and heap.Pop.
type Interface interface {
   sort.Interface
   Push(x interface{}) // add x as element Len()
   Pop() interface{}   // remove and return element Len() - 1.
}
```

实现`heap.Interface`需要实现以下5个方法：

- `sort.Interface`定义了三个方法（`sort.Interface`一般是一个集合）
  - `Len()`：返回集合元素的数量
  - `Swap(i, j int)`：交换第`i`和第`j`个元素
  - `Less(i, j int) bool`：判断第`i`个元素是否小于第`j`个元素
- `Push(x interface{})`：在集合第`Len()`个位置添加`x`
- `Pop() interface{}`：删除并放回集合第`Len()-1`个元素

## 函数

### 初始化

`Init(h Interface)`初始化实现类型中的集合为一个小顶堆，时间复杂度为O(n)。

```go
// Init establishes the heap invariants required by the other routines in this package.
// Init is idempotent with respect to the heap invariants
// and may be called whenever the heap invariants may have been invalidated.
// The complexity is O(n) where n = h.Len().
func Init(h Interface) {
   // heapify
   n := h.Len()
   // n/2 -1 表示一个小顶堆的最后一个父节点（有子节点）
  // 从最后一个父节点依次向上调用down(h, i, n)将每一个父子结构调整为一个小顶堆。
   for i := n/2 - 1; i >= 0; i-- {
      down(h, i, n)
   }
}
```

`down(h Interface, i0, n int) bool`方法将每一个父子结构调整为一个小顶堆，返回一个bool值（该父子结构是否被调整过）。

```go
func down(h Interface, i0, n int) bool {
  // 父节点
	i := i0
	for {
		j1 := 2*i + 1 // 左孩子
    // 左孩子不能超过len-1, j1<0是因为int可能会溢出，导致j1<0
		if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
			break
		}
    // 因为小顶堆的父节点必须小于孩子节点，
    // 这里先比较左右孩子的大小，将较大的那一个的 index 赋值给 j，
    // 最后比较父节点（i）和较大的孩子节点（j）的大小。
		j := j1 // left child
		if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
			j = j2 // = 2*i + 2  // right child
		}
    // 如果父节点比两个孩子节点小，则跳出循环
		if !h.Less(j, i) {
			break
		}
    // 如果父节点比两个孩子节点大，则将父节点和孩子节点中较大的那个交换。
		h.Swap(i, j)
		i = j
	}
  // 如果发生了交换，则 i > io，返回true，反之则为false。
	return i > i0
}
```

### Push

`Push(h Interface, x interface{})`：将一个元素添加到`heap`，然后通过`up(h, h.Len()-1)`调整堆结构。

```go
// Push pushes the element x onto the heap.
// The complexity is O(log n) where n = h.Len().
func Push(h Interface, x interface{}) {
   h.Push(x)
   up(h, h.Len()-1)
}
```

`up(h Interface, j int)`：向上调整，直到根节点或者父节点大于子节点。

```go
// 向上调整
func up(h Interface, j int) {
   for {
      // 父节点为 (j-1)/2
      i := (j - 1) / 2 // parent
      // i == j 已经up到根节点了
     // !h.Less(j, i)表示父节点大于子节点，不用再向上调整了
      if i == j || !h.Less(j, i) {
         break
      }
      h.Swap(i, j)
      j = i
   }
}
```

### Pop

`Pop(h Interface) interface{}`：弹出堆顶元素，然后将集合最后一个元素放到堆顶，最后执行down操作调整堆。

```go
// Pop removes and returns the minimum element (according to Less) from the heap.
// The complexity is O(log n) where n = h.Len().
// Pop is equivalent to Remove(h, 0).
func Pop(h Interface) interface{} {
   n := h.Len() - 1
   h.Swap(0, n)
   down(h, 0, n)
   return h.Pop()
}
```

### Remove

`Remove(h Interface, i int) interface{}`：删除集合中下标为`i`的节点，返回被删的节点。

```go
// Remove removes and returns the element at index i from the heap.
// The complexity is O(log n) where n = h.Len().
func Remove(h Interface, i int) interface{} {
   n := h.Len() - 1
   if n != i {
      // 交换第i个元素和最后一个元素
      h.Swap(i, n)
      // 最后一个元素如果不执行down，就执行up。
      if !down(h, i, n) {
         up(h, i)
      }
   }
   return h.Pop()
}
```

### Fix

`Fix(h Interface, i int)`：当对上的节点值发生变化时，可以使用`Fix`来调整堆结构。

```go
// Fix re-establishes the heap ordering after the element at index i has changed its value.
// Changing the value of the element at index i and then calling Fix is equivalent to,
// but less expensive than, calling Remove(h, i) followed by a Push of the new value.
// The complexity is O(log n) where n = h.Len().
func Fix(h Interface, i int) {
   if !down(h, i, h.Len()) {
      up(h, i)
   }
}
```