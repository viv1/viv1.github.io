---
layout: post
title: Solving the "Bugs" Hackerearth problem with a clever heap
date: 2024-03-23 01:10
category: Data Structures
tags: ["data-structures", "algorithms", "dsa", "heap", "binary-search-tree", "fenwick-tree", "binary-indexed-tree", "advanced", "coding", "interview", "bst", "hackerearth"]
summary: Here I deep-dive into an advanced DSA problem and suggest multiple solutions.
---

## TABLE OF CONTENTS
<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Introduction](#introduction)
- [Problem Statement](#problem-statement)
- [Thought Process and Available Options](#thought-process-and-available-options)
   * [Breaking Down Requirements:](#breaking-down-requirements)
   * [Discarded Naive Approach - Using Linked List:](#discarded-naive-approach-using-linked-list)
   * [Approach 1 - Using BST:](#approach-1-using-bst)
   * [Approach 2 - Fenwick Tree/ BIT:](#approach-2-fenwick-tree-bit)
   * [Approach 3 - Using Two Heaps: ](#approach-3-using-two-heaps)
- [Best Approach:](#best-approach)
- [References:](#references)

<!-- TOC end -->

<!-- TOC --><a href="#" name="introduction"></a>
## Introduction
Few years ago, while practicing for coding interviews, I had solved a fascinating problem on hackerrank. When I came across it again recently, I noticed that no one has described any efficient solution for it yet.

This is an attempt to fill that gap by deepdiving into several approaches and finding the best one.

<!-- TOC --><a href="#" name="problem-statement"></a>
## Problem Statement
```md
You are developer at XYZ company. You like to call the bugs in your code as enemies. You maintain an array A of the list of enemies in decreasing order of their difficulty i.e., the most difficult bug will be the first element of the array. Initally, there is no bugs in the code. You are given N tasks. Each task contains one of the following two types of operations:

1. 1 P: Add a bug with difficulty P into the array A.

2. 2: Sort the array in decreasing order and print the difficulty of (n / 3)th bug in the sorted array, where n is the size of the array A. If the number of bugs is less than 3, print Not enough enemies.

Constrants:
1 <= N <= 5 * 10^5
1 <=n P <= 10^9
```

<!-- TOC --><a href="#" name="thought-process-and-available-options"></a>
## Thought Process and Available Options

Let's try to get the time complexity needed for this solution. There are `N (<= 5 * 10^5)` inputs. If we take assume complexity of finding N/3 as `x`, we know that the total time complexity will become `O(N) * x`.

Now, if `x` is `O(N)`, then total time complexity will become `O(N^2)`. With `5 * 10^5`, this solution will give `TLE (Time Limit Exceeded)`.

So, we need something better than `O(N)` to carry out each operation.

<!-- TOC --><a href="#" name="breaking-down-requirements"></a>
### Breaking Down Requirements:

To solve this problem, we need to maintain a data structure that can efficiently store the bugs (enemies) and perform the following operations:

- Insert a new bug with a given difficulty.
- Sort the bugs in decreasing order of their difficulty.
- Retrieve the difficulty of the (N/3)th bug in the sorted array.

<!-- TOC --><a href="#" name="discarded-naive-approach-using-linked-list"></a>
### Discarded Naive Approach - Using Linked List:
Using an unsorted list to store the bugs and perform the insertion (`O(N)`) and then finding the `N/3` bug (`O(N)`) will be an a linear time operation `O(N)`.

So, as discussed above, we can't go with this approach.

We clearly need some form of efficient structured storage that allows efficient storage and retreival. 


<!-- TOC --><a href="#" name="approach-1-using-bst"></a>
### Approach 1 - Using BST:

A better option is to use a data structure that supports efficient insertion, sorting, and retrieval operations, such as a *binary search tree (BST)*.


Using a binary search tree (BST) can provide efficient time complexities for all the required operations. Here's how we can implement the solution using a BST:

```md
Insertion (Operation 1 P): 
Insert the new bug with difficulty P into the BST. This operation has an average time complexity of O(log N), where N is the number of bugs in the BST.

Sorting and Retrieving (N/3)th Bug (Operation 2): 
To retrieve the (N/3)th bug in the sorted array, we can perform an in-order traversal of the BST, which will give us the bugs in sorted order. During the traversal, we can keep track of the count and return the difficulty of the (N/3)th bug when the count reaches (N/3). This operation has a time complexity of O(N), where N is the number of bugs in the BST.
```

***Time Complexity:***

*Insertion (Operation 1 P):* `O(log N)` on average for a BST, where N is the number of bugs.

*Sorting and Retrieving (N/3)th Bug (Operation 2):* `O(N)` for the in-order traversal of the BST, where n is the number of bugs.

***Space Complexity:***
O(N) to store all elements.

<!-- TOC --><a href="#" name="approach-2-fenwick-tree-bit"></a>
### Approach 2 - Fenwick Tree/ BIT:

We can also use a Fenwick Tree (BIT) to solve this dynamic update and query problem. 

***Algorithm:***
```md
- We initialize a BIT of size MAX_VAL, where MAX_VAL is the maximum possible value of the bug difficulties. 
- When inserting an element x into the BIT, we use x as the index and increment the values of all ancestors of x by 1. This operation is equivalent to calling the update function of the BIT. 
- To remove an element, we decrement the values of its ancestors by 1, again using the update function. 
- To find the rank of an element x, we leverage the sum function of the BIT, which calculates the prefix sum up to index x. 
- Finally, to retrieve the k-th smallest element, we perform a binary search on the BIT, utilizing the rank and sum functions to guide the search process.
```

***Time Complexity:***

*Insertion (Operation 1 P):* O(log MAX_VAL)

*Sorting and Retrieving (N/3)th Bug (Operation 2):* O(log MAX_VAL)

***Space Complexity:***

O(N) to store all elements in the BIT.


<!-- TOC --><a href="#" name="approach-3-using-two-heaps"></a>
### Approach 3 - Using Two Heaps: 

The core idea behind this approach is to maintain two separate heaps: a min-heap of size `(N/3)` to store the `(N/3)` largest elements (most difficult bugs), and a max-heap to store the remaining elements. By keeping the heaps balanced, we can efficiently retrieve the `(N/3)th` element in constant time.

> Wondering if you have seen a related approach anywhere ? Google "Median in a stream of numbers" problem, where you maintain 2 heaps of equal size.
{: .prompt-tip }

( {: .prompt-info }

***Algorithm:***
```md
Initialize an empty min-heap minHeap and an empty max-heap maxHeap.
For each operation of type 1 P (add a bug with difficulty P):
If the size of minHeap is less than (N/3), insert P into minHeap.
Else:
Insert P into maxHeap.
If the top element of maxHeap is greater than the top element of minHeap, swap their values to maintain the heap properties.
For each operation of type 2 (retrieve the (N/3)th bug):
If the size of minHeap is less than (N/3), print Not enough enemies.
Else, print the top element of minHeap (the (N/3)th largest element).
Time Complexity:
```

***Time Complexity:***

***Insertion (Operation 1 P):***

*Best case*: O(log N/3) when inserting into minHeap (size <= N/3)

*Worst case*: O(log N) when inserting into maxHeap and swapping with minHeap

***Retrieval (Operation 2):*** O(1) (top element of minHeap)

***Space Complexity:***

O(N) to store all elements in the heaps.

***Implementation:***

```python
# min heap of size 1/3, max heap of size remaining. 
# when a new element comes, check which heap it should go to
# move around elements accordingly

from heapq import heappush, heappop, heapify
 
minHeap = []
heapify(minHeap) # contains highest 1//3 elements
maxHeap = []
heapify(maxHeap) # contains remaining elements
totalSize = 0

def addToHeap(num):
    global totalSize
    totalSize += 1
    
    idealMinHeapSize = totalSize // 3
 
    if len(minHeap) == 0:
        heappush(minHeap, num)
        return
 
    if len(minHeap) < idealMinHeapSize:
        curr = heappop(maxHeap)
        heappush(minHeap, max(0-curr, num))
        heappush(maxHeap, 0 - min(0-curr, num))
    else:
        if num > minHeap[0]:
            curr = heappop(minHeap)
            heappush(minHeap, max(curr, num))
            heappush(maxHeap, 0 - min(curr, num))
        else:
            heappush(maxHeap, 0 - num)
 
N = int(input())
for i in range(N):
    x = input().split()
    if len(x) == 1:
        if totalSize >= 3:
            print(minHeap[0])
        else:
            print("Not enough enemies")
    else:
        addToHeap(int(x[1]))

```

<!-- TOC --><a href="#" name="best-approach"></a>
## Best Approach:

Two Heaps approach provides constant time retrieval of the (N/3)th element, which is the most critical operation in this problem. Additionally, it has an efficient logarithmic time complexity for insertion and a reasonable space complexity of O(N).

The BST approach is less efficient for retrieval, as it requires a full in-order traversal. The Order Statistic Tree (Fenwick Tree) approach, while having logarithmic time complexity for both insertion and retrieval, has a higher space complexity due to the need to store a Fenwick Tree of size MAX_VAL, which can be prohibitive for large values of MAX_VAL.

The Two Heaps approach strikes the best balance between time and space complexity for this specific problem, making it the preferred solution among the three discussed approaches.

<!-- TOC --><a href="#" name="references"></a>
## References:

- [https://www.hackerearth.com/practice/algorithms/searching/binary-search/practice-problems/algorithm/victory-over-power-4a0cb459/](https://www.hackerearth.com/practice/algorithms/searching/binary-search/practice-problems/algorithm/victory-over-power-4a0cb459/)
- [https://www.geeksforgeeks.org/order-statistic-tree-using-fenwick-tree-bit](https://www.hackerearth.com/practice/algorithms/searching/binary-search/practice-problems/algorithm/victory-over-power-4a0cb459/)
