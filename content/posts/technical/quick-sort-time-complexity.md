---
title: "Why the Average Complexity of QuickSort is O(nlogn)?"
date: 2025-04-21T17:29:51+07:00
tags: ["math", "computer science"]
---

For most developers, QuickSort is a fast and efficient sorting algorithm with a time complexity of `O(nlogn)`. This makes it significantly better than other common sorting algorithms, like Selection Sort or Bubble Sort, which have a time complexity of `O(n²)`. However, the question remains: **Why is the average time complexity of QuickSort O(nlogn)?**

In this blog, we will delve deep into the mathematical and probabilistic principles that explain this efficiency, helping you understand the underlying reasons why QuickSort is faster than other algorithms on average.

### QuickSort Basics: A Reminder

Before diving into the mathematical reasoning, let’s quickly remind ourselves how QuickSort works. QuickSort is a **divide-and-conquer** sorting algorithm that **recursively partitions an array into two subsets**: one with elements smaller than a pivot value and the other with elements larger than the pivot value. This partitioning continues until the array is fully sorted.

Here's the code that implements the partitioning process in QuickSort:

```python
# Partition function
def partition(arr, low, high):
    pivot = arr[high]
    i = low - 1
    for j in range(low, high):
        if arr[j] < pivot:
            i += 1
            swap(arr, i, j)
    swap(arr, i + 1, high)
    return i + 1

# Swap function
def swap(arr, i, j):
    arr[i], arr[j] = arr[j], arr[i]

# The QuickSort function implementation
def quickSort(arr, low, high):
    if low < high:
        pi = partition(arr, low, high)
        quickSort(arr, low, pi - 1)
        quickSort(arr, pi + 1, high)
```

### How Can an Algorithm Be Considered Effective?

An algorithm is considered effective if it solves a problem efficiently, especially with the least number of comparisons.

For example, consider the following JavaScript code that sorts an array of words based on their length:

```javascript
const words = ["apple", "banana", "cherry", "kiwi", "pear"];
words.sort((a, b) => a.length - b.length);
```

In this case, **the number of comparisons** refers to how many times the expression `a.length - b.length` is evaluated. The function compares the length of each word and orders them accordingly. The question we need to answer is: **How many comparisons does this algorithm make?**

An effective sorting algorithm minimizes unnecessary comparisons. For instance, if we’ve already compared two elements (e.g., "apple" and "kiwi"), there’s no need to compare them again unless it's necessary. This reduces the total number of comparisons, thus improving the algorithm’s efficiency.

Optimal sorting algorithms handle this efficiently by avoiding redundant checks. For example, if we know that "kiwi" is already longer than "apple" and "banana" is shorter, there’s no need to directly compare "kiwi" with "banana" again.

### What is Average Time Complexity?

To understand average time complexity, we need to review some fundamental concepts from probability and statistics that every Vietnamese student learns in their first year of university. If you're unfamiliar with it, you can take a few minutes to revisit this concept on [Expected Value](https://vi.wikipedia.org/wiki/Gi%C3%A1_tr%E1%BB%8B_k%E1%BB%B3_v%E1%BB%8Dng).

Let’s break down the concept with an example:

We start with an ordered array:

```javascript
original = [1, 3, 4, 5, 6, 8, 9];
```

After shuffling the elements:

```javascript
shuffled = [3, 6, 5, 1, 4, 8, 9];
```

Now, we define a random variable `𝑋𝑖𝑗` that equals `1` if the algorithm does compare the
`i-th` smallest and `j-th` smallest elements in **the original array**, and `0` if it does not. Let `𝑋` denote the total number of comparisons made by the algorithm. Since the algorithm never compares the same pair of elements twice, we have:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?X = \sum_{i=0}^{n-1} \left( \sum_{j=i+1}^{n} X_{ij} \right)" title="X = \sum_{i=0}^{n-1} \left( \sum_{j=i+1}^{n} X_{ij} \right)" />
</div>

Therefore, the expected value of `𝑋`, denoted `E[𝑋]`, is:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?E[X] = \sum_{i=0}^{n-1} \left( \sum_{j=i+1}^{n} E[X_{ij}] \right)" title="E[X] = \sum_{i=0}^{n-1} \left( \sum_{j=i+1}^{n} E[X_{ij}] \right)" />
</div>

### Understanding `E[𝑋𝑖𝑗]`

