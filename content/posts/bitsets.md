+++
title = "A bit(set) of knowledge."
date = "2026-09-28T19:46:39+02:00"
author = "Erulathra"
tags = ["performance", "low-level", "C++"]
keywords = ["bitset", "dirty-flags", "low-level", "performance"]
showFullContent = false
hideComments = false
+++

![Notepad](images/posts/bitset/notepad.png)

## An anecdote

Some time ago, when I was creating an interactive tool for artists, I encountered
an issue that requires me to iterate over a very large array of objects.
The array contains duplicates so additionally I needed to mark object as processed
to save artists' precious time.

The first thing that came to my mind was a hash set. That solution worked nice,
but the higher number of objects makes the Unreal's builtin `THashSet` too slow
to operate in real time.

This was due to a few reasons:

* The hash set needed to be relocated during iteration.
* The calculation of the hashes was too slow.
* The implementation of hash set requires a critical section during adding items.

After some profiling, I had got a great idea: What if I just use flags instead of
hash set? After some digging, I found that UE has a `TBitArray` container, that
ideally solves my problem. All of my objects have unique 32-bit sequential ids, so
everything I needed, was just enabling a bit with index the same as the id of
the object. If the bit is enabled, it means that my object was processed.

> Everything works fine? It's Counter-Strike time?[^1]\
> Answer is: no.

Now, we need to iterate over bits, but how to do it efficiently?

## FastIterateOverTrueBits()

Our precious `x86_64` processors (today even more precious), have an interesting
bit manipulation instruction that can speed up our processing[^2] up to 32 times.
This instruction is `bsf` (bit scan forward), and it returns the index of the
first disabled bit from the least significant one[^3].

We have all required knowledge now, and we can write the function below.

```cpp
	void IterateOverEnabledBits(u32* data, u32 numBits) const
	{
		for (u32 dataIndex = 0; dataIndex < mNumWords; ++dataIndex)
		{
			u32 dataValue = data[dataIndex];
			while (dataValue != 0)
			{
				// NOTE: The union of positive and negated int below is used to
				//    find lowest bit. It can be replaced with std::countr_one
				const u32 temp = dataValue & u32(-i32(dataValue));
				// NOTE: Function `CountTrailingZeros` calls `bsf` internally.
				const u32 trailingZeros = CountTrailingZeros(dataValue);
				const u32 bitIndex = dataIndex * 32 + trailingZeros;

				if (bitIndex < numBits)
				{
 					Callback(bitIndex); // Your callback/code
				}
				dataValue ^= temp;
			}
		}
	}
```

Not so complicated, right?

## Benchmark

But how fast is it?

I prepared a [micro benchmark](https://github.com/Erulathra/MicroBenchmarks/blob/master/BitsetBench.cpp) :sunglasses:.
The benchmark runs 4 tests twice. Once for the sequential data, and once for the
random accessed data (a common scenario in OOP world).

The first test, is a naïve approach. It loads each object and checks the flag stored
in it. The second one, uses our approach to iterate over a bitset. The third and
fourth, are quite similar. They are testing performance for iterating over an array
of indexes. The third one uses a sorted array, and the fourth one shuffles this
array to simulate _random_ additions during runtime.

The results are on the two plots below. Both of them show a time measured in processor's
time-stamp counter tics[^5], so the lower numbers are better[^4]. The `x` axis tells,
how many flags were enabled.

![Triangle](images/posts/bitset/plot_sequential.png)
![Triangle](images/posts/bitset/plot_shuffled.png)

# Results

So iterating using sorted indexes was generally the fastest one, next we have
bitset, iterating over the shuffled indexes and the naïve approach. So why should
I use bit set instead of sorted array?

There are few reasons:

* Bit set is always sorted. Maintenance of the sorted array isn't
always easy, so in most of the circumstances we end up with unsorted array,
and sorting it would be probably very expensive.
* The feature that the bitset gets its name. Bitset is a **set**,
so you never get duplicates in your container, and checking a particular
bit has always `O(n)` complexity.
* The last one (for me the most important one) is the fact that our data is
tightly packed. We can store `512` flags on the one `64` byte cache line. This
can greatly reduce memory throughput.

## Conclusions

Of course **You should always use a proper algorithm for your problem**. The bitset
isn't a silver bullet and wouldn't be a perfect solution for every case.
However I think that this quite old data structure is underestimated today
(at least I had never heard of them during my CS studies), so I decided to dedicate
a few blogposts. Yes, **a few** because this is not all what I have to tell you
on this topic.

Next time we will benchmark different ways to iterate over bitsets from the
language perspective. I am really excited because this is what I wanted to
test myself for a long time.

So, see You soon!

[^1]: A bad (and quite old) polish meme.

[^2]: The fast iteration algorithm was taken from [this](https://lemire.me/blog/2018/02/21/iterating-over-set-bits-quickly/)
    great blog post.

[^3]: C++ provides an easy and (mostly) portable way to generate this instruction
    without using assembly or intrinsics. Check `std::countr_zero` function family.

[^4]: The benchmark was built using `LLVM 22.1.8` with level 2 optimizations and
    `AVX2` instruction set. After compiling the benchmark, it was run on the
    Ryzen 7 7700, so the time-stamp counter frequency is `3.8GHz`.

[^5]: A [Nice video](https://youtu.be/tAcUIEoy2Yk) by Casey Muratori about performance counters.
