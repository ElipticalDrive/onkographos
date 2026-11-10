# Isomorphic Node Recycling (DAOG)

Also called: Reciclaje de nodos isomorfos

## Overview
Isomorphic Node Recycling is a compression method that converts a standard Sparse Voxel Octree into a Directed Acyclic Octal Graph (DAOG) by reusing structurally identical (isomorphic) nodes [S1, unofficial]. Unlike standard octrees where every node is a distinct memory object, this method identifies nodes with identical child structures or leaf values and merges them into a single shared instance, reducing memory footprint [S1, unofficial]. It is distinguished from [Bottom-up SVO to DAG Reduction](898-bottom-up-svo-to-dag-reduction.md) by its top-down, hash-based insertion strategy during construction, and from [Pointerless Sparse Voxel Directed Acyclic Graph (PSVDAG)](949-pointerless-sparse-voxel-directed-acyclic-graph-psvdag.md) by its reliance on a hash table for structural equality checks rather than implicit indexing [S1, unofficial].

## How it works
**Inputs:** A voxel $v$, a current node $n$, a bitstream $b$ encoding the spatial position, the current depth $d$, and a hash table $ht$ mapping structural hashes to node references [S1, unofficial].
**Outputs:** The resulting node reference and the updated hash table [S1, unofficial].

**Data Structures:**
1.  **Node:** Contains an array of 8 child pointers (or indices in serialized form) and a 64-bit integer `leafs` for leaf data [S1, unofficial].
    ```cpp
    struct Node {
        Node* children[8]; // Pointers to child nodes
        int64_t leafs;     // Packed leaf data
    };
    ```
2.  **Hash Table:** Maps a computed structural hash to a unique node instance to enable reuse [S1, unofficial].

**Algorithm Steps:**
The method traverses the tree recursively [S1, unofficial]. For each voxel insertion at depth $d$:

1.  **Check Depth:** If $d > 0$ (internal node):
    a.  Extract the child index from the bitstream using 3 bits per level [S1, unofficial].
    b.  Retrieve the child $c$ from the current node $n$ [S1, unofficial].
    c.  **Reuse Check:** If $c$ exists and is already in the hash table, replace $c$ with the canonical node found in the hash table [S1, unofficial].
    d.  **Create New:** If $c$ does not exist, create a new empty node [S1, unofficial].
    e.  **Recurse:** Call the algorithm recursively on child $c$ with depth $d+1$ [S1, unofficial].
    f.  Update the parent node's child pointer to the returned node [S1, unofficial].

2.  **Leaf Level:** If $d = 0$ (leaf node):
    a.  Create a new node $n'$ containing the voxel data $v$ [S1, unofficial].
    b.  **Reuse Check:** If $n'$ is in the hash table, replace $n'$ with the found node [S1, unofficial].
    c.  **Insert:** Otherwise, insert $n'$ into the hash table [S1, unofficial].
    d.  Return $n'$ and the updated hash table [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Node:
    def __init__(self, leaf=None):
        self.children = [None] * 8
        self.leaf = leaf
        self.is_leaf = leaf is not None

def compute_hash(n, memo=None):
    """Structure-dependent hash function with memoization optimization [S1, unofficial]."""
    if memo is None:
        memo = {}
    if id(n) in memo:
        return memo[id(n)]
    
    if n.is_leaf:
        h = n.leaf
    else:
        h = 0
        for c in n.children:
            if c is not None:
                h ^= compute_hash(c, memo)
        # Optimization: linear congruential generator to distribute nodes uniformly in buckets [S1, unofficial]
        a, b_val, p = 1, 1, 2**31 - 1
        h = (a * h + b_val) % p
    
    memo[id(n)] = h
    return h

