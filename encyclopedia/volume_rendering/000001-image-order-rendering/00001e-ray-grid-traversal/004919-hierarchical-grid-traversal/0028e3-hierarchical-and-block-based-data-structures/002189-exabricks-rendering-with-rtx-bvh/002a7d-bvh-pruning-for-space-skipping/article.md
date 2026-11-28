# BVH Pruning for Space Skipping
Also called: Space Skipping via BVH Pruning

## Overview
BVH Pruning for Space Skipping is a technique that accelerates volume rendering by excluding fully transparent regions from the Bounding Volume Hierarchy (BVH) during construction, thereby allowing hardware ray traversal to skip empty space and non-visible data without runtime checks [S1]. It differs from sibling methods like [Adaptive Sampling with Region Delimiters](3421-adaptive-sampling-with-region-delimiters.md) by integrating visibility culling directly into the BVH structure rather than managing traversal state, and from [Sparse Brick Set Rendering](9909-sparse-brick-set-rendering.md) by relying on transfer-function-aware pruning rather than static sparsity masks.

## How it works
**Inputs:**
*   A set of Adaptive Block-Refined (ABR) regions.
*   Precomputed scalar ranges for each ABR.
*   A transfer function mapping scalar values to opacity.

**Outputs:**
*   A BVH containing only regions with non-zero maximum opacity.

**Data Structures:**
*   **BVH Primitive:** Each leaf node represents an ABR. The primitive is defined by its bounding box. If the region is pruned, the bounding box is effectively empty (or the node is removed from the hierarchy).

**Steps:**
1.  **Compute Maximum Opacity:** For each ABR, retrieve its precomputed scalar range $[min\_val, max\_val]$. Evaluate the transfer function $T(v)$ over this range to find the maximum opacity:
    $$
    \alpha_{max} = \max_{v \in [min\_val, max\_val]} T(v)
    $$
2.  **Prune Decision:**
    *   If $\alpha_{max} = 0$, the region is fully transparent. Discard the region from the BVH construction by returning an empty box for its primitive [S1].
    *   If $\alpha_{max} > 0$, the region may contain visible data. Return the region's standard bounding box for its primitive [S1].
3.  **BVH Construction:** Build the BVH using the resulting set of primitives (valid boxes and empty boxes).

**Pitfalls:**
*   **Transfer Function Changes:** The BVH is static with respect to the transfer function. Any change to the transfer function or iso-value requires a full BVH rebuild to ensure correct pruning [S1].
*   **No Runtime Checks:** Because inactive regions are excluded from the BVH, no validity checks are needed during traversal, but this also means the BVH does not adapt to dynamic data changes without rebuild [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_pruned_bvh(abr_regions, transfer_function):
    """
    Builds a BVH where fully transparent ABRs are excluded.
    
    Args:
        abr_regions: List of ABR objects, each having:
            - scalar_range: tuple (min_val, max_val)
            - bounding_box: the spatial bounds of the region
        transfer_function: A function T(v) returning opacity for scalar v.
        
    Returns:
        BVH: A Bounding Volume Hierarchy containing only non-transparent regions.
    """
    primitives = []
    
    for region in abr_regions:
        min_val, max_val = region.scalar_range
        
        # Compute maximum opacity in the scalar range
        # Note: In practice, this might use a precomputed histogram or texture lookup
        # rather than evaluating every scalar value.
        alpha_max = 0.0
        for v in range(min_val, max_val + 1):
            opacity = transfer_function(v)
            if opacity > alpha_max:
                alpha_max = opacity
        
        # Prune if fully transparent
        if alpha_max == 0:
            # Return empty box or skip adding to primitives list
            # The source specifies "returning an empty box", which effectively
            # removes it from traversal if the BVH builder handles empty bounds.
            continue 
        else:
            primitives.append(region.bounding_box)
            
    # Construct BVH from the filtered primitives
    bvh = BVHBuilder(primitives)
    return bvh
```

## Subtopics

### [Region Clustered BVH Volume Rendering](17973-region-clustered-bvh-volume-rendering.md)
A GPU-based Empty Space Skipping (ESS) technique that uses a Bounding Volume Hierarchy (BVH) built from clustered active regions of a volume, traced via NVIDIA OptiX, to skip empty space during direct volume rendering.

## Operating characteristics
*   **Time Complexity:** BVH construction is $O(N \log N)$ where $N$ is the number of non-pruned regions. The cost of pruning is proportional to the number of regions and the cost of evaluating the transfer function over the scalar range.
*   **Memory Complexity:** The BVH size is reduced by the number of pruned regions, saving memory proportional to the number of excluded leaves and internal nodes.
*   **Performance:** Hardware ray traversal is cheap and amortized over multiple samples. Space skipping is achieved without runtime overhead. However, a full BVH rebuild on large datasets (e.g., Exajet) takes approximately 300ms on a Titan RTX or RTX 8000 GPU [S1].
*   **Strengths:** No additional structures needed for space skipping; no runtime validity checks; leverages hardware acceleration efficiently.
*   **Limitations:** Requires BVH rebuild on transfer function or iso-value changes; not suitable for dynamic transfer functions without significant latency.

## Background
This method is part of the ExaBricks rendering framework for Adaptive Block-Refined (ABR) data, which uses hardware-accelerated RTX BVH traversal [S1]. It was introduced to handle visibility-dependent regions more efficiently than static sparsity masks, avoiding the need for additional structures like those used by Morrical et al. [S1].

## Sources
[S1] Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
