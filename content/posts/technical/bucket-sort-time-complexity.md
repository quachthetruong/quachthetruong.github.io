---
title: "Why the Average Complexity of Bucket sort is O(n)?"
date: 2025-05-20T17:29:51+07:00
tags: ["math", "computer science"]
---

### Part 1: Introduction and Code

Bucket Sort is an efficient sorting algorithm when input values are uniformly distributed over a range. It works by distributing elements into different "buckets", sorting each bucket, and then concatenating the results.

Here’s a typical Python implementation where each bucket is sorted with `Insertion Sort`:

```python
def insertion_sort(bucket):
    for i in range(1, len(bucket)):
        key = bucket[i]
        j = i - 1
        while j >= 0 and bucket[j] > key:
            bucket[j + 1] = bucket[j]
            j -= 1
        bucket[j + 1] = key

def bucket_sort(arr):
    n = len(arr)
    buckets = [[] for _ in range(n)]

    # Put array elements in different buckets
    for num in arr:
        bi = int(n * num)  # assuming input numbers are in [0,1)
        buckets[bi].append(num)

    # Sort individual buckets using insertion sort
    for bucket in buckets:
        insertion_sort(bucket)

    # Concatenate all buckets into arr[]
    index = 0
    for bucket in buckets:
        for num in bucket:
            arr[index] = num
            index += 1
```

#### Why Insertion Sort?

Insertion sort is simple and efficient for small or nearly sorted lists. Since each bucket contains only a fraction of the input, sorting each bucket with insertion sort is fast.

Insertion sort complexity: Sorting a bucket with `n` elements costs `O(n²)`.

### Part 2: Mathematical Proof

Setting Up the Problem:

- `n`: total number of elements.
- `k`: number of buckets.
- `n_i`: number of elements in bucket i.

We model the assignment of elements to buckets using indicator random variables:

<div style="display: inline-block; background-color: #6A6767; width: 30rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> <img src="https://latex.codecogs.com/svg.latex?\color{white}X_{ij}=\begin{cases}1 &\text{if } j\text{ is in bucket }i\\0 &\text{otherwise}\end{cases}" title="X_{ij} definition" /> </div>

Using this, we express the size of each bucket `n_i` as:

<div style="display: inline-block; background-color: #6A6767; width: 30rem; height: 4rem; padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> <img src="https://latex.codecogs.com/svg.latex?\color{white}n_i = \sum_{j=1}^{n} X_{ij}" title="n_i = \sum_{j=1}^{n} X_{ij}" /> </div>

This setup allows us to compute the expected value of `n_i^2` which is crucial for bounding the sorting time in each bucket.

We want calculate expected value of `n_i^2`:

<div style="display: inline-block; background-color: #6A6767; width: 30rem; height: 5rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> <img src="https://latex.codecogs.com/svg.latex?\color{white}E(n_i^2) = E\left(\left(\sum_{j=1}^{n} X_{ij}\right)^2\right) = E\left(\sum_{j=1}^{n}\sum_{l=1}^{n} X_{ij}X_{il}\right)" title="E[n_i^2] = E\left[\left(\sum_{j=1}^{n} X_{ij}\right)^2\right]" /> </div>

Split the sum:

<div style="display: inline-block; background-color: #6A6767; width: 30rem; height: 5rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> <img src="https://latex.codecogs.com/svg.latex?\color{white}E(n_i^2) = E\left(\sum_{j=1}^n X_{ij}^2\right) + E\left(\sum_{j\neq{l}}X_{ij}X_{il}\right)" title="E[n_i^2] = E\left[\sum_{j=1}^n X_{ij}^2 + \sum_{j \neq l} X_{ij} X_{il}\right]" /> </div>

In the final expansion, the summation splits into two cases: when `j=l` and when `j≠l`. Since each element is equally likely to go into any bucket, the probability that a given element `j` ends up in bucket `i` is
`1/k`. This means the indicator variable `X_ij` equals `1` with probability `1/k`, and `0` otherwise.

So, we compute the expectations:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> <img src="https://latex.codecogs.com/svg.latex?\color{white}E(X_{ij}^2) = 1^2 \cdot \frac{1}{k} + 0^2 \cdot \left(1 - \frac{1}{k}\right) = \frac{1}{k}" title="\mathbb{E}[X_{ij}^2] = 1^2 \cdot \frac{1}{k} + 0^2 \cdot \left(1 - \frac{1}{k}\right) = \frac{1}{k}" /> </div>

