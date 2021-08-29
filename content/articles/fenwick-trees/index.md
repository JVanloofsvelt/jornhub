+++
title = "Fenwick Trees"
date = 2021-08-18T21:21:52+02:00
keywords = ["Fenwick tree", "Binary index tree"]
summary = "This is not the best song in the world, it is just a tribute."
math = true
draft = false
+++

## Intro

When I first learned about binary index trees (also called Fenwick trees) I could find multiple resources that explain how to update and query such a tree. I imagine most people quickly understand why they work, but I came across at least a few comments of people who were left wondering how Peter Fenwick[^1] was able to come up with such an elegant datastructure. Like myself, these people were looking to spark their intuition. Some articles do in fact attempt to transfer some intuition to the reader, but I couldn’t find any that completely satisfied me.
This article is my attempt at explaining the concept. I hope to relieve those of you who have already spent some time reading but are still feeling unsatisfied like I was myself.

I will not recap the properties of a BIT, or the motivation for using a BIT, since this information is readily available. If you’d like an introduction or if you want to refresh your memory, then first head out to [wikipedia](https://en.wikipedia.org/wiki/Fenwick_tree) where the first few paragraphs are a good run-up for the rest of this article.

I will start by positing two key concepts that can be grokked independently, and I hope that by tying those two together I can lead you to that light-bulb moment.

[^1]: Or was it Boris Ryabko?

## Prerequisite

I assume you have spent some time reading and trying to understand Binary Index Trees. Even if you don't fully understand the concept yet, this article may improve your understanding.

## Terminology

Prefix sum
: cumulative sum of some range of elements of some sequence, starting from that sequence's first element. [^prefixsum]

Input array
: the array for which prefix sum queries will be performed, facilitated by the use of a Fenwick tree.  

BIT
: acronym for “Binary index tree”, also called Fenwick tree

LSB
: acronym for “least significant bit” [^lsb]  

MSB
: acronym for “most significant bit” [^msb] 


  
[^lsb]: https://en.wikipedia.org/wiki/Bit_numbering#Least_significant_bit
[^msb]: https://en.wikipedia.org/wiki/Bit_numbering#Most_significant_bit
[^prefixsum]: https://en.wikipedia.org/wiki/Prefix_sum

## Key 1 - Binary representation of integers: a decision tree

Deriving the binary representation of a number can be thought of as a knapsack problem[^knapsack]: given the distinct digits the base-2 numeral system consists of (i.e. 0 and 1), you want to express an integer in the fewest digits/bits possible. You start with the largest power of two that is smaller than the integer (that will be its most significant bit; we're executing a greedy algorithm here) and you fill it up with smaller and smaller bits until your knapsack is full. You then have a combination of bits that precisely represents your integer.

[^knapsack]: https://en.wikipedia.org/wiki/Knapsack_problem

For example, to represent the number 7:

```c
Summing the separate bits:  
  binary:  100 + 010 + 001 = 111  
  decimal:   4 +   2 +   1 =   7

The running total of that summation:  
 binary:  100 -> 110 -> 111  
 decimal:    4 ->  6 ->   7
```
Where 4 and 6 are subsolutions, and 7 is the final solution for the number we wanted to represent.  

Observe that the running total of the previous summation is monotonically increasing: as you add more bits the number represented by the subsolution grows. Note the symmetry with the computation of a prefix sum where you have a subtotal that grows with each step.

We can represent this process as the traversal of some sort of binary decision tree. For each bit of an N-bit number, starting at the MSB, you either decide to include it or not (i.e. set it to 0 or to 1):

{{< figure src="binary-decision-tree.webp" alt="Binary decision tree" width="50%" >}}

In bold is the bit under consideration, if you decide to include it in your number then you continue by traversing to the right, otherwise you continue to the left. All remaining less significant bits are considered to be 0 unless you traverse further down the tree to set them to 1. You can stop your traversal down the tree at any level once you have all positive bits needed to represent your number.

Observe that:

- The number of layers in the tree matches the number of bits needed to represent your number; that is: \\( log_2(max\\_number\\_value) \\). In practical applications of BITs this will often be a constant such as 32 or 64 bits, depending on the datatype you choose to represent an index of the input array.
- Any node in the tree can be a terminal node; you only move further down the tree if you need more positive bits to represent your number. That is also why the last bit of every leaf node is positive: if you didn’t need a positive bit there you would have halted at the previous layer.
You can't traverse further down the right, yet that last bit is included implicitly.
- 0 is not included in the tree, consider it to be a special, but very simple case.
- There is one node for every (non-zero) number that can be represented by N bits; correspondingly, there is one node for every element in an array that can be indexed using your number. **_This symmetry will prove to be the reason why a Fenwick tree can be represented by an array of size equal to that of the input array._**

## Key 2 - A bit manipulation trick

If you have studied the implementation of BITs then you hqve probably already come across the \\( (i\\ \\& -i) \\) trick. This is an expression that works in both C and Python for example, and it returns the least significant positive bit of i; it gives us an integer where only that bit is set.

Why does it work? Assuming i is positive, then making it negative will have the same result as flipping all of the bits of that integer, followed by incrementing it by 1 ([see 2’s complement for why this is the case](https://en.wikipedia.org/wiki/Two's_complement)). If you ‘AND’ it with itself, it will yield the LSB. Let’s first look at an example using i = 6, and then try to generalize:

{{< figure src="bit-manipulation-trick-300pct.webp" alt="Bit manipulation trick" width="80%" >}}

In order to generalize, observe that:

1. Flipping all bits sets all bits that are less significant than the LSB to 1; the lesser significant bits are by definition 0, so flipping them makes them 1.
2. Flipping all bits also sets the LSB to 0
3. Incrementing that by 1 causes a carry-over that sets all those less significant bits to 0 again. The LSB becomes 1 again, since this position receives the carry-over.
4. The more significant bits remain unaffected by the carry-over since we know the LSB was set to 0 before it received the carry-over, and that means there is no further carry-over into the MSBs.
5. In step 4, the more significant bits and their flipped counterpart cancel each other out due to the bitwise AND operation.
6. The less significant bits were 0 for i as well as for –i, so nothing changes there by AND-ing i and -i
7. Only the LSB is set in both i and –i, so it is the only that bit preserved.

Tying these observations together proves that \\( (i\\ \\& -i) \\)  indeed preserves only the LSB.

## First intuition

Let’s imagine that we don’t yet know what Fenwick trees are, and that we’re looking for a way to transform the input array into a different kind of datastructure that affords both prefix-sum lookups and updates that are faster than \\( O(N) \\). Perhaps a kind of divide and conquer approach, would that be effective? We can start by splitting the array in half, and then summarize each half by calculating their sums. Doing this recursively we’d end up with some kind of binary tree so there’s a chance we’ll find a \\( O(log(N)) \\) solution there since that’s how fast you can descend a binary tree.

Now recall our decision tree as described in Key 1; recall how we reconstruct the binary representation of a number, an algorithm we bit-bangers happen to be already intimately familiar with. Notice the following similarity between calculating prefix sums and reconstructing that binary representation: they are both monotonically increasing in nature. If we were to store some partial solution of the prefix sum calculation in that tree, then we can reconstruct the prefix sum at the same pace as we are stepping through the tree: \\( O(log(N)) \\). If we want the prefix sum of the first k numbers, then we would step through the tree to reconstruct integer \\( (k-1) \\) (which is the 0-based index for the k-th element in the input array) while summing up the partial solutions along the way.

## The datastructure

Here below we have a diagram that demonstrates a conceptual decision tree. It maps indexes of an input array to partial sums in another array of equal size. The binary tree reflects how an index number could be decomposed into its separate bits; and correspondingly, how a prefix sum can be decomposed in partial sums.
Let’s imagine the second array is already provisioned with partial sums:

{{< figure src="binary-tree-300pct.webp" alt="Binary tree" width="100%" >}}

At each node the bit under consideration is marked in bold, if the bit is included in the binary representation of the index number then you navigate down the tree along its right child, otherwise along its left child. Each node is mapped to a partial prefix sum, and these partial sums are added to a running total each time you advance from a node to its right child (i.e. the bit is included in the index number).

Note that index 0 is a separate case; it is not part of the binary tree. The prefix sum of the first element is evidently the element itself.

There are quite a few articles on Fenwick trees that try to explain the concept with 1-based indexing because it’s supposedly easier to understand. In my opinion it adds another layer of indirection that is not helpful, and perhaps even confusing. In the drawing we have here, the most significant bit simply divides all non-0 numbers: 1-7 on the left of the root node, and 9-15 on the right. Since this first bit is the entry point for the decomposition of the index number in its separate bits (as described in Key 2), I believe it’s key to have it centered this way.

Now we have an array of prefix sum subsolutions where each element is the sum of a span of elements of the input array. For example, \\( S[1:4] \\) is the sum of 4 elements of the input array:
```c
fenwick_tree[4] = input_array[1] + input_array[2] + input_array[3] + input_array[4];
```

We can visualize these spans by drawing (fat) arrows that point to the index where the sum of its elements is stored:

{{< figure src="fenwick-tree-arrows-300pct.webp" alt="Fenwick tree partial sum spans" width="100%" >}}

If we stack these fat arrows we get a diagram that I have seen reappear in several articles on Fenwick trees so it may look familiar to you. However, when these stacked arrows/bars were the only diagram offered it failed to spark my intuition. This is what it looks like:

{{< figure src="fenwick-tree-stacked-300pct.webp" alt="Fenwick tree spans stacked" width="100%" >}}

These "yardsticks" can be combined so that we can efficiently compute the prefix sum for any span of elements in the input array. At most 4 (or \\(log_2(max\\_index) \\)) subsolutions need to be combined to compute the prefix sum for any index.

## From Binary tree to Fenwick tree

The binary decision tree concept is somewhat confusing because it doesn't reflect the order in which partial sums are added up: if for a certain node the bit it represents is not included in the index number, then nothing needs to be added to the running total of your prefix sum computation. Instead, you traverse down the tree along its left child, leaving the intermediate sum unaffected.

Modifying the previous diagram, we can draw a different tree that more accurately reflects the order of summation:

{{< figure src="fenwick-m-ary-tree-300pct.webp" alt="Fenwick m-ary tree" width="100%" >}}

In red you see a tree that is oriented from left to right instead of top-down. It is not a binary tree, but it is an N-ary tree where each node has up to N children, i.e. \\( log_2(max\\_index) \\). Traversing from a parent to a child means setting one more (less significant) bit to 1 (the next one to the right of the last positive bit); correspondingly, it means adding another subsolution to a running total.

This tree is what appears if you contract the left-descending edges on the previous diagram, thereby eliminating node-visits where no partial sum is added to your running total.

Furthermore, we can see that this tree is skewed: the nodes that represent the larger numbers are found deeper in the tree; that's mostly just a detail with respect to runtime complexity since the number of layers is limited to N, and in practice N will often be a constant of 32 of 64.

I guess this is a more accurate representation of what a Fenwick tree is, rather than the binary tree we drew earlier. This suggests that the name “Binary Index Tree” is perhaps a bit of a misguiding name for this datastructure; or at least so for those still struggling to grok the concept.

## Navigating the Fenwick tree - queries and updates

We did not yet describe a way to navigate this tree: there are no node objects with pointers to child nodes; there are no parent-child references. That’s because we don’t need such references! The indexes that the nodes represent are not arbitrary, they are a set of contiguous numbers and the value of their key is encoded in the way they are stored in the array of subsolutions: the key is the index in the array (after all, that was our deliberate choice when we conceived the binary tree); and because we know the pattern used to initially lay out the conceptual tree (i.e. we know the kind of relationship between parent and child) we can simply calculate the index of each child or parent node.

This is where the bit manipulation from Key 2 comes into play. In the Fenwick tree, navigating from the left to the right (i.e. from a parent to a child node) means setting one more bit to 1. Conversely, navigating from a child to a parent means unsetting the least significant bit. Unsetting a bit is the same as subtracting that bit:

```c
parentIndex = childIndex - LSB(childIndex);
```

### Querying

Decomposing an index into its separate bits, and hence the separate corresponding subsolutions, means recursively subtracting its LSB until the remainder is 0. This little algorithm is what is commonly used to query a Fenwick tree:

```c
int prefix_sum(int i) {
    int sum = fenwickTree[0];
    for (; i != 0; i -= LSB(i))
        sum += fenwickTree[i];
    return sum;
}
```

### Updating

Updating the tree seems to be a little less intuitive. Our Fenwick tree/array contains several overlapping subsolutions for prefix sums, if we update one element of the input array then we need to update several subsolutions in the Fenwick tree. We’d need to add some (and the same) delta to each of those overlapping subsolutions.

We can start at the index of the updated element itself, adding a delta there. Next, we need to update all overlapping subsolutions for indexes that are greater. Only those that are greater, because the prefix sum of smaller indexes should remain unaffected: updating an element in the input array only affects the prefix sum for all indexes that are equal or greater. As you'll see, it is not as evident as simply traversing up the binary tree from child to parent.

So, how do we find these other nodes that need to be updated? Take a look at the diagram below. Note that each overlapping subsolution in the Fenwick tree consecutively finds itself at some level higher in the binary tree, and further to the right at some greater index. To find the first overlapping subsolution we need to add _something_ to our index. Recall that each level of the binary tree corresponds with the position of one bit in the binary numeral; this means that to find the next overlapping subsolution we need to at least trigger a bitwise carry-over when incrementing our index, so that some more significant bit gets set to 1.

{{< figure src="navigate-by-LSB-300pct.webp" alt="Navigate Fenwick tree by LSB" width="100%" >}}

The smallest number that we can add to our index that will trigger a bitwise carry-over is the number that is represented by its LSB. Adding any smaller number will (by definition of LSB) add positive bits to numeral positions where the bits of the index number were 0, and therefore not trigger a carry-over.

Note that \\( 11 + LSB(11) \\)  doesn't take you from a child to a parent node in either the binary tree or the Fenwick tree. It's not that kind of tree traversal.

The following sample recursively adds the LSB of index i to itself, upating the Fenwick tree along the way:

```c
void add(int i, int delta) {
    if (i == 0) {
        fenwickTree[0] += delta;
        return;
    }
    for (; i < SIZE; i+= LSB(i))
        fenwickTree[i] += delta;
}
```
