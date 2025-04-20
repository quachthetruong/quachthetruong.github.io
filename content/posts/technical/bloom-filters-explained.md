---
title: "Bloom Filters Explained: A Fast and Space-Efficient Probabilistic Solution"
date: 2025-04-18T17:29:51+07:00
tags: ["math", "computer science"]
---

## Part 1: Motivation

### How do we check if something is in a set — fast?

The simplest way is a `List`:

```python
if x in items:
  ...
```

But this is `O(n)` — too slow for large-scale systems.

A `HashSet` improves to `O(1)` lookups on average,
but it stores the full elements, requiring **more memory than raw data** — especially for strings or objects.

### So what if we trade a little accuracy for massive savings?

What if a structure could:

- Use only `~9.6` bits per element (significantly smaller than storing object or string),

- Be wrong only `1%` of the time (false positives),

- And never say "no" to something that’s truly there?

That’s the power of the **Bloom Filter**.

### Where is it used?

Bloom filters are quietly at the heart of many systems:

- LSM trees, the foundation of modern NoSQL databases like Apache Cassandra, MongoDB, use Bloom filters to skip disk reads — asking:

> “Does this file probably contain the key?”

- Platforms like Quora, Medium, and Yahoo use Bloom filters to:

  - Prevent duplicate content,

  - Avoid redundant processing,

  - Speed up internal caching systems.

Even if you don’t see them — Bloom filters are working behind the scenes, making large systems fast and efficient.

## Part 2: What is a false positive? (Definition + Example)

### What is a false positive?

✅ When you check an element that **was inserted**, the Bloom filter says “yes” — that’s a `true positive`, and it’s 100% guaranteed correct.

⚠️ But sometimes it says “yes” to something that **was never inserted** — that’s a `false positive`.

A false positive happens when a new element matches **the bit pattern of others** — even though it was never added.

> The filter sees all bits set and says “Probably yes” — but it’s wrong.

