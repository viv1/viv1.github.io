---
layout: post
title: Beggar's Method - Using combinatorics with coding
date: 2024-03-31 14:42
category: Math in Coding
tags: ["permutation", "combination", "combinatorics", "beggars method", "maths", "coding", "problem solving", "algorithms"]
summary: We explore the 'problem-solving' synergy between maths and coding through the Beggar's Method and its variations.
---

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Introduction](#introduction)
- [Concept Revision](#concept-revision)
- [The Beggar's Method](#the-beggars-method)
- [Variations](#variations)
   * [Variation 1](#variation-1)
   * [Variation 2](#variation-2)
   * [Variation 3](#variation-3)
   * [Variation 4](#variation-4)
   * [Variation 5](#variation-5)
   * [Variation 6](#variation-6)
   * [Variation 7](#variation-7)
   * [Variation 8](#variation-8)
   * [Variation 9](#variation-9)
- [Is Math better than DP in these cases ?](#is-math-better-than-dp-in-these-cases-)
- [References](#references)

<!-- TOC end -->

<!-- TOC --><a href="#" name="introduction"></a>
## Introduction

Mathematics and coding share a common goal: they provide powerful optimizations to simplify seemingly complicated problems. For instance, consider a very trivial example: adding the number 5 twenty times. A brute force approach would be to perform `5 + 5 + 5 + ... (repeated twenty times)`. However, an optimized approach leverages the concept of multiplication, reducing the problem to `5 * 20 = 100`.

In both mathematics and coding, we strive to find optimizations that make problems easier to solve. While brute force methods can solve problems, optimizations often offer more efficient and elegant solutions. The essence of mathematics and coding lies in identifying and applying optimizations that transform intricate problems into simpler, more manageable forms.

By leveraging these optimizations, we can solve problems more efficiently, conserve resources, and unlock solutions that might otherwise be impractical or impossible to achieve through brute force methods alone. 

<!-- TOC --><a href="#" name="concept-revision"></a>
## Concept Revision

We have all learnt permutations and combinations in high school mathematics. Let's recall the definitions:
- ***nCr*** - no. of ways to **select** `r` objects from a group of `n` people.
- ***nPr*** - no. of ways to **arrange** `r` objects from a group of `n` people.

Today, we are going to use combinatorics to solve a specific type of coding problem. As we will see, these questions can indeed be solved using other data structures and techniques, however the combinatorics way is more elegant, faster, and ***intuitive*** (at least when you get the concept). 

<!-- TOC --><a href="#" name="the-beggars-method"></a>
## The Beggar's Method

The famous Beggar's method is a concept to which I was introduced in high school, and the one which made me appreciate mathematics even more.

Let's recall the theory. It says:
```
If there are `n` "identical" coins and `r` beggars, the number of ways to distribute `n` coins among `r` beggars is:

`n+r-1 C r-1`
```

> The keyword "identical" is very important. The coins to be distributed should be of the same type. Otherwise, this formula is not applicable.
{: .prompt-warning }

***Well, how did we arrive here ?***

Let's assume the coins are `*` and the beggars are divided by `|`.

So, this is like selecting ***r-1*** locations from total ***r-1*** `|` + ***n*** `*` positions.

Let's take a visual and more concrete example:

If we have 4 identical coins and 3 beggars, here are the total possibilities:

```bash
Coins + Beggars     B1, B2, B3
* * * * | |         (4, 0, 0)
* * * | * |         (3, 1, 0)
* * * | | *         (3, 0, 1)
* * | * * |         (2, 2, 0)
* * | * | *         (2, 1, 1)
* * | | * *         (2, 0, 2)
* | * * * |         (1, 3, 0)
* | * * | *         (1, 2, 1)
* | * | * *         (1, 1, 2)
* | | * * *         (1, 0, 3)
| * * * * |         (0, 4, 0)
| * * * | *         (0, 3, 1)
| * * | * *         (0, 2, 2)
| * | * * *         (0, 1, 3)
| | * * * *         (0, 0, 4)
```

Notice how we're placing two dividers (`|`) to create three groups among the coins. 
So, if there are `r` beggars, we place `r-1` dividers. The total places where the dividers could be is ```n (for `*`) + (r-1)(for `|`)```

So, why am I writing about this ? How does this help us solve any problem ? Well let me give you several variations this problem can be formulated:

<!-- TOC --><a href="#" name="variations"></a>
## Variations

<!-- TOC --><a href="#" name="variation-1"></a>
### Variation 1
***Q - Find the number of non negative integer solutions of the equation `X + Y + Z = 10`.***

Did you notice this is the exact same as the beggar's method, with 10 as the number of coins (with 1 unit each), and X, Y, and Z being the 3 beggars ?

So, the answer would simply be: 
```bash
(10+3-1) C (3-1) = 12 C 2 = 66.
```
Or,
```python
>>> from math import comb
>>> print(comb(10+3-1, 3-1))
66
```


<!-- TOC --><a href="#" name="variation-2"></a>
### Variation 2
***Q - Find the number of distinct terms in the expansion of `(X + Y + Z)^10`.***

This is exactly same as above. `n+r-1 C r-1`, with n=10 and r=3.


<!-- TOC --><a href="#" name="variation-3"></a>
### Variation 3
***Q - Find the number of solutions of the equation  `X * Y * Z = 1024` , with X,Y,Z as natural numbers (i.e. `>=0`).***
```
1024 = 2^10
Let X = 2^A, Y = 2^B, Z = 2^C

So, X * Y * Z = 2^A * 2^B * 2^C = 2^10
=> A + B + C = 10
```

So, `n+r-1 C r-1`, with n=10 and r=3.

<!-- TOC --><a href="#" name="variation-4"></a>
### Variation 4
***Q - Find the number of solutions for the equation `X1 + X2 +....X5 = 15` such that each `Xi >= 1` (i.e. positive integer solutions) ***

Let's distribute 1 to each `Xi` first. Then `newN = N-r = 15 - 5 = 10`. 

So, the answer is : `(newN+r-1) C r-1` = `10+5-1 C 5-1` = `14C4` = `1001`.


<!-- TOC --><a href="#" name="variation-5"></a>
### Variation 5
***Q - Find the number of integral solutions of the equation `X + Y + Z + W = 20` where `X≥ 1 ,Y≥ 2, Z >0 , W ≥ 3`.***

Let's again distribute minimum values to each of X, Y, Z and W. 

So, similar to previous variation, `newN = 20-(1+2+1+3) = 13`. 
So, the ans is `13+4-1 C 4-1` = `16C3` = `560`.

<!-- TOC --><a href="#" name="variation-6"></a>
### Variation 6
***Q - There are four different dice, in how many ways sum of their outcomes will be `20`.***
Let's formulate this as:
```
X + Y + Z + W = 20, with X,Y,Z,W <= 6 and >=1.

Let X = 6-A, Y = 6-B, Z=6-C, W=6-D, with A,B,C,D >= 0

So, we can rewrite:

X + Y + Z + W = 20
=> 6-X + 6-Y + 6-Z + 6-W = 20
=> 24 - (X+Y+Z+W) = 20
=> X+Y+Z+W = 24-20 = 4

Applying beggars =>
4+4-1 C 4-1 = 7C3 = 35
```

<!-- TOC --><a href="#" name="variation-7"></a>
### Variation 7
***Q - How many non negative integral solutions are there to the system of equations `X + Y + Z + W + K = 20` and `W + K = 7`.***

```
Let, A = W + K
So, X + Y + Z + A = 20 => X + Y + Z = 20-A = 13 

This implies, the number of ways to distribute 13 among X, Y, Z => nCr with n=13 and r = 3
=> 13+3-1 C 3-1 = 105

Also, W + K = 7, So the number of ways to distribute 7 among W, K => nCr with n=7 and r = 2
=> 7+2-1 C 2-1 => 8

So, the total ways = 105 * 8 = 840.
```

<!-- TOC --><a href="#" name="variation-8"></a>
### Variation 8
***Q - Find the number of non negative integral solutions of the equation `X + Y + Z ≤ 10` (i.e. `X, Y, Z >= 0`)***
Let's introduce a new variable `A` to make the equation => `X+Y+Z+A = 10`. 

So, answer is `nCr` with n=10, and r=4 => 
```
10+4-1 C 4-1 = 13C3 = 286
```

<!-- TOC --><a href="#" name="variation-9"></a>
### Variation 9
***Q - Find the number of non negative integral solutions of `2X + Y + Z = 18`.***

```
Y+Z = 18-2X  => Let X=k. So, using beggar's:

((18-2k)+(2-1)) C (2-1) => 19-2k C 1 => 19-2k.

Also, since Y and Z >= 0, so, k will vary from 0 to 9. (18-2k >= 0  => k <= 9). 

Therefore, let's consider each value of k from [0,9] and sum the,:
i.e, ans  = sum(19-2k) for all k in [0,9]. => 190 - 2(9*10/2) = 100.
```



<!-- TOC --><a href="#" name="is-math-better-than-dp-in-these-cases-"></a>
## Is Math better than DP in these cases ?

Yes. Much better. `nCr` computation can be done in `O(N)`. The DP implementation is `O(N*R)`.  Here is a little code to run to see comparative performances. We use `math.comb` from the the `math` module for the most efficient implementation.

```python
from math import comb
import time

def nCrMostEfficient(N, r):
    return comb(N, r)

def nCrDP(n, r):
    """
    Returns the value of n Choose r
    """
    # Base cases
    if r > n:
        return 0
    if r == 0 or r == n:
        return 1

    # Create a 2D array to store the values of nCr
    dp = [[0] * (r + 1) for _ in range(n + 1)]

    # Fill the dp table in a bottom-up manner
    for i in range(n + 1):
        for j in range(min(i, r) + 1):
            # Base cases
            if j == 0 or j == i:
                dp[i][j] = 1
            # Calculate using previous values
            elif i > 0 and j > 0:
                dp[i][j] = dp[i - 1][j] + dp[i - 1][j - 1]

    return dp[n][r]

N = 10
r = 3

start = time.time()
print(nCrMostEfficient(N+r-1, r-1))
end = time.time()
print(f"Combinatorics approach took: {end - start} seconds")

start = time.time()
print(nCrDP(N+r-1, r-1))
end = time.time()
print(f"DP approach took: {end - start} seconds")
```

This gives:

```bash
264385836
Combinatorics approach took: 1.9073486328125e-05 seconds
264385836
DP approach took: 6.794929504394531e-05 seconds
```

The DP approach is `~3.5` times slower for such a small output. For a larger output, the time difference will keep on increasing.

<!-- TOC --><a href="#" name="references"></a>
## References

- [https://kingseducation.in/mod/book/view.php?id=290&chapterid=58](https://kingseducation.in/mod/book/view.php?id=290&chapterid=58){:target="_blank"}

- [https://qr.ae/pNuusN](https://qr.ae/pNuusN){:target="_blank"}