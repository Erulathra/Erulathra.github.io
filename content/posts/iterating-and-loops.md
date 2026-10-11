+++
title = "It's not a loop, it's a spiral."
date = "2026-10-10T17:32:44+02:00"
author = "Erulathra"
tags = ["performance", "low-level", "C++"]
keywords = ["bitset", "dirty-flags", "low-level", "loops", "for"]
showFullContent = false
hideComments = false
+++

{{< figure
  src="images/posts/iterating-and-loops/flashlight.jpg"
  caption="Silhouette with Flashlight in Foggy Night Scene, Jan Reichelt, 2025"
>}}

## Lambdas are ugly

> This post is a sequel to [A bit(set) of knowledge.](posts/bitsets) post.
You don't need to read it to understand this one, but I highly encourage you
to do it.

I am not a big fan of `C++` lambdas. I am 100% sure that they are a pretty handy
language feature, but using them in the templates is a headache. Forget an
ampersand or const, and you get a magic incantation in the compiler output[^0].

{{< figure
  src="images/posts/iterating-and-loops/template-madness.png"
  caption="Actually, this error isn't so long but still unhelpful."
>}}

From the other side, I love range-based for loops. They improve the readability
of the code, and reduce its buggability. For this reason when I worked with
bitsets (described in the previous post), I came up with the idea that I can
implement an iterator and a helper struct which would replace this:

```cpp
bitArray.IterateOverEnabledBits(
	[&](u32 index)
	{
		forEachResult.Push(index);
	}
);
```

with this:

```cpp
for (u32 index : EnabledBitsRange(bitArray))
{
	rangeResult.Push(index);
}
```

But, what performance implications does this approach have? I can't justify an application
slowdown due to only my personal preference. On the other hand, maybe my first
intuitive approach with lambda was the slowest one? We can't answer it without
tests, so just like in my previous post I decided to write a benchmark.

## Iterator, Range and the Lambda