That’s the **trade-off for speed and space** — and you’ll see it clearly in the [next example.](#example-when-a-false-positive-happens)

### How does it work?

A Bloom filter is built with:

- `m`: a bit array of length `m`, all bits start at 0

- `k`: `k` independent hash functions

- `n`: number of elements inserted

To insert an element `O(1)`:

- Hash it using all `k` functions

- Flip the corresponding `k` bits to 1

To check membership `O(1)`:

- Hash the element using the same `k` functions

- If any of the `k` bits is 0 → the element is definitely not in the set

- If all are 1 → the element is possibly in the set (could be a false positive)

### Example: When a False Positive Happens

Let’s step through a small Bloom filter:

#### Setup:

- `m` = 10 (bit array of size 10)

- `k` = 3 hash functions

- Inserted elements: 'apple' and 'banana' so `n` = 2

Start with an empty bit array:

```makefile
Initial: [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```

#### Step 1: Insert 'apple'

Let’s say:

- h₁(apple) = 2

- h₂(apple) = 4

- h₃(apple) = 7

Update the bit array:

```makefile
After insert apple: [0, 0, 1, 0, 1, 0, 0, 1, 0, 0]
```

#### Step 2: Insert 'banana'

Let’s say:

- h₁(banana) = 1

- h₂(banana) = 4

- h₃(banana) = 8

Update the bit array:

```makefile
After insert banana: [0, 1, 1, 0, 1, 0, 0, 1, 1, 0]
```

#### Step 3: Check 'banana' (True Positive)

- h₁(banana) = 1 → bit is 1

- h₂(banana) = 4 → bit is 1

- h₃(banana) = 8 → bit is 1

All bits are 1 → Bloom filter says “Yes” → ✅ correct!

#### Step 4: Check 'mango' (False Positive)

- h₁(mango) = 2

- h₂(mango) = 4

- h₃(mango) = 7

Check bits:

- 2 → 1 ✅

- 4 → 1 ✅

- 7 → 1 ✅

Bloom filter says “Yes”, but 'mango' was never inserted → ❌ false positive

This happens because 'apple' and 'banana' already set those bits.

## Part 3: Mathematical Proof

Okay — now you understand what a false positive is.
Let’s take it a step deeper and explore the probability theory behind Bloom filters.

Don’t worry — it only uses basic math that every student learns in university.
We'll walk through this step-by-step, keeping things intuitive.

### What is our goal?

We want to answer:

> What’s the probability that a new element (not in the set) returns a false positive?

That means: all the `k` bits checked during the query are already 1—
even though the element was never inserted, not even once among the `n` inserted items.

#### Step 1: Probability a Bit is Still 0 After One Flip

Suppose we have an array of `m` bits, all starting as 0.
Now we flip 1 random bit to 1.
The chance that a specific bit stays 0 is:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem;
align-items: center; display:flex; border-radius: 8px;">
<img src="https://latex.codecogs.com/svg.latex?\space\color{white}P[\text{bit = 0 after 1 flip}] = 1 - \frac{1}{m}
" title="P[\text{bit = 0 after 1 flip}] " />
</div>

Why? Because we only had a `1/m` chance to hit it.

#### Step 2: We Perform `k × n` Bit Flips

We insert `n` elements, each hashed with `k` functions. So we flip bits `k × n` times.

The probability that a specific bit is still 0 after all those flips is:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem;
align-items: center; display:flex; border-radius: 8px;"><img src="https://latex.codecogs.com/svg.latex?\space\color{white}P[\text{bit = 0 after } kn \text{ flips}] = \left(1 - \frac{1}{m} \right)^{kn}" title="P[\text{bit = 0 after } kn \text{ flips}] " />
</div>

#### Step 3: Exponential Limit Approximation

When `m` is large and `kn` is not too huge, we can approximate this with the exponential function:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem;
align-items: center; display:flex; border-radius: 8px;"><img src="https://latex.codecogs.com/svg.latex?\space\color{white}
P[\text{bit = 0 after } kn \text{ flips}] = \left(1 - \frac{1}{m} \right)^{kn} \approx e^{-kn/m}" title="\left(1 - \frac{1}{m} \right)^{kn} \approx e^{-kn/m}" />
</div>

This comes from the identity:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem;
align-items: center; display:flex; border-radius: 8px;">
<img src="https://latex.codecogs.com/svg.latex?\space\color{white}\lim_{m \to \infty} \left(1 - \frac{1}{m} \right)^m = \frac{1}{e}" title="\lim_{m \to \infty} \left(1 - \frac{1}{m} \right)^m = \frac{1}{e}" />
</div>

#### Step 4: Probability Bit is 1 (i.e. Flipped at Least Once)

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem;
align-items: center; display:flex; border-radius: 8px;"><img src="https://latex.codecogs.com/svg.latex?\space\color{white}P[\text{bit = 1}] \approx 1 - e^{-kn/m}" title="P[\text{bit = 1}] \approx 1 - e^{-kn/m}" />
</div>

This tells us how likely a bit is to be 1 after inserting `n` elements.

#### Step 5: False Positive = All `k` Bits Are 1

Now, suppose we query a new element that wasn’t inserted. Its `k` hash functions give us `k` bit positions. The probability that all those k bits are already 1 — just by chance — is:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem;
align-items: center; display:flex; border-radius: 8px;"><img src="https://latex.codecogs.com/svg.latex?\space\color{white}P[\text{false positive}] \approx \left(1 - e^{-kn/m} \right)^k" title="P[\text{false positive}] \approx \left(1 - e^{-kn/m} \right)^k" />
</div>

And that’s the final formula for Bloom filter false positives.

### Bloom Filter Example: 1 Million Items (n = 1,000,000)

| Size per element (m/n) | Total Size | Hash Functions (k) | False Positive Rate |
| :--------------------: | :--------: | :----------------: | :-----------------: |
|       6.25 bits        |  0.78 MB   |         4          |        4.99%        |
|        7.5 bits        |  0.94 MB   |         5          |        2.70%        |
|       9.58 bits        |  1.20 MB   |         6          |        1.00%        |
|       12.5 bits        |  1.56 MB   |         8          |        0.25%        |

## Part 4: What’s Next — Other Smart Probabilistic Structures for Big Data Problems

Bloom filters solve set membership efficiently — but what about other fundamental questions in computer science?

- How many unique users have viewed this video? → `HyperLogLog`

- How many times did user X access this page? → `Count-Min Sketch`

- What are the 50th, 90th, and 99th percentiles of the measured latencies? → `t-digest`

Want more?
→ Explore more [probabilistic data structures](https://en.wikipedia.org/wiki/Category:Probabilistic_data_structures).

### References:

- https://www.amazon.com/Probabilistic-Data-Structures-Algorithms-Applications/dp/3748190484
- https://en.wikipedia.org/wiki/Category:Probabilistic_data_structures
- https://brilliant.org/wiki/bloom-filter/
- https://redis.io/docs/latest/develop/data-types/probabilistic/
- https://en.wikipedia.org/wiki/Bloom_filter
