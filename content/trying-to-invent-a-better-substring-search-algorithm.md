+++
title = "Trying to invent a better substring search algorithm"
date = 2023-10-17

[taxonomies]
tags = ["rust", "algorithms"]
+++

I failed to create something new, but I'll share my thought process, and how it almost led me to reinventing a well-known solution.

# Table of contents

- [Prelude](#prelude).
- [The naive implementation](#the-naive-implementation).
- [The thought process](#the-thought-process).
- [The algorithm](#the-algorithm).
- [The Karp, and the Rabin](#the-karp-and-the-rabin).
- [Closing thoughts](#closing-thoughts).
- [Homework](#homework).
- [Extras](#extras):
    - [Can the algorithm overflow?](#can-the-algorithm-overflow)
    - [XOR variation](#xor-variation).
    - [Inputs that trigger the worst-case performance](#inputs-that-trigger-the-worst-case-performance).
    - [Fun results for tiny inputs](#fun-results-for-tiny-inputs).
    - [Tiny puzzle](#tiny-puzzle).
    - [Code for the `needle` crate](#code-for-the-needle-crate).
    - [Comment section](#comment-section).

# Prelude

```Python
needle = 'needle'
haystack = 'do I have a needle? Yes I do'

assert(needle in haystack)
```

<!-- more -->

I was once told that Python [uses] the [Boyer–Moore algorithm] (created in 1977, optimized in 1979) to search for substrings.

[uses]: https://github.com/python/cpython/blob/75cd86599bad05cb372aed9fccc3ff884cd38b70/Objects/stringlib/fastsearch.h#L5-L8
[Boyer–Moore]: https://en.wikipedia.org/wiki/Boyer-Moore_string-search_algorithm
[Boyer–Moore algorithm]: https://en.wikipedia.org/wiki/Boyer-Moore_string-search_algorithm

I briefly read about it, here's a summary: You preprocess the needle creating a lookup matrix, then, you use it to skip a lot of checks.

Uh... ok, cool.

-- _3 years passed by_ --

Can't we speed this up in a simpler way?

# The naive implementation

Instead of properly explaining the problem, I'll just show you a solution.

```Rust
// Time complexity: O(H * N)
// Space complexity: O(1)
fn naive_search(haystack: &str, needle: &str) -> bool {
    let [haystack, needle] =
        [haystack, needle].map(str::as_bytes);

    haystack
        .windows(needle.len())
        .any(|haystack_window| haystack_window == needle)
}
```

For each [window] (with the needle size) in the haystack, check if it matches the needle.

[window]: https://doc.rust-lang.org/nightly/std/primitive.slice.html#method.windows

I want this, but vroom vroom (faster).

# The thought process

What I'm looking for:

- For each window, tell if it can be skipped.
- Maximize the amount of skips, so my algorithm runs faster.
- Linear time complexity, and constant space complexity.

Here's what [sliding a window] looks like:

[sliding a window]: https://www.google.com/search?q=sliding+window+technique

|0|1|2|3|4|5|6|7|8|9|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|0|1|2|3|4|5|||||
||1|2|3|4|5|6||||
|||2|3|4|5|6|7|||
||||3|4|5|6|7|8||
|||||4|5|6|7|8|9|

My initial chain of thought:

- What insight can I gather about a **single window**?
    - IDK, start somewhere _cheap_.
- What are the cheapest operations I can do?
    - Arithmetic operations.
- What about multiplication?
    - Like a hash?

Yeah, by comparing the needle hash with the window hash, I can (hopefully) skip most comparisons.

_But_ I'm constrained by **linear time complexity**, with this in mind, how do I calculate the hash for each window?

I know how to do this in `O(N²)`, but not in `O(N)`.

Ok, gotta try something else (we'll come back to hashing later).

- Brain, give me another cheap arithmetic operation.
    - XOR.
- Mmmmmmmmmmmm... nope.
    - Addition?

Yes! Similar to the hash idea, we can compute the sum (of all elements) for both slices, if `sum(needle) != sum(haystack)`, equality is impossible and the check can be skipped.

Also, I know how to do this in `O(N)`:

||0|1|2|3|4|5|6|7|8|9|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|1st|0|1|2|3|4|5|||||
|2nd||1|2|3|4|5|6||||
|3rd|||2|3|4|5|6|7|||
|4th||||3|4|5|6|7|8||
|5th|||||4|5|6|7|8|9|

Calculate the first sum naively:

- First sum is `15`.

For the rest, reuse the previous sum, subtract the removed element and add the new one:

- `Second = First + 6 - 0 = 21`.
- `Third = Second + 7 - 1 = 27`.
- `Fourth = Third + 8 - 2 = 33`.
- `Fifth = Fourth + 9 - 3 = 39`.
- And so on.

That's `O(N)` for all windows, and addition is, of course, fast as hell.

# The algorithm

Let's preprocess the sum of the needle, and calculate the sum for each window, comparing both to perform skips.

```Rust
// Time complexity: O(H * N) // same as naive
// Space complexity: O(1)    // same as naive
fn sum_search(haystack: &str, needle: &str) -> bool {
    // Treat corner cases
    if needle.is_empty() {
        return true;
    } else if needle.len() >= haystack.len() {
        return haystack == needle;
    }

    let [haystack, needle] = [haystack, needle].map(str::as_bytes);

    let mut windows = haystack.windows(needle.len());

    // Unwrap Safety:
    //   We know that `0 < needle.len() < haystack.len()`, there is at
    //   least one window.
    let first_window = windows.next().unwrap();

    let sum_slice = |slice: &[u8]| -> u64 {
        slice.iter().copied().map(u64::from).sum()
    };

    let needle_sum = sum_slice(needle);
    let mut haystack_sum = sum_slice(first_window);

    // Short-circuit the expensive check to skip it
    if needle_sum == haystack_sum && first_window == needle {
        return true;
    }

    // Now, for the rest of the windows.
    for (removed_element_index, window) in windows.enumerate() {
        // Unwrap Safety:
        //   We know that `needle.len() > 0`, every window is non-empty.
        haystack_sum += *window.last().unwrap() as u64;
        haystack_sum -= haystack[removed_element_index] as u64;

        // If the sum doesn't match, skip the check
        if needle_sum != haystack_sum {
            continue;
        }
        // Check equality (expensive check)
        if window == needle {
            return true;
        }
    }

    false
}
```

Time to compare it to the naive solution and the fastest [Boyer–Moore] crate, [`needle`]:

[`needle`]: https://crates.io/crates/needle

(Benchmarked searching for `"Lorem is a ipsum and more"` in a [Shakespeare book].)

```Python
boyer_moore  =  1.52 ms
sum_search   =  2.43 ms # mine
naive_search = 13.87 ms
```

[Shakespeare book]: https://ocw.mit.edu/ans7870/6/6.006/s08/lecturenotes/files/t8.shakespeare.txt

Cool! It's slower than [Boyer–Moore] but faster than the naive algorithm, and it's very simple.

This is curious, there is no `"Lorem"` substring in the text, but there's a lot of `"L"`, `"Lo"` and `"Lor"`, that's probably why `naive_search` is much slower.

Time to see if we're faster than the `std`'s solution:

```Rust
fn std_search(haystack: &str, needle: &str) -> bool {
    haystack.contains(needle)
}
```

```Python
naive_search =  13.87 ms
sum_search   =   2.43 ms
boyer_moore  =   1.52 ms
std_search   = 127.84 µs # ...
```

{{
  image(
    src="/images/tom.jpg",
    alt="Image of Tom (the cat from Tom & Jerry) with a judgemental look in his eyes."
  )
}}

Oh, the needle is so small that the `std` [optimizes it] to [use SIMD] instructions instead.

[optimizes it]: https://github.com/rust-lang/rust/blob/725afd287a27891eec607d6859bce2a551114181/library/core/src/str/pattern.rs#L963-L968
[use SIMD]: https://github.com/rust-lang/rust/blob/725afd287a27891eec607d6859bce2a551114181/library/core/src/str/pattern.rs#L1732

```Rust
#[cfg(all(target_arch = "x86_64", target_feature = "sse2"))]
if self.len() <= 32 {
    if let Some(result) = simd_contains(self, haystack) {
        return result;
    }
}
// PR - https://github.com/rust-lang/rust/pull/103779
```

I'll avoid that by expanding the needle to:

`"Lorem is a ipsum and more, consectetur"` (38 bytes)

```Python
naive_search = 14.22 ms
sum_search   =  2.39 ms
std_search   =  1.27 ms # still very good
boyer_moore  =  1.01 ms
```

# The Karp, and the Rabin

After writing my algorithm down, I started researching [existing solutions]. Do you remember the hash idea? And how I couldn't compute all hashes in `O(N)`?

[existing solutions]: https://en.wikipedia.org/wiki/String-searching_algorithm#Classification_of_search_algorithms

Well, it actually has a name, that's the [`Rabin-Karp` algorithm], it skips checks by using a hash, actually, a *rolling hash*.

[`Rabin-Karp` algorithm]: https://en.wikipedia.org/wiki/Rabin%E2%80%93Karp_algorithm

A *rolling hash* is exactly what I was looking for, with it, you're able to calculate the next window hash based on the previous one, by just peeking at the new (and, sometimes the deleted) elements.

But wait, that's extremely similar to what I did, isn't my sum thing a *rolling hash*?

Here's the [main property] of a hash:

[main property]: https://doc.rust-lang.org/nightly/std/hash/trait.Hash.html#hash-and-eq

```py
a == b -> hash(a) == hash(b)
```

And to be clear, we were relying on the [contraposition](https://en.wikipedia.org/wiki/Material_conditional#Formal_properties) of it to perform the skips:

```py
hash(a) != hash(b) -> a != b   # This has the same meaning as the above
```

Yes, the sequence sum meets this property, so it does look like a rolling hash, but it doesn't comply to the other properties that we usually expect from a hash function (like uniformity, necessary for good bucket distribution).

To settle my doubt, here is [a quote from the `Rabin-Karp` Wikipedia article](https://shorturl.at/jwFGT):

> “A trivial (but not very good) rolling hash function just adds the values of each character in the substring.”

# Closing thoughts

To me, it has become clear that, from the problem statement, you can arrive very close to the [`Rabin-Karp` algorithm], well, at the trivial part of it.

I found two crates that use `Rabin-Karp`:

1. [memchr](https://github.com/BurntSushi/memchr/blob/ce7b8e606410f6c81a63f45abb24c5b5aab5d741/src/arch/all/rabinkarp.rs#L19-L54).
2. [aho-corasick](https://github.com/BurntSushi/aho-corasick/blob/3852632f10587db0ff72ef29e88d58bf305a0946/src/packed/rabinkarp.rs#L177-L184).

(Props to [`@BurntSushi`](https://github.com/burntSushi) for lovely writing the linked comprehensive comments.)

If you're using `axum`, `futures` or the `regex` crate, then your code is probably using `Rabin-Karp`, because they depend on `1` or `2`.

The hard part of `Rabin-Karp` is coming up with a decent *rolling hash function*, the two links above point to [an implementation reference](http://www-igm.univ-mlv.fr/~lecroq/string/node5.html).

---

It's also clear that I ran away from the deep problem when I gave up on the hashing problem, the ideal thing would be to explore on it until I've come up with a good hashing solution.

But what can I expect? That algorithm was all just a shower thought.

# Homework

If I managed to spark your interest on this subject, you should read more about it!

So, here are some suggested points for you to think about:

- Check [the classification of (substring) search algorithms](https://en.wikipedia.org/wiki/String-searching_algorithm#Classification_of_search_algorithms).
- Why is `std`'s [`<&str>::contains`](https://doc.rust-lang.org/nightly/std/primitive.str.html#method.contains) [implementation](https://github.com/rust-lang/rust/blob/4af886f8ab94543caad689ee6bf6a93fa8bd4a98/library/core/src/str/pattern.rs#L1253-L1325) so fast?
- What makes for a good *rolling hash function*?
    - Check [the one](http://www-igm.univ-mlv.fr/~lecroq/string/node5.html) used in `memchr` and `aho-corasick` crates.
- For what inputs each substring search algorithm shines the most?
    - Is it possible to create an adaptative algorithm that collects insights from the input on-the-fly and changes strategy based on it? Is there any gain on falling back to another strategy mid-way, or can you always take the correct decision upfront?

# Extras

## Can the algorithm overflow?

It would require a needle of `72_057 TB`.

## XOR variation

At first, I thought `XOR` wasn't a solution, so I focused on addition.

However, to quickly recalculate the sum for the new window, we were relying on the invertibility property of addition, that is:

```
A + B - B == A
```

Guess what:

```
A ^ B ^ B == A
```

It also applies to XOR, let's see what happens:

```Rust
fn xor_search(haystack: &str, needle: &str) -> bool {
    ...

    let xor_slice = |slice: &[u8]| -> u64 {
        slice.iter().copied().map(u64::from).fold(0, |a, b| a ^ b)
    };
    let needle_xor = xor_slice(needle);
    let mut haystack_xor = xor_slice(first_window);

    ...

    for (removed_element_index, window) in windows.enumerate() {
        // Unwrap Safety:
        //   We know that `needle.len() > 0`, every window is non-empty.
        haystack_xor ^= *window.last().unwrap() as u64;
        haystack_xor ^= haystack[removed_element_index] as u64;
        ...
```

```Python
naive_search = 14.22 ms
xor_search   =  2.69 ms # here
sum_search   =  2.39 ms
std_search   =  1.27 ms
boyer_moore  =  1.01 ms
```

It works, it's still faster than the naive, but its slower than `sum_search` because XOR of `u8`s is limited to `u8::MAX` and the chance of collision is higher, in contrast to the `u64` used in `sum_search`.

The performance of the `sum` solution gets increasingly better for bigger needles, but the `xor`'s should get diminishing return rather quickly (please, do not blindly trust time complexity constraints).

## Inputs that trigger the worst-case performance

Here's a bad case the `sum` solution, but great for `xor`:

```Rust
needle = "22";
haystack = "1313131313131313...";
```

In contrast, this one is good for `sum` and bad for `xor`:

```Rust
needle = "223";
haystack = "113113113...";
```

In `"223"`, the `2`s cancel each other out, the same happens to the `1`s in `"113"`, the resulting XOR for both sequences is `b'3'`.

## Fun results for tiny inputs

For tiny needles (up to 24 bytes of length), this one runs faster than the naive algorithm:

```Rust
fn weird_search(haystack: &str, needle: &str) -> bool {
    let [haystack, needle] =
        [haystack, needle].map(str::as_bytes);

    let sum_slice = |slice: &[u8]| -> u64 {
        slice.iter().copied().map(u64::from).sum()
    };
    let needle_sum = sum_slice(needle);

    haystack.windows(needle.len()).any(|haystack_window| {
        let window_sum = sum_slice(haystack_window);
        window_sum == needle_sum && haystack_window == needle
    })
}
```

It's the naive, plus a sum check, here are the results:

```Python
naive_search =  13.82 ms
weird_search =   8.37 ms # this one
boyer_moore  =   3.82 ms
xor_search   =   2.68 ms
sum_search   =   2.42 ms
std_search   = 137.29 µs
```

Calculating the sum for a whole window can be faster than comparing two slices (in `C`'s `strcmp` fashion), it's probable that `SIMD` instructions are contributing to this.

## Tiny puzzle

What assumptions can we make about `haystack` after these checks?

```Rust
fn sum_search(haystack: &str, needle: &str) -> bool {
    // Treat corner cases
    if needle.is_empty() {
        return true;
    } else if needle.len() >= haystack.len() {
        return haystack == needle;
    }
    ...
}
```

<details>
<summary>Answer (click me) ←</summary>
<code>haystack.len() >= 2</code>
</details>

## Code for the `needle` crate

```Rust
fn boyer_moore(haystack: &str, needle: &str) -> bool {
    ::needle::BoyerMoore::new(needle.as_bytes())
        .find_first_in(haystack.as_bytes())
        .is_some()
}
```

## Comment section

I'd like the comment section to live inside of GitHub, it's an experiment:

Click [here](https://github.com/marcospb19/blog/discussions/2) for the comment section of this post.
