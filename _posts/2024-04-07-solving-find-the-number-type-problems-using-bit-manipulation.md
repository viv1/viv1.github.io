---
layout: post
title: Solving 'find-the-number' problems using bit-manipulation
date: 2024-04-07 23:54
category: Algorithms
tags: ["bit-manipulation", "xor", "leetcode", "coding-problems", "single-number", "single-number-ii", "single-number-iii"]
description: We explore the power of bit manipulation and the XOR operation in solving coding problems involving finding unique elements in an array.
---

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Introduction](#introduction)
- [Revisiting XOR](#revisiting-xor)
- [Problem 1: Single Number](#problem-1-single-number)
- [Problem 2: Single Number II](#problem-2-single-number-ii)
- [Problem 3: Single Number III](#problem-3-single-number-iii)
- [Conclusion](#conclusion)

<!-- TOC end -->

<!-- TOC --><a href="#" name="introduction"></a>
## Introduction

`Bit manipulation` is an interesting way to solve number related problems by doing operations at the `bit` level. There are several basic bit operations like `AND`, `OR`, `XOR`, and `NOT` that can help solve seemingly complicated problems in a simpler way. We will see the power of these ***using 3 "find the number" type problems***: [Single Number I](https://leetcode.com/problems/single-number/description/){:target="_blank"}, [Single Number II](https://leetcode.com/problems/single-number-ii/description/){:target="_blank"} and [Single Number III](https://leetcode.com/problems/single-number-iii/description/){:target="_blank"}.

<!-- TOC --><a href="#" name="revisiting-xor"></a>
## Revisiting XOR

We will be using `XOR` to solve 2 of the 3 problems today, so it makes sense to revisit it.

The `XOR` (`Exclusive OR`) operation is a **logical** bitwise operation that compares two binary values or Boolean values and returns a result based on the following rule:

```
A ⊕ B = (A ^ ~B) ∨ (~A ^ B)
```
In simpler terms, it means:
- If both inputs are the same (either both 0 or both 1), the result is 0.
- If the inputs are different (one is 0, and the other is 1), the result is 1.

The `XOR` operation is often represented by the `⊕` symbol or the `^` symbol in most programming languages.

Here's the truth table for the `XOR` operation:

| A | B | A ⊕ B |
| :-| : | : |
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |


In programming languages, the `XOR` operation can be performed on individual bits. For example, in Python:

```python
a = 3    # Binary: 0011
b = 5    # Binary: 0101
c = a ^ b  # c = 6 (Binary: 0110)
```

In this example, the `XOR` operation is performed bitwise on the binary representations of a and b, resulting in the binary value 0110, which is decimal 6.

The `XOR` operation has several interesting properties that make it useful in various applications, such as:

- Commutative: `A ⊕ B = B ⊕ A`
- Associative: `(A ⊕ B) ⊕ C = A ⊕ (B ⊕ C)`
- Identity element: `A ⊕ 0 = A`
- Self-inverse: `A ⊕ A = 0`
- Cancellation: `A ⊕ B ⊕ A = B`

Now that we have a good understanding of the `XOR` operation, we can explore some leetcode problems that we can solve using `XOR`.

<!-- TOC --><a href="#" name="problem-1-single-number"></a>
## Problem 1: [Single Number](https://leetcode.com/problems/single-number/description/){:target="_blank"}

`Given a non-empty array of integers nums, every element appears twice except for one. Find that single one.`

**Solution**: The key idea is to leverage the fact that `XORing` a number with itself results in 0, and `XORing` a number with 0 returns the number itself.

So, if we `XOR` all the numbers, starting with 0, the numbers which appear twice will become 0. And only the single frequency item will remain, which is the answer.

```python
def singleNumber(nums):
    result = 0
    for num in nums:
        result ^= num
    return result
```

<!-- TOC --><a href="#" name="problem-2-single-number-ii"></a>
## Problem 2: [Single Number II](https://leetcode.com/problems/single-number-ii/description/){:target="_blank"}

`Given an integer array nums where every element appears three times except for one, which appears exactly once. Find the single element and return it.`

**Solution**: The key idea here is that, for numbers which appear 3 times, their set bits will also occur 3 times. And the set bit will occur only one time for the single frequency number. So, all we need to do is count the set-bits at each position, and if count of that bit is not divisible by 3, add the bit to the answer.

- We iterate through all the bit positions of the numbers in the nums array (assuming 32-bit integers).
- For each bit position i, we calculate the sum of the set bits at that position across all the numbers in nums. This is done by right-shifting each number by i bits and taking the bitwise AND with 1 to get the bit value at position i.
- Then, we sum up these bit values for all the numbers.
- Since every number appears three times except for one, the sum of the set bits at each position will be divisible by 3 for the numbers that appear three times, and it will not be divisible by 3 for the single number.
- We take the sum of the set bits modulo 3 (using `bit_sum % 3`) to get the bit value for the single number at position i.

```python
def singleNumberII(nums):
    # 1. Get the sum of all the set bits at each position
    sum_of_set_bits = 0
    for i in range(32):  # Assume 32-bit integers
        bit_sum = sum(num >> i & 1 for num in nums)
        sum_of_set_bits |= (bit_sum % 3) << i

    # 2. The sum_of_set_bits now contains the single number
    return sum_of_set_bits
```


<!-- TOC --><a href="#" name="problem-3-single-number-iii"></a>
## Problem 3: [Single Number III](https://leetcode.com/problems/single-number-iii/description/){:target="_blank"}

`Given an integer array nums, in which exactly two elements appear only once and all the other elements appear exactly twice. Find the two elements that appear only once. You can return the answer in any order.`

**Solution**: So, this is slightly more involved idea. Basically, we do a `XOR` for all numbers (which eventually gives `num1 ^ num2`). Now, find the position of any one set bit in this combined `XOR`. Group the numbers into 2 groups based on this set-bit, i.e. one with numbers with this bit sit, and other with not set. Now, `XOR` of all items in each group gives one answer each. 

Here are the detailed step:

- Calculate the `XOR` of all the elements in the array. This will result in the XOR of the two elements that appear only once, as the elements that appear twice will cancel each other out (`a ^ a = 0`).
- Find the rightmost set bit in the XOR result from step 1. This bit will be different between the two unique elements.
- Divide the array into two groups: one group containing elements with the rightmost set bit, and another group containing elements without the rightmost set bit.
- `XOR` all the elements in each group separately. The XOR of the first group will give us one of the unique elements, and the XOR of the second group will give us the other unique element.
- Do a `XOR` for all numbers. Now, find the position of any one set bit in this XOR. Group the numbers into 2 groups based on this set-bit, i.e. one with numbers with this bit sit, and other with not set. `XOR` of all items in each group gives one answer each.

```python
def singleNumbers(nums):
    # 1. Calculate the XOR of all the elements
    xor = 0
    for num in nums:
        xor ^= num

    # 2. Find the rightmost set bit
    rightmost_set_bit = xor & (-xor)

    # 3. Divide the array into two groups
    num1, num2 = 0, 0
    for num in nums:
        if num & rightmost_set_bit:
            num1 ^= num
        else:
            num2 ^= num

    return [num1, num2]
```

<!-- TOC --><a href="#" name="conclusion"></a>
## Conclusion

We have seen how bit manipulation algorithms made solving the problems easier for us. The advantage of bit manipulation in most cases in the savings on space complexity. It is almost always constant space operation `O(1)`, and implementation is short, which is why it is a great choice when opportunity arises.