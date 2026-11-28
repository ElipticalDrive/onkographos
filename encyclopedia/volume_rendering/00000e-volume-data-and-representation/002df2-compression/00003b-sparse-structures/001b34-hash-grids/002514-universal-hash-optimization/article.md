# Universal Hash Optimization

Also called: Universal Hash Function for Hash Grids

## Overview
Universal Hash Optimization is a technique used within sparse voxel structures, specifically Sparse Voxel Octrees (SVOs), to distribute nodes uniformly across hash table buckets, thereby reducing collision rates [S1, unofficial]. It addresses the problem where standard hashing accumulates a large percentage of nodes in single buckets, causing search times to exceed expectations due to excessive equality function executions [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A node $n$ from the sparse voxel structure [S1, unofficial].
- Output: A hash bucket index for the node [S1, unofficial].

**Data Structures:**
- The hash table consists of $m$ buckets [S1, unofficial].
- Parameters $a$ and $b$ are integers chosen from the range $\{0, \dots, p-1\}$ [S1, unofficial].
- $p$ is a prime number used as the modulus for the linear congruential transformation [S1, unofficial].

**Formulas:**
The core universal hash function is defined as [S1, unofficial]:
$$ H_{ab}(k) = ((a \cdot k + b) \bmod p) \bmod m $$
Where:
- $k$ is the intermediate hash key [S1, unofficial].
- $a, b \in \{0, \dots, p-1\}$ [S1, unofficial].
- $p$ is a prime number [S1, unofficial].
- $m$ is the number of buckets in the hash table [S1, unofficial].

**Algorithm Steps:**
1. Initialize intermediate key $k \leftarrow 0$ [S1, unofficial].
2. If node $n$ is not a voxel, iterate through all children $c$ of $n$ [S1, unofficial].
3. For each child $c$, compute its recursive hash $h_c$ [S1, unofficial].
4. XOR the child's hash into $k$: $k \leftarrow k \oplus h_c$ [S1, unofficial].
5. Apply the universal hash transformation to the accumulated key: $k \leftarrow (a \cdot k + b) \bmod p$ [S1, unofficial].
6. If $n$ is a voxel, set $k$ to the information value of the node: $k \leftarrow \text{info}(n)$ [S1, unofficial].
7. Return $H_{ab}(k) = ((a \cdot k + b) \bmod p) \bmod m$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def universal_hash_optimization(node, a, b, p, m):
    """
    Computes the hash bucket index for a node in a sparse voxel structure
    using a universal hash function to minimize collisions.
    
    Args:
        node: The current node in the tree.
        a: Integer multiplier for the hash function.
        b: Integer additive constant for the hash function.
        p: Prime modulus for the linear congruential generator.
        m: Number of buckets in the hash table.
        
    Returns:
        int: The bucket index for the node.
    """
    k = 0
    
    if not node.is_voxel:
        for child in node.children:
            child_hash = universal_hash_optimization(child, a, b, p, m)
            k = k ^ child_hash
            k = (a * k + b) % p
    else:
        k = node.info
        
    return ((a * k + b) % p) % m
```

**Parameters:**
- $p$: A prime number. Typical values are not specified in the sources.
- $m$: The number of buckets in the hash table [S1, unofficial].
- $a, b$: Integers such that $a, b \in \{0, \dots, p-1\}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- The method relies on the property that the probability of collision between two distinct (non-isomorphic) nodes is less than the table size [S1, unofficial].
- The definition of $\text{info}(n)$ for leaf nodes is not specified in the sources [S1, unofficial]. It typically refers to the stored data (e.g., density, color) or a hash of that data [G].

## Subtopics

### [Memoization for Hash Lookup](5295-memoization-for-hash-lookup.md)
An optimization that caches computed hash values in an external hash table keyed by memory address to avoid redundant recursive computations. It complements Universal Hash Optimization by reducing computational overhead, while Universal Hash Optimization ensures uniform distribution.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Reduces search times by minimizing collisions, which otherwise require multiple equality checks [S1, unofficial].
- **Strengths:** Provides a theoretically sound method for uniform distribution, reducing the accumulation of nodes in specific buckets [S1, unofficial].
- **Limitations:** Requires careful selection of parameters $a, b, p$ to maintain the collision probability bound [S1, unofficial].

## Background
The method originates from a thesis on Sparse Voxel Octrees [S1, unofficial]. It was implemented to address performance degradation caused by hash collisions in standard hash functions applied to SVO structures [S1, unofficial]. The approach adapts a universal hash function [BB11] to the specific structure of the octree nodes [S1, unofficial].

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
