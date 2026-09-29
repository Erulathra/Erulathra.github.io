+++
title = "A bit(set) of knowledge. [Part 1]"
date = "2026-09-28T19:46:39+02:00"
author = "Erulathra"
tags = ["bitset", "dirty-flags", "low-level", "performance"]
keywords = ["bitset", "dirty-flags"]
cover = "images/posts/bitset/notepad.jpg"
showFullContent = false
hideComments = false
+++

## An anecdote

Some time ago, when I was creating a interactive tool for artists, I encontered
an issue that requires me to iterate very large array of objects.
The array contains duplicates so additionally I needed to mark object as processed
to save artists' precious time.

The first thing that came to my mind was a `HashSet`. That solution worked nice,
but the higher number of objects makes the Unreal's builtin `THashSet` too slow
to operate in real time.

This was due to few reasons:

* The `HashSet` needed to be relocated during iteration.
* The calculation of the hashes were to slow.
* The implementation of `HashSet` requires a critical section during adding items.

After some profiling, I had got a great idea: What if I just use flags instead of
`hashset`? After some digging, I found that UE has a `TBitArray` container, that
ideally solves my problem. All of my objects have unique 32bit sequential ids, so
everything I needed was just enable bit with index the same as the id of the object.
If the bit is enabled it means that my object was processed.

> Everything works fine? It's Counter-Strike time?[^1]\
> Answer is: no.

We now need to iterate over bits, but how do it efficiently?

## FastIterateOverTrueBits()

The naïve approach requires only bit shift operator. Just iterate over
indexes from 0 to bitset size. This solves our issue, but we can do it much faster.

Our precious `x86_64` processors (today even more precious), have an interesting
bit manipulation instruction that can speed up our processing[^2] up to 32 times.
This instruction is `bsf` (bit scan forward) which returns the index of the
first disabled bit from the least significant one[^3].

So we have all knowledge now, and we can write the function bellow.

```cpp
	void IterateOverEnabledBits(u32* data, u32 numBits) const
	{
		for (u32 dataIndex = 0; dataIndex < mNumWords; ++dataIndex)
		{
			u32 dataValue = data[dataIndex];
			while (dataValue != 0)
			{
				// NOTE: The union of positive and negated int bellow is used to
				//    find lowest bit. It can be replaced with std::countr_one
				const u32 temp = dataValue & u32(-i32(dataValue));
				// NOTE: Function `CountTrailingZeros` calls `bsf` internally.
				const u32 trailingZeros = CountTrailingZeros(dataValue);
				const u32 bitIndex = dataIndex * 32 + trailingZeros;

				if (bitIndex < numBits)
				{
 					// Call your callback here.
				}
				dataValue ^= temp;
			}
		}
	}
```

Not so complicated, right?

## Benchmark

But how fast is it? I prepared a [micro benchmark](https://github.com/Erulathra/MicroBenchmarks/blob/master/BitsetBench.cpp) :sunglasses:.
The benchmark runs 4 tests twice. Once for the sequnetial data, and once for the
random accessed data (a common scenario in OOP world).

The first test, is an naïve approach. It loads each object and checks the flag stored
in it. The second one, uses our approach to iteration over a bitset. The third and
fourth, are quite similar. They are testing performance for iterating array of indexes.
The third one uses an sorted array, and the forth one shuflles this array to simulate
_random_ additions during runtime.

The results are on the two plots bellow. Both of them shows a time measured in processor's
time-stamp counter tics[^5], so the lower numbers are better[^4]. The `x` axis tells,
how many flags were enabled.

![Triangle](images/posts/bitset/plot_sequential.png)
![Triangle](images/posts/bitset/plot_shuffled.png)

# Results

So the iterating using sorted indexes was generally the fastest one, next we have
bitset, iterating over the shuffled indexes and the naïve approach. So why should
I use bit set instead of sorted array?

There are three reasons:

* Bit set is always sorted. Maintenance of the sorted array isn't
always easy, so in most of the circumstances we end up with unsorted array,
and sorting it would be probably very expensive.
* The feature that the bitset gets its name. Bitset is a **set**,
so you never get duplicates in your container.
* GPU loves bitsets. Bitsets are trivial to implement in shaders.
* A the last one (for me the the most important one) is the fact that our data is
tightly packed. We can store `512` flags on the one `64` byte cache line. This
can greatly reduce memory throughput.

## Conclusions

Of course **you should always use a proper algorithm to your problem**. The bitset
isn't a silver bullet and wouldn't solve all your problems. However I think that
this quite old data structure is underestimated today (at least I never heard of
them during my CS studies), so i decided to dedicate a few blogposts.
Yes, **a few** because this is not all what I have to tell You in this topic.

Next time we would benchmark different ways to iterate over bitsets from the
language perspective. I am really excited because this is what I wanted to
test myself for the long time.

So, see You soon!

[^1]: A bad (and quite old :older-man:) polish meme.

[^2]: The fast iteration algorithm was taken from [this](https://lemire.me/blog/2018/02/21/iterating-over-set-bits-quickly/)
    great blog post.

[^3]: C++ provides an easy and (mostly) portable way to generate this instruction
    without using assembly or intristics. Check `std::countr_zero` function falmily.

[^4]: The benchmark was build using `LLVM 22.1.8` with level 2 optimizations and
    AVX2 instruction set. After compiling the benchmark was run on the Ryzen 7 7700,
    so the time-stamp counter frequency is `3.8GHz`.

[^5]: A [Nice video](https://youtu.be/tAcUIEoy2Yk) by Casey Muratori about performance counters.