Before we look into tests, we should discuss how the three major approaches
work behind the scenes. The entire implementation of all data structures is in
the [MicroBenchmarks](https://github.com/Erulathra/MicroBenchmarks) repo.

### Iterator

Let's start from the easiest one, the iterator. Every `C++` programmer should be
familiar with this topic. In a nutshell, this is an object that simplifies iterating
through the data structure. Implementation of the iterator is highly dependent
on the data, so the performance might vary. In practice, we should implement
at least `operator++` which moves iterator to the next element and `operator==`, which
would compare our iterator with the iterator that points at the end[^2].

In the benchmark I implemented two iterators. The trivial one used in
`TStaticStack` (which is in fact a static array with an integer counter which
points at the top of the stack) is just a type alias to the element pointer. The
`begin()` returns pointer at the start of the array, and the `end()` returns
`begin() + mSize`.

```cpp
// Part of `Common/Stack.h`
template<typename T>
struct TStaticStack
{
	using Iterator = T*;

	T mData[capacity];
	u32 mSize;
	//(...)
};
```

A *bit* more complicated one is the `EnabledBitsIterator`, which looks like this:

```cpp
// Part of `Common/Stack.h`
struct EnabledBitsIterator
{
	/* Data */
	BitArray* mBitArray = nullptr;
	u32 mWordIndex = 0;
	u32 mWord = 0;
	u32 mCurrentIndex = 0;

	/* Interface */
	explicit EnabledBitsIterator(BitArray& bitArray)
	{
		// Jump to first set bit
		++(*this);
	}

	static EnabledBitsIterator CreateEnd(BitArray& bitArray) { /* (...) */ }

	u32 operator*() const
	{
		return mCurrentIndex;
	}

	// (...)

	EnabledBitsIterator& operator++()
	{
		while (mWordIndex < mBitArray->mNumWords)
		{
			if (mWord != 0)
			{
				const u32 temp = mWord & u32(-i32(mWord));
				const u32 trailingZeros = CountTrailingZeros(mWord);

				mWord ^= temp;

				// We found one, stop here.
				mCurrentIndex = mWordIndex * 32 + trailingZeros;
				return *this;
			}

			mWordIndex++;
			mWord = mBitArray->mData[mWordIndex];
		}

		return *this;
	}
};
```

The implementation of searching for the next enabled bit is analogous to the
foreach implementation form the previous post, the only difference is fact that
we need to store on which index we stopped searching. This is because we need to
disable current bit just after finding it to ensure that we don't iterate over
the same bits twice.

> This implementation is 100% my invention, so I suppose that there is a more clever
one. If you find it, please share it with me. However this one would be totally
sufficient for our benchmark.

### Range

The [reference](https://en.cppreference.com/cpp/language/range-for)
says[^reference] that to utilize range-based for loops, your type needs to
implement `begin()` and `end()` functions. So in practice this snippet:

```cpp
for (u32 i : array)
{
	// (...)
}
```

would be transpiled[^1] to:

```cpp
for (It begin = array.begin(), It end = array.end(); begin != end; ++begin)
{
	u32 i = *begin;
}
```

For-each[^forEach] is quite useful and it's obvious that we should implement it
for every most of the data structures, but a less common fact is that we can create
helper structs that allow us to iterate things in different ways that default.
The struct would look like:

```cpp
struct SomeNotObviousRange
{
	It begin();
	It end();
};

// Usage
for (Foo bar : SomeNotObviousRange(dataStructure))
{
	// (...)
}
```

I first saw that trick in the UE[^UERange] code base, and from that time I love
it. You can create multiple different ranges for different types of iterators.

### Lambda

Lambdas in C++ are quite powerful,
but *with great power, comes great responsibility*, which in anonymous functions
is **capture block**. Let's take this simple snippet of code:

```cpp
// Assume that a and b are u32.
auto sumLambda = [a, b]()
{
	return a + b;
}
```

If I transpile this snippet to *orthodox* `C++` it would look like this:

```cpp
struct SumFunctor 
{
	u32 mArgA;
	u32 mArgB;

	u32 operator() () const
	{
		return a + b;
	}
};

auto sumFunctor = SumFunctor{a, b};
```

Quite easy right? However compilers are quite smart, and implement many
handy optimizations. In reality if the lifetime of the lambda is trivial[^3],
the compiler would inline it, so there wouldn't be any performance implications
related to lambda execution.

Take a look at the captures. They are just fields of the functor, so we need
to copy them, as the compiler optimizations can't change the outcomes of the program.
For this reason, when we capture something complex, the lambda might be left unoptimized,
this is especially dangerous when we are capturing `=`.

> For now, this paragraph might look abstract from the topic, but trust me, all
evidence would take part in the *grand finale*.

## Benchmark

Ok, so after this loooong introduction we can get to the point. The whole benchmark
is implemented in [IteratorsBench.cpp](https://github.com/Erulathra/MicroBenchmarks/blob/master/IteratorsBench.cpp).
There are two major tests. In the first one we would measure performance of
iterating over enabled bits using:

* for-each template
* iterator
* range based for loop

The second one uses `TStaticStack` (array with additional steps). We would
iterate over it using:

* traditional for i loop
* for-each template
* for-each with std::function as parameter
* iterator
* range based for loop

## Results aka *The Grand Finale*

I ran benchmark on my personal workstation with `Ryzen 7 7700`.
I used `clang 23.1.1` with `O3` and `AVX2` flags enabled. Time was measured
in perf-counter cycles (`3.8GHz` on my CPU). The results are presented in the
table below:

| Approach                 | Array         | Enabled Bits |
|:------------------------:|:-------------:|:------------:|
| for-i                    | 6509.268494   | N/a          |
| for-each template        | 6186.908051   | 65927.032410 |
| for-each `std::function` | 281777.576111 | N/a          |
| iterator                 | 5944.576294   | 71173.477570 |
| range based for loop     | 5936.717224   | 69270.821930 |

### Array

Let's start from the more common approach. It turns out that despite
`std::function`, the *orthodox* for-i loop is the slowest one. It's due to the
fact that this approach requires to calculate pointer offset during each
iteration (I was really surprised that compiler didn't optimize that.) instead
of just incrementing the pointer like in the iterator and range-based loop
approach. Calling a function in the `for-each` approach also impacted iteration
time a bit. However those 4 approaches are very comparable, and the performance
impact would be negligible in real scenarios.

The more interesting one is `std::function`. Why it is about 40 times slower than
other approaches?

The short answer is:

> I used wrong tool, to solve my problem.

The longer one requires us to recap how the `std::function` works. Remember that
the lambda is in fact a functor? `std::function` is a construct that allows us to
store that functor in a generous way. We don't have to think what captures would
be in lambda or how many would be. This construct uses a **type erasure** pattern
to hide from the rest of the code its implementation details. Unfortunately,
it does it in most unperformant way. It stores our functor on the heap. Additionally
this type of lifetime blocks compiler from inlining our lambda.

So remember:

> Never use `std::function` in tight loops.

(Or at all, there are many more performant way to store callbacks. But this is a
topic for a different post[^UEDelegates].)

### Bitset

So after the benchmark, it appears that the most performant way to iterate over
a bitset is...

still a `for-each` template. Not a result I wanted :(. Still, if you don't iterate
over `2^16` bits, you can use iterator or range based for loop.

The interesting fact is that range-for was faster than iterators, despite the
fact that this approach uses the same iterator struct. I checked assembly and
actually there are fewer conditionals and fewer jumps there, but I didn't find
any information why compiler decides to optimize that only in this approach. So,
unfortunately, this is still a mystery for me. If you know why this happens, you
know where to find me me. In practice, I would not treat this as a pattern, you
should always test performance of your code, not just trust random guy from the
internet ;).

# Conclusions

![Boxer Meme](images/posts/iterating-and-loops/boxer-meme.jpg)

WOW, what a long post. I didn't expect that would take so many coffee cups to
finish it.

Return to the topic. I think that I would embrace `EnabledBitsRange` for
smaller bitsets. This is 100% personal preference but I find the range fors are
much cleaner. At the end I still need to profile my code, and switching from
range for to `for-each` isn't a big deal.

Next post would be (probably) shorter. We will talk about how to precisely
measure time in terms of performance. This is not a big topic, but I have used
that knowledge in two posts, so I think I should explain how I do it.

See, you next time.

[^0]: There is only one worse type of error than template one - The linker error.
[^1]: Source-to-source compilation is named transpilation. [Source.](https://en.wikipedia.org/wiki/Source-to-source_compiler)
[^2]: In fact this is only (I would say) a `C++` tradition. You just need
to implement function that moves iterator forward (or backward), and the
functionality that informs that iterator is at the end.
[^3]: For example, it doesn't go beyond the current stack frame.
[^reference]: I wanted to cite a reference here, but the `C++` reference is sometimes
so engineerish and fails to describe how things work in practice.
[^forEach]: I know that is not a correct name for that construct but typing
*range based for loop* each time hurts my fingers.
[^UERange]: Check `TActorRange` or `TObjectRange`.
[^UEDelegates]: UE's delegates are much more clever way to approach this problem.
If you are UE dev and you still want to store callback like this, you should
always use delegates instead `TFunction`.
