# Hash-Based Dispatching


## Overview
Hash-based dispatching is a method for integrating expert selection into the hash table lookup operation within hash-based gating networks, eliminating the need for explicit input feature reorganization [S1]. It differs from MLP-based dispatching by avoiding matrix multiplications on features and removing the requirement for a capacity factor, thereby reducing memory overhead and computational latency [S1].

## How it works
**Inputs:** A set of 3D points $x$, a set of $N$ hash experts, and a hash function $H$.
**Outputs:** Feature vectors retrieved from the appropriate expert hash tables for each point.

**Data Structures:**
*   **Hash Tables:** $N$ separate hash tables, one for each expert. Each table maps a hash index to a feature vector.
*   **Hash Function:** A function $H(\cdot)$ that maps a combined input (point coordinates and expert index) to a table index.

**Steps:**
1.  Compute a base hash index for each 3D point $x$ using the point's coordinates. [S1]
2.  Determine the expert index $e$ for each point $x$ via the gating network. [S1]
3.  Pass the expert index $e$ and the base hash index into the hash function $H$ to compute a unique hash table offset. [S1]
4.  Fetch the feature vector from the hash table at the computed offset. [S1]

**Formula:**
Let $h(x)$ be the hash index derived from point $x$. Let $e(x)$ be the expert index selected for point $x$. The final offset $O$ used to fetch the feature is:
$$ O = H(h(x), e(x)) $$
The feature $f$ is then retrieved as:
$$ f = \text{Table}_{e(x)}[O] $$
*Note: The exact implementation of $H$ combines the point-derived hash and the expert index to compute the offset directly [S1].*

**Parameters:**
*   Number of experts $N$: Not given in the sources.
*   Hash table size: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.
*   **Degenerate Input:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hash_based_dispatch(points, gating_expert_indices, hash_tables, hash_function):
    """
    Fetches features from hash tables based on point coordinates and expert indices.
    
    Args:
        points: Array of 3D points (N, 3).
        gating_expert_indices: Array of expert indices for each point (N,).
        hash_tables: List of N hash tables. Each table is a dictionary or array mapping hash index to feature.
        hash_function: Function that takes (point_hash, expert_index) and returns table offset.
        
    Returns:
        features: Array of fetched features (N, D).
    """
    features = []
    
    # 1. Compute base hash index for each point
    # [G] Assuming a standard hash function for 3D coordinates, e.g., MurmurHash or similar
    point_hashes = compute_point_hash(points) 
    
    for i in range(len(points)):
        p_hash = point_hashes[i]
        expert_idx = gating_expert_indices[i]
        
        # 2 & 3. Compute hash table offset using expert index and hash index
        # The expert index is passed into the hash function to compute the offset
        offset = hash_function(p_hash, expert_idx)
        
        # 4. Fetch features from the hash tables
        # [G] Assuming hash_tables[expert_idx] is the table for that expert
        # and the table supports direct indexing by offset
        feature = hash_tables[expert_idx].get(offset)
        
        # Handle collision or empty slot if necessary
        if feature is None:
            feature = zero_feature() # [G] Default fallback for empty hash slots
            
        features.append(feature)
        
    return features
```

## Subtopics

### MoE Dispatching
Mixture of Experts (MoE) dispatching uses matrix multiplication to route features to experts, requiring explicit reorganization of input features and a capacity factor to handle load balancing [S1]. It consumes more GPU memory and training time compared to Hash-based Dispatching [S1].

## Operating characteristics
**Time and Memory Complexity:**
Hash-based dispatching is more efficient than MoE dispatching. In training, Hash dispatching uses 8707M memory compared to MoE's 10386M, and takes 10.7 hours compared to MoE's 15.2 hours [S1]. In testing (inference), Hash dispatching uses 2857M memory compared to MoE's 2981M, and takes 6.65 seconds compared to MoE's 13.5 seconds [S1].

**Measured Performance:**
*   **Training Memory:** 8707M (Hash) vs 10386M (MoE) [S1].
*   **Training Time:** 10.7h (Hash) vs 15.2h (MoE) [S1].
*   **Test Memory:** 2857M (Hash) vs 2981M (MoE) [S1].
*   **Test Time:** 6.65s (Hash) vs 13.5s (MoE) [S1].

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
*   **Strengths:** Eliminates extra memory for reorganizing inputs; eliminates the need for a capacity factor; avoids matrix multiplication on features; boosts efficiency [S1].
*   **Limitations:** Not given in the sources.

**When to use it and when not:**
Use when efficiency in large-scale neural radiance fields is critical and memory/compute overhead of explicit dispatching is a bottleneck [S1]. Not given in the sources for when not to use.

## Background
Hash-based dispatching was introduced as part of the Switch-NeRF++ framework, which builds on HashGrid Encoding and Hash-based Gating [S1]. It was developed to address the inefficiencies of MLP-based gating networks in Switch-NeRF, which required extra memory and computation for dispatching [S1]. The method leverages the inherent structure of hash tables to integrate expert selection directly into the lookup process [S1].

## Sources
[S1] Learning Heterogeneous Mixture of Scene Experts for Large-Scale Neural Radiance Fields., https://doi.org/10.1109/tpami.2025.3603305, fetched 2026-10-26