def insert_voxel(v, n, b, d, ht):
    """
    Insert a voxel into the DAOG, reusing isomorphic nodes [S1, unofficial].
    :param v: Voxel data
    :param n: Current node
    :param b: Bitstream representing spatial position
    :param d: Current depth
    :param ht: Hash table {hash_value: node_reference}
    :return: (node, updated_hash_table)
    """
    if d > 0:
        # Extract 3 bits for child index at depth d
        child_idx = (b >> (d * 3)) & 7
        
        c = n.children[child_idx]
        if c is not None:
            h = compute_hash(c)
            if h in ht:
                c = ht[h]
        else:
            c = Node()
            
        c, ht = insert_voxel(v, c, b, d + 1, ht)
        n.children[child_idx] = c
        return n, ht
    else:
        n_prime = Node(leaf=v)
        h = compute_hash(n_prime)
        if h in ht:
            n_prime = ht[h]
        else:
            ht[h] = n_prime
        return n_prime, ht
```

**Parameters:**
*   **Hash Table:** Essential for average-case $O(1)$ lookup of isomorphic nodes [S1, unofficial].
*   **Bitstream Grouping:** 3 bits are used to index the 8 children at each level [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** The XOR-based hash may produce collisions for different structures [S1, unofficial]. The source applies a linear congruential generator to distribute nodes uniformly in buckets to mitigate this [S1, unofficial].
*   **Depth Direction:** The source explicitly uses $d+1$ in the recursive call for internal nodes [S1, unofficial].
*   **Memory Management:** Reusing nodes means multiple parents point to the same child [S1, unofficial]. The source describes a static construction phase, but shared references require careful handling if nodes are mutated [S1, unofficial].

## Subtopics

### Octree Serialization for GPU
A method to convert the DAOG into a contiguous memory buffer for GPU transmission, replacing pointers with indices [S1, unofficial].
*   **Definition:** Reaggregating nodes into a contiguous vector to eliminate memory gaps and replacing memory pointers with indices into the vector [S1, unofficial].
*   **Steps:**
    1.  Reaggregate nodes into a contiguous vector [S1, unofficial].
    2.  The first entry in the vector is always the root node [S1, unofficial].
    3.  Replace memory pointers with `int32_t` indices pointing to child nodes in the vector [S1, unofficial].
    4.  Serialize using a Depth First traversal [S1, unofficial].
*   **Layout:**
    ```cpp
    struct Node {
        int32_t children[8]; // Indices into the vector
        int64_t leafs;       // Packed leaf data
    };
    ```
*   **Characteristics:** The serialization order is Depth First, chosen arbitrarily due to time constraints [S1, unofficial].

### Structure-Dependent Hash Function
A specific hash function used to identify isomorphic nodes based on their recursive structure [S1, unofficial].
*   **Definition:** Computes a hash value by recursively hashing children and combining them [S1, unofficial].
*   **Algorithm:**
    1.  If the node is not a voxel (internal node):
        a.  Initialize hash $h = 0$ [S1, unofficial].
        b.  For all children $c$ in the node: $h \leftarrow h \oplus \text{recursive}(c)$ [S1, unofficial].
        c.  Return $h$ [S1, unofficial].
    2.  If the node is a voxel:
        a.  Return `info(n)` (the leaf data) [S1, unofficial].
*   **Optimization:** The source applies a linear congruential generator ($k \leftarrow (a \cdot k + b) \mod p$) to distribute nodes uniformly in buckets [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Insertion is $O(D)$ per voxel, where $D$ is the tree depth, plus the cost of hash computation [S1, unofficial]. Hash computation requires traversing the subtree unless memoization is used [S1, unofficial].
*   **Memory Complexity:** Reduced compared to a standard octree by sharing isomorphic nodes [S1, unofficial]. The hash table adds overhead proportional to the number of unique node structures [S1, unofficial].
*   **Strengths:** Not given in the sources.
*   **Limitations:** Hash table lookup reduces algorithm efficiency [S1, unofficial]. The source notes that optimizations (memoization, uniform distribution) were required to mitigate this [S1, unofficial].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Originated in the thesis "Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global" [S1, unofficial]. The method transforms the octree into a Directed Acyclic Octal Graph (DAOG) to exploit spatial redundancy and improve memory efficiency [S1, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