And when `j≠l`, because element assignments are independent:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> <img src="https://latex.codecogs.com/svg.latex?\color{white}E(X_{ij} X_{il}) = \frac{1}{k} \cdot \frac{1}{k} = \frac{1}{k^2}" title="\mathbb{E}[X_{ij} X_{il}] = \frac{1}{k} \cdot \frac{1}{k} = \frac{1}{k^2}" /> </div>

Substituting this into the total cost:

<div style="display: inline-block; background-color: #6A6767; width: 100%; max-width: 90rem; padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;  height: 5rem;"> <img src="https://latex.codecogs.com/svg.latex?\color{white}E\left(\sum_{j=1}^{n}X_{ij}^{2}\right)+E\left(\sum_{1\leq j,k\leq n,\; j\neq k}X_{ij}X_{ik}\right)=n\cdot\frac{1}{k}+n(n-1)\cdot\frac{1}{k^{2}}=\frac{n^{2}+nk-n}{k^{2}}" title="E\left(\sum_{j=1}^{n}X_{ij}^{2}\right)+E\left(\sum_{1\leq j,k\leq n,\; j\neq k}X_{ij}X_{ik}\right)=n\cdot\frac{1}{k}+n(n-1)\cdot\frac{1}{k^{2}}=\frac{n^{2}+nk-n}{k^{2}}" /> </div>

Final Complexity

<div style="display: inline-block; background-color: #6A6767; width: 100%; max-width: 85rem; padding-left: 1rem; align-items: center; display:flex; border-radius: 8px; height: 5rem;"> <img src="https://latex.codecogs.com/svg.latex?\color{white}O\left(\sum_{i=1}^{k}E(n_{i}^{2})\right)=O\left(\sum_{i=1}^{k}\frac{n^{2}+nk-n}{k^{2}}\right)=O\left(\frac{n^{2}}{k}+n\right)" title="O\left(\sum_{i=1}^{k}E(n_{i}^{2})\right)=O\left(\sum_{i=1}^{k}\frac{n^{2}+nk-n}{k^{2}}\right)=O\left(\frac{n^{2}}{k}+n\right)" /> </div>

Therefore, if `k=n`, i.e. we use `n` buckets, the expected time complexity becomes:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> <img src="https://latex.codecogs.com/svg.latex?\color{white}O\left(\frac{n^{2}}{n}+n\right) = O(n)" title="O(n)" /> </div>

### Part 3: Why Isn’t Bucket Sort Popular in Practice?

On paper, Bucket Sort sounds amazing — it's one of the few sorting algorithms that can achieve linear time. But in reality, you’ll rarely see it used in production systems or standard libraries like `Python’s sort()` or `Java’s Arrays.sort()`.

Why? It comes down to strict limitations:

- It only works well on uniformly distributed data. If your input values are clustered or uneven, bucket sort can slow down dramatically.

- It needs a lot of memory. In the best case, you create as many buckets as input elements, which is often impractical.

- It’s not in-place. That means it copies data around, consuming extra space compared to in-place algorithms like quicksort.

- It doesn’t generalize easily. For example, it can’t handle complex comparison logic, custom comparators, or non-numeric types well.

So, despite its theoretical speed, Bucket Sort is rarely the right tool for general-purpose sorting.
But in specific, controlled use cases, the bucket idea can still be useful:

- Distributed systems like Apache Spark or Hadoop use a similar idea called bucket partitioning — breaking data into ranges to parallelize processing.

- Radix Sort, a cousin of bucket sort, is used in systems where data can be broken down into digits or bytes — like sorting IP addresses, phone numbers, or fixed-length IDs — and works extremely well.

### Part 4: Conclusion

In this article, we've broken down why Bucket Sort can theoretically run in O(n) time, and how that result depends on strong assumptions like uniform distribution and using many buckets.

But in practice, these ideal conditions are rare. That’s likely why Bucket Sort doesn’t show up much in popular tools or libraries.

> If you’ve seen Bucket Sort used in a real application or system — not just as an example in a textbook — I’d love to hear about it. I'm still looking for real-world use cases beyond the theory.
