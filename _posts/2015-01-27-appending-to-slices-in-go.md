---
layout: post
title: Appending to slices in Go
redirect_from:
 - /2014/01/27/appending-to-slices-in-go/
---

<a href="https://www.flickr.com/photos/biketourist/135979696" title="Sliced 77 Honda Wagon by John, on Flickr"><img src="https://farm1.staticflickr.com/46/135979696_b2322d43b9.jpg" width="500" height="375" alt="Sliced 77 Honda Wagon"></a>

Using `append()` to extend Go slices is tricky, especially when compared to analogous operations on Java/C++ vectors or Ruby/Python/JavaScript arrays. Most of the difficulty comes from the fact that `append()` sometimes creates new buffers, and sometimes it doesn't.

In the following example, we create a slice, assign it to a second variable, and use `append()` on both slices:

{% highlight go %}
a := []int{0}
b := a

fmt.Println(a) // [0]
fmt.Println(b) // [0]

a = append(a, 1)
b = append(b, 2)

fmt.Println(a) // [0 1]
fmt.Println(b) // [0 2]
{% endhighlight %}

Following the calls to `append()`, `a` and `b` can be treated as entirely separate slices. This is fairly intuitive.

However, by tweaking the state of the initial slice, we can create entirely different behavior:

{% highlight go %}
a := make([]int, 1, 2) // same content and length, but larger capacity
b := a

fmt.Println(a) // [0]
fmt.Println(b) // [0]

a = append(a, 1)
b = append(b, 2)

fmt.Println(a) // [0 2] <- New behavior!
fmt.Println(b) // [0 2]
{% endhighlight %}

A few things to note here:

1. The code is virtually the same. Only the first line has changed.
2. The content of the original slice is the same. Only its capacity has changed.
2. We have a different outcome: following the calls to `append()`, `a` and `b` continue to refer to the same underlying buffer.