Now, let’s consider one of these `𝑋𝑖𝑗`’s for `i < j`. Denote the `i-th` smallest element in the array by `e𝑖`and the `j-th` smallest element by `e𝑗`. Conceptually, imagine lining up the elements in sorted order. There are three possible cases for the pivot selection during QuickSort:

#### Case 1: The pivot is between `e𝑖` and `𝑒𝑗`

In this case, the two elements `e𝑖` and `𝑒𝑗`end up in different partitions, and we **will never compare them**. This is because the pivot has separated these two elements into separate subsets.

#### Case 2: The pivot is exactly `e𝑖` or `𝑒𝑗`

If the pivot chosen during the partitioning step is either `e𝑖` or `𝑒𝑗`, then we **will compare these two elements directly** because they are now in the same subset.

#### Case 3: The pivot is less than `e𝑖` or greater than `𝑒𝑗`

You might wonder: What happens if the pivot is less than `e𝑖` or greater than `𝑒𝑗`? In these situations, the pivot does not directly affect the comparison between `e𝑖` and `𝑒𝑗`. Once the partitioning step occurs, both `e𝑖` and `𝑒𝑗` will still end up in the same subset. Ultimately, they will converge into one of the two scenarios above where they are compared directly, and thus this case **does not contribute to the expectation**.

At each step, the probability that `𝑋𝑖𝑗 = 1` (i.e., we compare `e𝑖` and `𝑒𝑗`) is exactly `2/(j−i+1)`. Therefore, the overall probability that
`𝑋𝑖𝑗 = 1` is:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?P(X_{ij} = 1) = \frac{2}{(j-i+1)}" title="P(X_{ij}=1) = \frac{2}{(j-i+1)}" />
</div>

### Summing Up the Expected Value

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?E[X_{ij}] = \frac{2\cdot{1} + (j-i-1)\cdot{0}}{(j-i+1)} = \frac{2}{j-i+1}" title="E[X_{ij}] = \frac{2}{(j-i+1)}" />
</div>

This means that for a given element `i`, it is compared to element `i+1` with probability `1`, to element `i+2` with probability `2/3`, to element `i+3` with probability `2/4`, and so on. Therefore, the expected value of X is:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?E[X] = 2 \cdot{\sum_{i=1}^{n-1} \left(\frac{1}{2}+\frac{1}{3}+\frac{1}{4}+...+\frac{1}{n-i+1} \right)}" title="E[X] = \sum_{i=0}^{n-1} \left( \sum_{k=2}^{n-i+1} \frac{2}{n-i+1} \right)" />
</div>

The sum of the series `1 + 1/2 + 1/3 + ... + 1/n`, denoted `𝐻𝑛`, is called the **nth harmonic number**. This series grows logarithmically and can be approximated as:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?Hn = {\displaystyle \ln n+\gamma }" title="Hn = {\displaystyle \ln n+\gamma }" />
</div>

`ln` is the natural logarithm and `γ` is the Euler-Mascheroni constant, approximately `0.577`. Since `γ` is a constant, it does not affect the overall growth rate of the sum. Therefore, in `Big-O` notation, we can express the growth of `𝐻𝑛` as `O(lnn)`, meaning that as n increases,the harmonic number grows logarithmically.

Thus, we can bound the expected value of `𝑋` as:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?E[X] \leq 2\cdot{(n-1)} \cdot{(Hn - 1)} < 2 \cdot{\ln n \cdot{n}}" title="E[X] = (n+1) \cdot \sum_{k=2}^{n} \frac{2}{k} - 2(n-1)" />
</div>

### Conclusion

Through this blog, we’ve seen how QuickSort’s average complexity of `O(nlogn)` arises from the expected value, driven by the harmonic series. While QuickSort is often used as an example, many sorting algorithms can also be understood probabilistically, just like this. Whether or not you see this as essential knowledge, it’s an interesting and indispensable topic in understanding algorithm efficiency at a deeper level. The formulas and concepts we used may seem abstract, but they’re a powerful tool for analyzing and optimizing algorithms.

### References:

- https://www.cs.cmu.edu/afs/cs/academic/class/15451-s07/www/lecture_notes/lect0123.pdf
- https://en.wikipedia.org/wiki/Harmonic_series_(mathematics)
- https://en.wikipedia.org/wiki/Expected_value
- https://www.geeksforgeeks.org/quick-sort-algorithm/
