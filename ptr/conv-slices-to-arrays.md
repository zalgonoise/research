## Concept

This document will cover how to convert a slice of a type into an array of the same type, containing the same data, in Go. It will cover basic concepts like *what is a slice* and *what is an array*, as well as all the nuances to using either; and lastly how to convert one into the other.

For this particular approach I will not be using the `copy(dest, src []T)` built-in function, but using the `unsafe` package to extract the backing array from a slice, instead. It is a very straight-forward process as can be seen in this document besides a few catches, of course.

Finally there is a section for benchmarks where both approaches (with using `copy()` and with pointer manipulation) to see how the approach differs from the simplest one, side by side.

This document was written with Go currently on version 1.19, on January 2023.

______________________

## Specification

### What is an array

An array in Go is a collection of items of a certain type, with a fixed-length. To quote [the Go specification document for arrays](https://go.dev/ref/spec#Array_types): 

> An array is a numbered sequence of elements of a single type, called the element type. The number of elements is called the length of the array and is never negative.
>
> The length is part of the array's type; it must evaluate to a non-negative [constant](https://go.dev/ref/spec#Constants) [representable](https://go.dev/ref/spec#Representability) by a value of type int. The length of array a can be discovered using the built-in function [len](https://go.dev/ref/spec#Length_and_capacity). The elements can be addressed by integer [indices](https://go.dev/ref/spec#Index_expressions) 0 through len(a)-1.

#### Constant values for the array size

It's important to note the part where the length of the array is a **constant int value**. This will be key later on, but it's significant to underline that constants do not work the same way as variables -- they are compiled with your code, and are immutable during runtime (they cannot change). More on constants / variables, below.

Having a constant value for the array length, in a nutshell, limits how you create arrays. The following examples **are not** possible in Go:

```go
package main

import "fmt"

func ToArr[T any](slice []T, size int) any {
	v := [size]T{}               // error: ./prog.go:8:8: invalid array length size
	copy(v[:], slice)
	return v
}

func Grow[T any](slice []T, size int) any {
	v := [len(slice) + size]T{}  // error: ./prog.go:14:8: array length len(slice) + size (value of type int) must be constant
	copy(v[:], slice)
	return v
}

func main() {
	ints := []int{1, 2, 3}
	newInts := ToArr[int](ints, 5)
	expanded := Grow[int](ints, 2)
	fmt.Printf("%v %T", expanded, expanded)
	fmt.Printf("%v %T", newInts, newInts)
}
```

#### What it means to have a fixed length

Since the length is fixed, it's impossible to add more items to an array that is already full, as it results in an index-out-of-bounds error when building the binary. Take [the following example](https://go.dev/play/p/AU6kvPWDrmu):

```go
package main

import "fmt"

func main() {
	ints := [3]int{1, 2, 3}
	ints[3] = 4
	fmt.Println(ints)
}
```

This code results in the following error when building the binary:

```
./prog.go:9:7: invalid argument: index 3 out of bounds [0:3]
```

When working with arrays, to append a value to it you would need to create a new array (of the new size); copy the previous array's contents; then modify the new index(es) with the desired values, [like this example](https://go.dev/play/p/_TtKCcBzbku):

```go
package main

import "fmt"

func main() {
	ints := [3]int{1, 2, 3}
	newInts := [5]int{}
	copy(newInts[:], ints[:])
	newInts[3] = 4
	newInts[4] = 5
	fmt.Printf("%v %T", newInts, newInts) // prints: [1 2 3 4 5] [5]int
}
```


### What is a slice

Like an array, a slice is a collection if items of a certain type, but with variable capacity. This means that the caller can use the `append([]T, ...T)` built-in function "expand" the backing array just like in the last example, while also abstracting this whole concept of having a constant int value for a capacity. To quote [the Go specification document for arrays](https://go.dev/ref/spec#Slice_types): 

> A slice is a descriptor for a contiguous segment of an underlying array and provides access to a numbered sequence of elements from that array. A slice type denotes the set of all slices of arrays of its element type. The number of elements is called the length of the slice and is never negative. The value of an uninitialized slice is nil.
>
> The length of a slice s can be discovered by the built-in function [len](https://go.dev/ref/spec#Length_and_capacity); unlike with arrays it may change during execution. The elements can be addressed by integer [indices](https://go.dev/ref/spec#Index_expressions) 0 through len(s)-1. The slice index of a given element may be less than the index of the same element in the underlying array.
>
> A slice, once initialized, is always associated with an underlying array that holds its elements. A slice therefore shares storage with its array and with other slices of the same array; by contrast, distinct arrays always represent distinct storage.
>
> The array underlying a slice may extend past the end of the slice. The capacity is a measure of that extent: it is the sum of the length of the slice and the length of the array beyond the slice; a slice of length up to that capacity can be created by [slicing](https://go.dev/ref/spec#Slice_expressions) a new one from the original slice. The capacity of a slice a can be discovered using the built-in function [cap(a)](https://go.dev/ref/spec#Length_and_capacity).

This part explains how a slice is a data structure with three elements: a pointer to the backing array containing the data in the slice, the length of the slice (how many items have been added to the slice), and the capacity of the slice (how many items the slice is able to store). Capacity in a slice is not a limitation, it is actually a threshold before a `grow` operation occurs. This `grow` operation will be similar to the example in the Arrays section, when expanding an array. It involves spawning a new array (of greater length), copying the items into it, and returning the new array. In the realm of slices, the pointer to the backing array is replaced, and the capacity updated. The length would only grow as the caller added items into the slice.

Go makes it all so simple in the context that it is a statically-typed programming language. This amount of flexibility under-the-hood is truly appreciated when you dive into it.

#### Breaking it down

For this, it's great to refer the [*Go Slices intro* blog post](https://go.dev/blog/slices-intro). It explains exactly how these concepts work (with images!) in-depth.

Besides this document, it's great to look at Go's source code for the slices definition, in [`src/runtime/slice.go`](https://github.com/golang/go/blob/master/src/runtime/slice.go#L15-L19); which is showing this exact representation:

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```


### What is a constant

A constant is an immutable value with an identifier, of a certain (supported) type. Immutability is a keyword here, constants cannot be changed / updated. While it's possible to define constants with a number of operations, it's worth to underline that the definition is only valid if the underlying values are constant as well. Invalid constant definitions raise errors on compile-time. To quote [the Go specification document for arrays](https://go.dev/ref/spec#Constants):

> A constant value is represented by a [rune](https://go.dev/ref/spec#Rune_literals), [integer](https://go.dev/ref/spec#Integer_literals), [floating-point](https://go.dev/ref/spec#Floating-point_literals), [imaginary](https://go.dev/ref/spec#Imaginary_literals), or [string](https://go.dev/ref/spec#String_literals) literal, an identifier denoting a constant, a [constant expression](https://go.dev/ref/spec#Constant_expressions), a [conversion](https://go.dev/ref/spec#Conversions) with a result that is a constant, or the result value of some built-in functions such as unsafe.Sizeof applied to [certain values](https://go.dev/ref/spec#Package_unsafe), cap or len applied to [some expressions](https://go.dev/ref/spec#Length_and_capacity), real and imag applied to a complex constant and complex applied to numeric constants. The boolean truth values are represented by the predeclared constants true and false. The predeclared identifier [iota](https://go.dev/ref/spec#Iota) denotes an integer constant.

### What is a variable

A variable is a mutable value with an identifier, of any (valid) type. Mutability allows a wider range of variable definitions, such as initializing a pointer to a type or even storing a function with an identifier. To quote [the Go specification document for arrays](https://go.dev/ref/spec#Variables):

> A variable is a storage location for holding a value. The set of permissible values is determined by the variable's [type](https://go.dev/ref/spec#Types).
>
> A [variable declaration](https://go.dev/ref/spec#Variable_declarations) or, for function parameters and results, the signature of a [function declaration](https://go.dev/ref/spec#Function_declarations) or [function literal](https://go.dev/ref/spec#Function_literals) reserves storage for a named variable. Calling the built-in function [new](https://go.dev/ref/spec#Allocation) or taking the address of a [composite literal](https://go.dev/ref/spec#Composite_literals) allocates storage for a variable at run time. Such an anonymous variable is referred to via a (possibly implicit) [pointer indirection](https://go.dev/ref/spec#Address_operators).
>
> Structured variables of [array](https://go.dev/ref/spec#Array_types), [slice](https://go.dev/ref/spec#Slice_types), and [struct](https://go.dev/ref/spec#Struct_types) types have elements and fields that may be [addressed](https://go.dev/ref/spec#Address_operators) individually. Each such element acts like a variable.


______________________

## Converting slices and arrays

### Array to slice

Converting an array to a slice is simple -- by casting its elements without a starting or ending index. This can be done with or without using the `copy()` function, which would determine if the slice is independent of the original array; or if changes in the original array would be reflected on the slice. Take [the example below to show both approaches](https://go.dev/play/p/IzC-5J3v4Rs):

```go
package main

import "fmt"

func main() {
	ints := [3]int{1, 2, 3}

	// type-conversion of [3]int to []int
	sliceA := ints[:]

	// element copy from [3]int into []int
	sliceB := make([]int, len(ints))
	copy(sliceB, ints[:])

	// now we alter the backing array
	ints[0] = 9

	// and inspect the slices elements
	fmt.Printf("%v %T\n", sliceA, sliceA) // prints: [9 2 3] []int
	fmt.Printf("%v %T\n", sliceB, sliceB) // prints: [1 2 3] []int
}
```

It's important to show the impact upfront of when you use `copy()` in this type of conversions. Both routes are acceptable provided you're aware of the concequences -- and both routes are valid if they are used appropriately (with mutability in mind): do you need a slice mutable by the backing array that created it; or slice that is completely independent of the backing array? As simple as that.

Note how `sliceB` is initialized with the length of the `ints` array. This is because if you were to initialize the slice as `sliceB := []int{}`, the resulting object would have a capactity of 0. This means that copying any number of items into it would not result in any changes to the slice because it would need to grow (in capacity) first. That would be resolved with `sliceB := append([]int{}, ints[:]...)` which is ready to grow a slice when necessary.


### Slice to array

Now this one is not as straightforward. There is that original strategy of copying the contents from A to B which is also valid for this approach, something that goes [like the following example](https://go.dev/play/p/haohLKwNLLb):

```go
package main

import "fmt"

func main() {
	slice := []int{1, 2, 3}

	ints := [3]int{}
	copy(ints[:], slice)

	// now we alter the backing slice
	slice[0] = 9

	// and inspect the slices elements
	fmt.Printf("%v %T\n", ints, ints) // prints: [1 2 3] [3]int
}
```

This is perfectly OK and it shares the same behavior as copying an array elements into a slice. It is in fact the recommended way of going about this, in general.

#### Pointer manipulation

...What if it isn't the *only* way, though? Just a couple of chapters above, we can see the underlying data structure for a slice and it's composed of a pointer to an array, its length and the slice's capacity -- would it be possible to fetch the array directly from that pointer? The answer is yes! You're definitely able to get the backing array from that slice; with a few catches when it comes to defining that array's length.

So the plan to this *unsafe* approach is:

1. Convert the slice to an `unsafe.Pointer`
2. Convert the pointer to a 3-element-array of `uint` or `int`. This should be perceived as the slice's data structure, which will be referred to as metadata
    - the first element in this array is the pointer to the backing array
    - the second element in this array is the underlying array's length
    - the third element in this array is the slice's capacity value
3. Extract the first element of the slice's metadata, and convert it to an `unsafe.Pointer`
4. Have an int constant preemptively defined, so that you can convert this pointer to an array of that length (hardest step in this process)
5. Work with the array

And it definitely works as you would expect -- all the slice data structure elements will be visible and accessible, provided that you have already some information upfront.

Let's break down the steps with Go code:

1. Convert the slice to an `unsafe.Pointer`

```go
	slice := []int{1, 2, 3}
	ptr := unsafe.Pointer(&slice) 
```


2. Convert the pointer to a 3-element-array of `uint` or `int`. This should be perceived as the slice's data structure, which will be referred to as metadata
  - the first element in this array is the pointer to the backing array
  - the second element in this array is the underlying array's length
  - the third element in this array is the slice's capacity value

```go
	slice := []int{1, 2, 3}
	ptr := unsafe.Pointer(&slice)
	sliceData := *(*[3]uint)(ptr) // sliceData: [824634354440 3 3] [3]uint 
```

3. Extract the first element of the slice's metadata, and convert it to an `unsafe.Pointer`

```go
	arrPtr := unsafe.Pointer(uintptr(sliceData[0]))
```

4. Have an int constant preemptively defined, so that you can convert this pointer to an array of that length (hardest step in this process)

```go
	const size = 3
	// (...)
	arr := *(*[size]int)(arrPtr) // arr: [1, 2, 3] [3]int
```

5. Work with the array


#### Details and caveats

This approach works *exactly* like `copy()`. The resulting array will be completely separate from the slice and updating the slice will not affect the new array. Here is [an example](https://go.dev/play/p/7ZYKlRps2Cx):

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	slice := []int{1, 2, 3}

	sliceData := *(*[3]uint)(unsafe.Pointer(&slice))
	fmt.Printf("%v %T\n", sliceData, sliceData) // prints: [824634199712 3 3] [3]uint -- first value is a memory address so it changes

	arr := *(*[3]int)(unsafe.Pointer(uintptr(sliceData[0])))

	// now we alter the backing slice
	slice[0] = 9

	// and inspect the slices elements
	fmt.Printf("%v %T\n", arr, arr) // prints: [1 2 3] [3]int
}
```

The pros is that it is just a little bit faster than `copy()`. The cons is just `unsafe` in general -- this means that you're working with raw memory addresses and it's very easy to mess up. One example of this is if you do not set the right checks for the slice's capacity, as you are able to extract an array of `n` items -- inclusively other data if that value overflows the capacity of your slice. See the [following example](https://go.dev/play/p/iO4kHxj0N59) where I try to take an array of 15 elements from the address of the backing array of a slice containing 3 items:

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	slice := []int{1, 2, 3}

	sliceData := *(*[3]uint)(unsafe.Pointer(&slice))
	fmt.Printf("%v %T\n", sliceData, sliceData) // prints: [824634199128 3 3] [3]uint

	arr := *(*[15]int)(unsafe.Pointer(uintptr(sliceData[0])))

	for _, item := range arr {
		// try to inspect memory addresses.
		// 64 is an easy filter to avoid casting the capacities, lenghts and zero values
		if item > 64 {
			data := *(*[15]int)(unsafe.Pointer(uintptr(item)))
			fmt.Printf("%v %T\n", data, data)
		}
	}
}
```

The loops output would be something like:

```
[1 2 3 824634199128 3 3 824634199128 3 3 16 5421664 824633803344 0 0 824634199280] [15]int
[1 2 3 824634199128 3 3 824634199128 3 3 16 5421664 824633803344 0 0 824634199280] [15]int
[5420800 0 0 0 0 0 0 0 0 12 824633745408 0 0 0 0] [15]int
[4841220 29 0 0 0 0 0 0 0 0 0 0 0 0 0] [15]int
[824634199312 4726929 5334536 7 824634199616 4459911 5334400 824633984400 4266928 139991418156800 2109440 0 4342405 0 7] [15]int
[1 2 3 824634199128 3 3 824634199128 3 3 16 5421664 824633803344 0 0 824634199280] [15]int
```

If a converter function is written, this check needs to be set where an error would be returned for a length / capacity mismatch.


#### Generics

Say, you wanted to have a generic function that converts any slice into an array of the same size. That's not going to be a very straight-forward process. Regarding all steps that involve pointer manipulation; you're good-to-go. You have access to the values, the slice's capacity and length, but the issue is the constant definition -- it's impossible to type-cast the pointer to an array if you do not have that value as a constant already.

How to avoid this? By brute-force of course! I made a quick converter (I will explain "quick" below) for slices with generics and all, fits all types, it's great. The problem is that your slice is expected to have a capacity of a log(2) value -- it would be the only way to support greater values (over 1k) without literally initializing a library with thousands of constants to fit all needs. In a nutshell, if you want to convert a slice to a [256]T or [2048]T you can; if it's a value outside of log(2) up and under 10k, you need to implement it yourself. 

[Here is the code](https://github.com/zalgonoise/x/blob/master/ptr/slices.go#L65) for this converter:


```go
package ptr

import (
	"errors"
	"unsafe"
)

var ErrInvalidCap = errors.New("invalid slice capacity")

const (
	Cap1 int = 1 << iota
	Cap2
	Cap4
	Cap8
	Cap16
	Cap32
	Cap64
	Cap128
	Cap256
	Cap512
	Cap1024
	Cap2048
	Cap4096
	Cap8192
)


func ToArray[T any](slice []T) (any, error) {
	sliceData := *(*[3]uint)(unsafe.Pointer(&slice))

	switch sliceData[2] {
	case 1:
		return *(*[Cap1]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 2:
		return *(*[Cap2]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 4:
		return *(*[Cap4]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 8:
		return *(*[Cap8]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 16:
		return *(*[Cap16]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 32:
		return *(*[Cap32]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 64:
		return *(*[Cap64]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 128:
		return *(*[Cap128]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 256:
		return *(*[Cap256]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 512:
		return *(*[Cap512]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 1024:
		return *(*[Cap1024]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 2048:
		return *(*[Cap2048]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 4096:
		return *(*[Cap4096]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	case 8192:
		return *(*[Cap8192]T)(unsafe.Pointer(uintptr(sliceData[0]))), nil
	default:
		return nil, ErrInvalidCap
	}
}
```

For brevity, I have removed the code documentation for this function. But, in a nutshell, we can see the 5 steps mentioned above applied to this function, with two actions in mind -- getting the slice's metadata; and then switching on its capacity value. Defaults to the `invalid capacity` error, otherwise it can convert the slice to an array.

One caveat is returning `any` type, which implies that the caller will need to type-convert that `any` type to the corresponding array type. None of these steps will make it any easier than using `copy()` in the end; solely for a readability and maintainability point-of-view. Great research material, though.

### Benchmarks

So, how do these approaches compare to each other? Is pointer-manipulation that advantageous? Is copy that slow to begin with? Turns out the difference is quite negligible however manipulating pointers is a tiny bit faster. Made me happy (mostly getting it to work the way I wanted it to, but the performance was a big deal as well).

To test the performance for either approach, I wrote the following benchmark test:

```go
package ptr_test

import (
	"errors"
	"reflect"
	"testing"

	. "github.com/zalgonoise/x/ptr"
)

func BenchmarkToArray(b *testing.B) {
	b.Run("PointerConversion256Elems", func(b *testing.B) {
		input := []int{
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
		}
		wants := [256]int{
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
		}

		b.ResetTimer()
		for n := 0; n < b.N; n++ {
			res, err := ToArray(input)
			if err != nil {
				b.Errorf("unexpected error: %v", err)
				break
			}

			// validate results
			b.StopTimer()
			if !reflect.DeepEqual(wants, res) {
				b.Errorf("unexpected output error: wanted %v %T; got %v %T", wants, wants, res, res)
				return
			}
			b.StartTimer()
		}
	})

	b.Run("CopyElementsToArray256Elem", func(b *testing.B) {
		input := []int{
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
		}
		wants := [256]int{
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
			1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32,
		}

		var conv = func(source []int) (any, error) {
			const size = 256
			dest := [size]int{}
			copy(dest[:], source)
			return dest, nil
		}

		b.ResetTimer()
		for n := 0; n < b.N; n++ {
			res, err := conv(input)
			if err != nil {
				b.Errorf("unexpected error: %v", err)
				break
			}

			// validate results
			b.StopTimer()
			if !reflect.DeepEqual(wants, res) {
				b.Errorf("unexpected output error: wanted %v %T; got %v %T", wants, wants, res, res)
				return
			}
			b.StartTimer()
		}
	})
}
```

These tests are performing the same operations as discussed in this document, with a target slice of 256 elements. Running these tests shows the following results:

> formatted with [`prettybench`](https://github.com/cespare/prettybench)

```
Running tool: /usr/bin/go test -benchmem -run=^$ -coverprofile=/tmp/vscode-goXlpszH/go-code-cover -bench ^BenchmarkToArray$ github.com/zalgonoise/x/ptr

goos: linux
goarch: amd64
pkg: github.com/zalgonoise/x/ptr
cpu: AMD Ryzen 3 PRO 3300U w/ Radeon Vega Mobile Gfx
PASS
coverage: 6.4% of statements
benchmark                                          iter       time/iter   bytes alloc        allocs
---------                                          ----       ---------   -----------        ------
BenchmarkToArray/PointerConversion256Elems-4    1271818    979.30 ns/op     2048 B/op   1 allocs/op
BenchmarkToArray/CopyElementsToArray256Elem-4    992716   1221.00 ns/op     2048 B/op   1 allocs/op
ok      github.com/zalgonoise/x/ptr     102.362s
```

These results show a solid 19%~20% gain in performance with no catches -- provided you have the capacity constant under control. No changes in amount of allocations, no changes in number of allocated bytes.

## Conclusion

It was a fantastic way of looking into Go slices in the context of a data structure -- not as a collection of items. A great opportunity to explore how manipulating the *backing array* of a slice affects the slice, when `copy()` is used and when it isn't. An amazing chance of working with the unsafe package and the opportunity of exploring memory addresses.

In reality, this approach would *maybe* be OK if you had a (really) large data collection in a slice, and *needed* to work with it as an array. Because of the performance gains, it *could* be a scenario where this approach is considered. For an everyday application: no way :) It's important to keep in mind the readability and maintainability aspect of the product. Performance gains are welcome provided they don't hinder the ability of reading and maintaining your code base.