In both examples, the assignment expression `b := a` creates a second [slice header](http://golang.org/pkg/reflect/#SliceHeader), one that happens to have the same buffer pointer, length, and capacity as the first. As long as neither `a` nor `b` is reassigned, they are effectively aliases of the same slice. However, if either or both are reassigned in an append expression such as `a = append(a, 1)`, then they may or may not continue to refer to the same buffer.

A [slice's capacity will determine](http://blog.golang.org/go-slices-usage-and-internals) whether or not `append()` will create a new buffer. In our trivial example, slice capacity is immediately apparent, but that won't necessarily be the case in more complex code. You might not know where your slices are coming from--they may arrive at your code as arbitrary function arguments, or they may be created based on user input or some other external state. You also can't predict a slice's capacity--it  can diverge from the length of the slice either explicitly (calling `make()` with a specific length and capacity) or implicitly (via `append()` and subslicing).

<a href="https://www.flickr.com/photos/matilda89/6902196810" title="Sliced but whole together by matilda89, on Flickr"><img src="https://farm8.staticflickr.com/7238/6902196810_e80e426d33.jpg" width="500" height="332" alt="Sliced but whole together"></a>

Unless you write explicit logic that reads the capacity of a slice, it is not really clear at runtime whether `append()` will retain the underlying buffer or create a new one. In other words, if you assign a slice to a second variable, and then `append()` to one of them, it's not clear whether your two slices will still refer to the same memory.

The ambiguity here is bad. Direct assignment of slice variables, such as `b := a` in the previous examples, starts to seem like a code smell, and it's tempting to avoid it altogether. But there are other ways in which a slice can end up referencing the same buffer as another slice.

For example, through subslicing:

{% highlight go %}
a := make([]int, 1, 2)
b := a[0:1] // subslices always refer to the same buffer
{% endhighlight %}

Or re-assigning the result of an `append()`:

{% highlight go %}
a := make([]int, 1, 2)
b := append(a, 1) // in this case, append() preserves the same buffer
{% endhighlight %}

Or, most treacherously, obscuring the same operations with functions, whose internal behavior may or may not be immediately apparent:

{% highlight go %}
foobar := func(s []int) []int {
	return append(s, 0)
}

a := make([]int, 1, 2)
b := foobar(a) // in this case, foobar() preserves the same buffer
{% endhighlight %}

One compelling rule of thumb that emerges is to **always make full copies of slices when your code really means to use copies**. Copying eliminates any ambiguity as to whether two slices refer to the same memory, because they simply won't.

There are two bits of unpleasantness here, one small and one big. The small one is that copying is ugly in Go:

```
// Go
b := append([]int{}, a...)

// JavaScript
b = a.slice()

# Python
b = list(a)

# Ruby
b = a.dup

// Java
Vector<int> b = (Vector<int>)a.clone();

// C++
std::vector<int> b = a;
```

Go's slice-copy seems like an idiom when a primitive is more appropriate (and to be fair, JavaScript is in the same boat). Even crusty old C++'s syntax is preferable, since it appears to say what it means.

<a href="https://www.flickr.com/photos/derodeolifant/15766606252" title="Mirror-fish (Herman van Goubergen) by Marjan Smeijsters, on Flickr"><img src="https://farm4.staticflickr.com/3938/15766606252_f6ef350288.jpg" width="500" height="333" alt="Mirror-fish (Herman van Goubergen)"></a>

The bigger issue is that it's not always clear when you "mean" to use a copy. Consider a function `join` that combines a prefix with a list of suffixes:

{% highlight go %}
func join(prefix []int, suffixes [][]int) [][]int {
	res := [][]int{}
	for _, s := range suffixes {
		res = append(res, append(prefix, s...))
	}
	return res
}
{% endhighlight %}

This code is nice and clean. It is also incorrect. Check out what happens when you run it on slices of  identical content but varying capacity:

{% highlight go %}
suffixes := [][]int{
	{1},
	{2, 3},
}

fmt.Println(join([]int{0}, suffixes))          // [[0 1] [0 2 3]]
fmt.Println(join(make([]int, 1, 2), suffixes)) // [[0 1] [0 2 3]]
fmt.Println(join(make([]int, 1, 3), suffixes)) // [[0 2] [0 2 3]] WRONG!
{% endhighlight %}

The key insight here is that `append(prefix, s...)` does not consistently generate new buffers when the slice capacity varies, which can lead to aliasing and memory clobbering when you didn't want it.

Since we can't rely on a consistently-behaving `append()`, the appropriate thing to do is to make explicit copies, which ensures that we're operating on fresh, non-aliased buffers:

{% highlight go %}
func join(prefix []int, suffixes [][]int) [][]int {
	res := [][]int{}
	for _, s := range suffixes {
		prefixCopy := append([]int{}, prefix...) // explicit copy
		res = append(res, append(prefixCopy, s...))
	}
	return res
}
{% endhighlight %}

From this, we get what we expected:

{% highlight go %}
suffixes := [][]int{
	{1},
	{2, 3},
}

fmt.Println(join([]int{0}, suffixes))          // [[0 1] [0 2 3]]
fmt.Println(join(make([]int, 1, 2), suffixes)) // [[0 1] [0 2 3]]
fmt.Println(join(make([]int, 1, 3), suffixes)) // [[0 1] [0 2 3]] fixed!
{% endhighlight %}

I find the corrected version to be significantly uglier than the original, and not just ugly, but unpleasantly subtle, in a way that feels defensive rather than expressive. And yet when you get used to it, you kind of just get it: the Go slice design seems to be a best-of-evils compromise between high-level convenience (try doing a re-allocating append operation in pure C) and more low-level performance concerns (try doing memory-efficient subslicing on huge Ruby arrays).

As far as list abstractions go, Go slices are pretty leaky, and are probably best thought of as a thin film of syntactic sugar over a memory buffer. In other words, you really should know what's going on internally before using them. Go's own documentation admits to as much, and the official Go blog's ["Go slices: usage and internals"](http://blog.golang.org/go-slices-usage-and-internals) ought to be considered required reading for anyone using the language.