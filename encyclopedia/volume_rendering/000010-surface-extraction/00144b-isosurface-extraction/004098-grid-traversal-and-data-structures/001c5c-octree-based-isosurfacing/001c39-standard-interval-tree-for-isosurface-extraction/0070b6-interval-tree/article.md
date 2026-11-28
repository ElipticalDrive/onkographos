# Interval Tree
Also called: Dynamic Interval Tree
## Overview
The Interval Tree is a dynamic data structure used to store a collection of intervals on the real line, supporting efficient insertion, deletion, and intersection queries [S1, unofficial]. It is distinguished from static interval structures by its support for dynamic updates while maintaining logarithmic time complexity for operations [S1, unofficial].
## How it works
The Interval Tree maintains a set of intervals $I = [a, b]$ where $a, b \in \mathbb{R}$ [S1, unofficial]. The structure supports three primary operations: Insert, Delete, and Query [S1, unofficial].

**Inputs and Outputs:**
- **Insert(T, I)**: Input is the tree $T$ and interval $I=[a,b]$ where $a,b \in \mathbb{R}$ [S1, unofficial]. Output is the updated tree [S1, unofficial].
- **Delete(T, I)**: Input is the tree $T$ and interval $I=[a,b]$ where $a,b \in \mathbb{R}$ [S1, unofficial]. Output is the updated tree [S1, unofficial].
- **Query(T, I)**: Input is the tree $T$ and query interval $I=[a,b]$ where $a,b \in \mathbb{R}$ [S1, unofficial]. Output is a list of all intervals in $T$ that intersect with $I$ [S1, unofficial].

**Data Structures:**
The specific internal layout (e.g., balanced binary search tree nodes, auxiliary fields) is not detailed in the sources [S1, unofficial]. The structure is treated as an abstract dynamic container supporting the specified operations [S1, unofficial].

**Algorithm Steps:**
1. **Insertion**: To add an interval $I=[a,b]$ to $T$, call `Insert(T, I)` [S1, unofficial]. The operation adds the interval to the collection stored by $T$ [S1, unofficial].
2. **Deletion**: To remove an interval $I=[a,b]$ from $T$, call `Delete(T, I)` [S1, unofficial]. If the interval exists, it is removed; otherwise, the tree remains unchanged [S1, unofficial].
3. **Querying**: To find all intervals intersecting with a query interval $I=[a,b]$, call `Query(T, I)` [S1, unofficial]. The function returns all intervals $J \in T$ such that $J \cap I \neq \emptyset$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def insert_interval(tree, interval):
    """Adds an interval [a, b] to the interval tree. Complexity: O(log(n)) [S1, unofficial]."""
    tree.add(interval)

def delete_interval(tree, interval):
    """Deletes an existing interval [a, b] from the interval tree. Complexity: O(log(n)) [S1, unofficial]."""
    if interval in tree:
        tree.remove(interval)

def query_intervals(tree, query_interval):
    """Queries for all intervals in the tree that intersect with the input interval. Complexity: O(log(n) + k) [S1, unofficial]."""
    a, b = query_interval
    results = []
    # Standard interval tree traversal uses min/max fields in nodes to prune non-intersecting branches in O(log n + k) time [G].
    # The sources do not specify the internal traversal algorithm.
    def traverse(node):
        if node is None:
            return
        if node.left is not None and node.left.min <= b:
            traverse(node.left)
        if max(a, node.interval.start) <= min(b, node.interval.end):
            results.append(node.interval)
        if node.right is not None and node.right.max >= a:
            traverse(node.right)
    traverse(tree.root)
    return results
```

**Parameters:**
- $T$: The interval tree data structure [S1, unofficial].
- $I$: An interval $[a, b]$ where $a, b$ are real numbers [S1, unofficial].
- $k$: The number of intervals returned by a query operation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Non-existent Interval**: The `Delete` operation handles the case where the interval does not exist by doing nothing [S1, unofficial].
- **Precision**: Since intervals are on the real line, floating-point precision issues may arise during intersection checks, though specific mitigation strategies are not provided in the sources [S1, unofficial].
## Operating characteristics
- **Time Complexity**: Insert and Delete operations take $O(\log(n))$ time [S1, unofficial]. The Query operation takes $O(\log(n) + k)$ time, where $k$ is the number of intervals returned [S1, unofficial].
- **Memory Complexity**: Not given in the sources [S1, unofficial].
- **Strengths**: Supports dynamic updates (insertion and deletion) while maintaining efficient query performance [S1, unofficial].
- **Limitations**: The specific internal overhead and memory usage are not detailed [S1, unofficial].
- **When to Use**: Suitable for scenarios requiring frequent updates to the set of intervals and efficient range intersection queries [S1, unofficial].
- **When Not to Use**: If the set of intervals is static, simpler structures like interval tables or fixed-sized buckets may be more appropriate [S1, unofficial].
## Background
The Interval Tree was introduced by [Ede80] [S1, unofficial]. It is a standard data structure in computational geometry and computer science for managing intervals [S1, unofficial].
## Sources
[S1] Compression of Voxelized Vector Field Data by Boxes is Hard, https://doi.org/10.48550/arxiv.2510.20801, fetched 2026-10-26 (unofficial)
