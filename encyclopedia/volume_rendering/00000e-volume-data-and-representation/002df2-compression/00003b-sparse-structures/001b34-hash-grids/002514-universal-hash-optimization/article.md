# Universal Hash Optimization

Also called: Universal Hash Function for Hash Grids

## Overview
Universal Hash Optimization is a technique used within sparse voxel structures, specifically Sparse Voxel Octrees (SVOs), to distribute nodes uniformly across hash table buckets, thereby reducing collision rates [S1, unofficial]. It addresses the problem where standard hashing accumulates a large percentage of nodes in single buckets, causing search times to exceed expectations due to excessive equality function executions [S1, unofficial]. It sets itself apart from sibling methods like [Hash Grid Loss](2093-hash-grid-loss.md) by focusing on the mathematical distribution properties of the hash function itself rather than lossy compression or memory layout optimizations.

## How it works
**Inputs and Outputs:**
- Input: A node $n$ from the sparse voxel structure.
- Output: A hash value (bucket index) for the node.

**Data Structures:**
- The hash table consists of $m$ buckets.
- Parameters $a$ and $b$ are integers chosen from the range $\{0, \dots, p-1\}$.
- $p$ is a prime number used as the modulus for the linear congruential transformation.

**Formulas:**
The core universal hash function is defined as [S1, unofficial]:
$$ H_{ab}(k) = ((a \cdot k + b) \bmod p) \bmod m $$
Where:
- $k$ is the intermediate hash key.
- $a, b \in \{0, \dots, p-1\}$.
- $p$ is a prime number.
- $m$ is the number of buckets in the hash table.

**Algorithm Steps:**
1. Initialize intermediate key $k \leftarrow 0$.
2. Check if node $n$ is a voxel leaf:
   - If $n$ is **not** a voxel (internal node):
     - Iterate through all children $c$ of $n$.
     - For each child $c$, compute its recursive hash $h_c$.
     - XOR the child's hash into $k$: $k \leftarrow k \oplus h_c$.
     - Apply the universal hash transformation to the accumulated key: $k \leftarrow (a \cdot k + b) \bmod p$.
   - If $n$ **is** a voxel (leaf node):
     - Set $k$ to the information value of the node: $k \leftarrow \text{info}(n)$.
3. Compute the final bucket index:
   - Return $H_{ab}(k) = ((a \cdot k + b) \bmod p) \bmod m$.

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
        # Internal node: aggregate children hashes
        for child in node.children:
            # Recursive call to get child's hash
            child_hash = universal_hash_optimization(child, a, b, p, m)
            # XOR accumulation
            k = k ^ child_hash
            # Universal hash transformation step
            k = (a * k + b) % p
    else:
        # Leaf node: use node info as key
        k = node.info
        
    # Final bucket mapping
    return ((a * k + b) % p) % m
```

**Parameters:**
- $p$: A prime number. Typical values are not specified in the sources, but must be large enough to cover the range of intermediate keys [S1, unofficial].
- $m$: The number of buckets in the hash table [S1, unofficial].
- $a, b$: Integers such that $a, b \in \{0, \dots, p-1\}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Collision Probability:** The method relies on the property that the probability of collision between two distinct (non-isomorphic) nodes is less than the size of the table [S1, unofficial]. Poor choice of $a, b, p$ can lead to clustering.
- **Recursive Depth:** For deep octrees, recursion depth may impact stack usage [G].
- **Voxel Info:** The definition of $\text{info}(n)$ for leaf nodes is not specified in the sources [S1, unofficial]. It typically refers to the stored data (e.g., density, color) or a hash of that data.

## Subtopics

### [Memoization for Hash Lookup](5295-memoization-for-hash-lookup.md)
An optimization that caches computed hash values in an external hash table keyed by memory address to avoid redundant recursive computations. It complements Universal Hash Optimization by reducing computational overhead, while Universal Hash Optimization ensures uniform distribution.

## Operating characteristics
- **Time Complexity:** The hashing process is recursive. For an internal node, it processes all children. The complexity depends on the tree depth and branching factor. The final modulo operations are $O(1)$.
- **Memory Complexity:** Requires storage for the hash table of size $m$. The recursive implementation uses stack space proportional to tree depth [G].
- **Performance:** Reduces search times by minimizing collisions, which otherwise require multiple equality checks [S1, unofficial].
- **Strengths:** Provides a theoretically sound method for uniform distribution, reducing the "noise" or unnecessary information accumulation in specific buckets [S1, unofficial].
- **Limitations:** Requires careful selection of parameters $a, b, p$. The recursive nature may be costly without memoization [S1, unofficial].

## Background
The method originates from a thesis on the compression and efficiency of Sparse Voxel Octrees for Global Illumination calculation [S1, unofficial]. It was implemented to address performance degradation caused by hash collisions in standard hash functions applied to SVO structures [S1, unofficial]. The approach adapts a universal hash function [BB11] to the specific structure of the octree nodes.

## Sources
[S1] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
