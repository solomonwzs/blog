---
title: Golang Optimization
date: 2017-10-25
tags:
- golang
categories:
- note
---

Golang 优化技巧

<!-- more -->

## unsafe.Pointer & uintptr

### unsafe.Pointer

`unsafe.Pointer` 类似与C语言中的 `void` 指针，指向一个变量的地址，可以转换成其他具体类型的指针，下面两段代码的效果相等：

Go 代码
```go
func main() {
	var f float32 = 12.5
	iptr := (*int)(unsafe.Pointer(&f))
	fmt.Println(*iptr) // output: 1095237632
}
```

C 代码
```c
int main() {
  float f = 12.5;
  int *iptr = (int*)&f;
  printf("%d\n", *iptr); // output: 1095237632
  return 0;
}
```

### uintptr

`uintptr` 可以把 `unsafe.Pointer` 转换成整数，主要用于计算地址的偏移，下面两段代码的效果相等：

Go 代码
```go
func main() {
	b := []byte{12, 34, 56, 78}
	ptr := unsafe.Pointer(&b[0])
	fmt.Println(*(*byte)(unsafe.Pointer(uintptr(ptr) + 1))) // output: 34
}
```

C 代码
```c
int main() {
  int arr[] = {12, 34, 56, 78};
  int *b = arr;
  printf("%d\n", *(b + 1)); // output: 34
  return 0;
}
```

### 使用安全

注意，`uintptr` 与 `unsafe.Pointer` 之间的转换必须写在同一个表达式内，不能用临时变量存储 `uintptr`，例如，下面的写法是错误的：

```go
tmp := uintptr(ptr)
fmt.Println(*(*byte)(unsafe.Pointer(tmp + 1)))
```

这是因为 `uintptr` 是一个数字，Go 内部有垃圾回收机制，这时变量的地址会改变，但 `uintptr` 的值不会更新

## 字符串转换与字节数组间转换

一般转换方法：

```go
s := "abcd"
arr := []byte(s)
```

转换时，系统会在堆上为 `arr` 分配一块内存，然后把字符串里的内容拷贝进去

透过 `gdb` 可知 `string` 和 `[]byte` 的底层结构：

```
(gdb) l main.main
3	import (
4		"fmt"
5		"unsafe"
6	)
7
8	func main() {
9		s := "0123456789"
10		arr := []byte(s)
11		fmt.Println(s, arr)
12	}
(gdb) b 11
Breakpoint 1 at 0x487b49: file /home/solomon/workspace/go/t/main.go, line 11.
(gdb) r
Starting program: /home/solomon/workspace/go/t/main
[New LWP 1338]
[New LWP 1339]
[New LWP 1340]
[New LWP 1341]

Thread 1 "main" hit Breakpoint 1, main.main () at /home/solomon/workspace/go/t/main.go:11
11		fmt.Println(s, arr)
(gdb) info locals
s = 0x4b7609 "0123456789"
arr = {array = 0xc4200140f0 "0123456789", len = 10, cap = 16}
(gdb) ptype s
type = struct string {
    uint8 *str;
    int len;
}
(gdb) ptype arr
type = struct []uint8 {
    uint8 *array;
    int len;
    int cap;
}
```

因此可通过 `unsafe.Pointer` 来构造 `string` 或者 `[]byte` 来避免内存数据的拷贝

```go
package main

import (
	"strings"
	"testing"
	"unsafe"
)

func str2bytes(s string) []byte {
	x := (*[2]unsafe.Pointer)(unsafe.Pointer(&s))
	h := [3]unsafe.Pointer{x[0], x[1], x[1]}
	return *(*[]byte)(unsafe.Pointer(&h))
}

func bytes2str(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}

var _s = strings.Repeat("a", 10240)

func BenchmarkNormalConvert(b *testing.B) {
	for i := 0; i < b.N; i++ {
		b := []byte(_s)
		_ = string(b)
	}
}

func BenchmarkPointerConvert(b *testing.B) {
	for i := 0; i < b.N; i++ {
		b := str2bytes(_s)
		_ = bytes2str(b)
	}
}
```

输出：

```
-> % go test -v -bench . -benchmem
goos: linux
goarch: amd64
BenchmarkNormalConvert-4    	 1000000	      1675 ns/op	   20480 B/op	       2 allocs/op
BenchmarkPointerConvert-4   	500000000	         3.09 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	_/home/solomon/workspace/go/t	3.562s
```

注意，这个方法触动到底层被隐藏的细节，如果有一天底层的数据结构有改动，程序就会出错，因此除非转换成为瓶颈，否则不建议使用
