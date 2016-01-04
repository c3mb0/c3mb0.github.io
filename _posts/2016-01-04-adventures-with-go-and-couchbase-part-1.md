---
layout:     post
title:      Adventures with Go and Couchbase, Part 1
date:       2016-01-04
summary:    Join me while I build a completely useless and highly educational piece of service.
categories: golang couchbase
---
It's no secret I love Team Fortress 2 (4k hours and proud, baby). However, I've recently decided to spend my precious free time in a more productive manner, so I just hack the Source Engine instead. Between C++, DLL injections, virtual function tables and signature scanning, my love for lower level programming has rekindled, so I've decided to pick up a project in Go.

## Why Go?

I've been experimenting with a few languages and frameworks for almost a year now, namely Elixir (Phoenix), JavaScript (Node.js) and Go (Revel). Amongst them, Go made me feel all warm and fuzzy inside. Here are a couple of reasons why:

  * Statically typed - Being able to compile the code you've written is half way to success
  * Multiple function returns - You no longer have to write a class to encapsulate return values from a function
  * Package management - _go get_ any package easily
  * Natively compiled
  * Designed with concurrency in mind - I did not have the opportunity to leverage this feature yet, but it's nice to know it's there
  * Super easy deployment - Just drag and drop the binary
  * Feature-rich built-in libraries with amazing documentation
  * Great editor support - I'm using the _go-plus_ package for Atom and I highly recommend it
  * Gives you a C/C++ vibe, and it has garbage collection!

Without further ado, let's dive in!

## Concepts in Go for Stupid People

One of the main reasons why I'm writing this post is to have a reference document for the future Me, for whom I will hopefully save a ton of time looking at the screen, feeling like a stupid person. Below are the concepts that took me a while to understand.

### Slices

You will see the word _slice_ all over the internet if you're searching something about Go. Then, if you're like me, will think to yourself, "Isn't that just an array? Why call it something fancier?". Well, no, slices are _not_ arrays, and I will try to explain the difference in terms of C.

In C, an array is represented through a pointer that points to the base address of the block of allocated memory for the said array. For example:

{% highlight c %}
int sample[50];
// synonymous with
int *sample;
sample = (int *)malloc(sizeof(int) * 50);
{% endhighlight %}

Think of a slice in Go as a _struct_. Also, Go uses the same pointer arithmetic as C. Considering these, here are the representative internals of an _int_ slice and how we would get the slice equivalent of _sample_ in Go ([source](https://www.reddit.com/r/golang/comments/283vpk/help_with_slices_and_passbyreference/ci79fla)):

{% highlight c %}
// type []int struct {
//     data* int -> this is the int *sample from above
//     len int
//     cap int
// }
sample := make([]int, 50)
{% endhighlight %}

Here, you might have your first "A-ha!" moment like I did.

So what is the use of this additional level of abstraction provided by Go? From what I understand, it basically frees the developer from juggling pointers. If you were to pass an array by reference in C, you would do:

{% highlight c %}
int main(int argc, char *argv[]) {
    int *sample;
    sample = (int *)malloc(sizeof(int) * 50);
    foo(sample);
    // sample[0] is now 7
}

void foo(int *reference) {
    reference[0] = 7;
}
{% endhighlight %}

In Go, you can just do:

{% highlight c %}
func main() {
    sample := make([]int, 50)
    foo(sample)
    // sample[0] is now 7
}

func foo(value []int) {
    value[0] = 7
}
{% endhighlight %}

This is because even though the _sample_ slice is passed by value, when you assign _sample_ to _value_, their internals are assigned to each other, meaning that their __base address pointers are now equal__. Thus, whatever changes you make to the _value_ slice will affect the values in the _sample_ slice.

But it does not end here. Below is an equivalent Go function as the one above:

{% highlight c %}
func main() {
    sample := make([]int, 50)
    foo(&sample)
    // sample[0] is now 7
}

func foo(reference *[]int) {
    sample := *reference
    sample[0] = 7
}
{% endhighlight %}

Confused yet? If they are the same, then why would you ever want to use the second version of _foo_? It reintroduces pointer arithmetics, the very thing we are trying to avoid! Hold on to your butts as I try to clear things up with some code.

Both _foo_ approaches work just fine when we want to retrieve and change the slice's values. Since the first one is simpler and clearer, it should be the one to use for mentioned purposes. What if we wanted to change the array that the slice points to instead? For example, what if we wanted to truncate the last 20 elements of _sample_ and shrink the slice size to 30? Let's give it a try with the pass-by-value approach:

{% highlight c %}
func main() {
    sample := make([]int, 50)
    // len(sample) is 50
    foo(sample)
    // len(sample) is still 50!
}

func foo(value []int) {
    value = value[:30]
    // len(value) is 30
}
{% endhighlight %}

Uh-oh! This did not work because the __local__ slice _value_ was reassigned to a new slice of size 30 that __still points to the base address of _sample___, which does not affect the size our original _sample_ slice. Let's take things a step further with the following:

{% highlight c %}
func main() {
    sample := make([]int, 50)
    foo(sample)
    // len(sample) is still 50
    // sample[0] is now 7!
}

func foo(value []int) {
    value = value[:30]
    // len(value) is 30
    value[0] = 7
}
{% endhighlight %}

Makes sense?

Since pass-by-value did not work for truncating the slice, I assume you can already guess that pass-by-reference is the correct approach for the task. Let's have a look:

{% highlight c %}
func main() {
    sample := make([]int, 50)
    // len(sample) is 50
    foo(&sample)
    // len(sample) is now 30!
}

func foo(reference *[]int) {
    sample := *reference
    *reference = sample[:30]
}
{% endhighlight %}

In short, if you are __changing the structure of the underlying array within the slice__, you have to __pass the slice by reference__. If you are __merely accessing and modifying the values of the underlying array within the slice__, you should be __passing the slice by value__ for the sake of simplicity.

_For Part 2, I will be writing about deferring in Go and how I got started with Couchbase. Did I get something wrong? Got nice/bad things to say? [Let me know!](http://c3mb0.github.io/contact/)_
