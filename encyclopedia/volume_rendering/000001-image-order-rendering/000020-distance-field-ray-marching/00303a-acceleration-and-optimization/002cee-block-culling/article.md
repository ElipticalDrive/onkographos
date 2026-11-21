# Block Culling

Also called: RT+Cull

## Overview
Block culling is an optimization technique for volume rendering that reduces the time required to reach content values inside RGBA volumes by skipping blocks that do not contribute to the visible result [S1, unofficial]. It is particularly effective when sparsity in the volume data results from runtime parameters, such as a narrower allowed density range, rather than inherent data sparsity [S1, unofficial]. Unlike [Beam Optimization](932-beam-optimization.md) which focuses on ray coherence, or [Bounding Shape Optimization](7750-bounding-shape-optimization.md) which uses geometric bounds, block culling operates on the hierarchical block structure to prune traversal paths based on content relevance [S1, unofficial].

## How it works
Block culling integrates with hardware ray tracing (RT) to evaluate whether a block contains relevant content before performing expensive ray marching or traversal steps.

**Inputs and Outputs:**
- **Input:** A hierarchical volume data structure (e.g., bricks or blocks), a user-defined content range (density/opacity thresholds), and ray intersection data.
- **Output:** A rendered image with reduced computation time by skipping irrelevant blocks.

**Data Structures:**
The method relies on the existing block hierarchy. Each block typically stores metadata indicating its content range (min/max density or opacity). If this metadata is not explicitly stored, it must be computed or approximated [G].

**Steps:**
1. **Ray-Block Intersection:** Determine if a ray intersects a block in the hierarchy.
2. **Content Range Check:** Compare the block's stored content range against the user-requested content range.
3. **Culling Decision:**
   - If the block's content range does not overlap with the user-requested range, the block is culled (skipped).
   - If there is an overlap, the block is processed (traversed or ray-marched).
4. **Traversal/Rendering:** Proceed with standard ray tracing or block walking for non-culled blocks.

**Parameters:**
- **User-requested content range:** The density/opacity interval of interest. Typical values depend on the application (e.g., [0.1, 0.9] for visible structures).
- **Block content range:** Precomputed min/max values for each block.

**Edge Cases and Pitfalls:**
- **No Culling Benefit:** If the user-requested content range is greater than or equal to the actual content range of the volume, block culling provides no significant performance gain [S1, unofficial].
- **Performance Regression:** The method does not cause significant performance regression when it cannot impact performance (i.e., when no blocks are culled) [S1, unofficial].
- **Sparse Content:** If the volume is inherently too sparse, block culling may not be as effective as block walking, and the overhead of checking ranges might outweigh benefits if not optimized [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def block_culling(ray, block_hierarchy, user_content_range):
    """
    ray: Ray origin and direction
    block_hierarchy: Root block of the volume hierarchy
    user_content_range: Tuple (min_density, max_density)
    """
    if block_hierarchy is None:
        return None

    # Check if ray intersects block
    if not ray_intersects_block(ray, block_hierarchy.bounds):
        return None

    # Check if block contains relevant content
    if not ranges_overlap(block_hierarchy.content_range, user_content_range):
        return None  # Cull this block

    # If leaf block, perform ray marching or shading
    if block_hierarchy.is_leaf:
        return ray_march_block(ray, block_hierarchy.data, user_content_range)

    # Recurse into children
    results = []
    for child in block_hierarchy.children:
        result = block_culling(ray, child, user_content_range)
        if result is not None:
            results.append(result)

    return composite_results(results)

def ranges_overlap(range_a, range_b):
    return range_a.min <= range_b.max and range_b.min <= range_a.max
```

## Subtopics

### [Edit Culling](16389-edit-culling.md)
Edit culling is a technique used during the construction of Signed Distance Field (SDF) bricks to reduce the number of edits evaluated per brick, thereby reducing construction latency. It determines which edits affect a sub-brick based on the principle that a sub-brick is entirely encapsulated by its parent, ensuring that no edit affecting the child is missed while culling those that do not. It sets itself apart from sibling methods by focusing on the hierarchical relationship between parent and child bricks to minimize redundant distance evaluations during the critical construction phase.

### Block Walking with Culling
Block walking with culling combines block walking and block culling algorithms to reduce the time to reach content values inside RGBA volumes [S1, unofficial]. In block traversal mode, 'RT+Walk+Cull' achieved 5.543ms on Fish (a) and 6.945ms on Fish (c) [S1, unofficial]. In blend mode, 'RT+Walk+Cull' achieved 107.725ms on Fish (a) and 11.892ms on Fish (c) [S1, unofficial]. The combination results in the best performance on average compared to using either technique alone or standard ray tracing [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Time:** Reduces rendering time by skipping irrelevant blocks. Performance gains are most significant when sparsity results from runtime parameters [S1, unofficial].
- **Memory:** Requires storing content range metadata for each block. No additional significant memory overhead is noted [S1, unofficial].

**Measured Performance:**
- **Block Traversal Mode:**
  - Fish (c): RT+Cull achieved 4.823ms vs 7.375ms for RT [S1, unofficial].
  - Fish (a): RT+Cull achieved 9.740ms vs 9.745ms for RT [S1, unofficial].
  - Chip (d): RT+Cull achieved 2.309ms vs 2.310ms for RT [S1, unofficial].
  - Rock (e): RT+Cull achieved 7.949ms vs 7.949ms for RT [S1, unofficial].
- **Blend Mode:**
  - Fish (c): RT+Cull achieved 8.079ms vs 10.798ms for RT [S1, unofficial].
  - Fish (a): RT+Cull achieved 113.780ms vs 113.983ms for RT [S1, unofficial].
  - Chip (d): RT+Cull achieved 15.132ms vs 15.098ms for RT [S1, unofficial].

**Scaling:**
- Performance gains are consistent across different datasets (Fish, Chip, Rock) when sparsity is present [S1, unofficial].
- No significant performance regression when no blocks are culled [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Effective for runtime-adjustable density ranges; no significant penalty when ineffective [S1, unofficial].
- **Limitations:** Less effective when content is inherently too sparse (block walking may be better); requires precomputed content ranges [S1, unofficial].

**When to Use:**
- Use when sparsity is due to runtime parameters (e.g., narrow density ranges) rather than inherent data sparsity [S1, unofficial].
- Use in conjunction with block walking for optimal performance [S1, unofficial].

**When Not to Use:**
- Avoid when the user-requested content range covers the entire actual content range, as no culling benefit is realized [S1, unofficial].
- Avoid when content is inherently too sparse, as block walking may be more effective [S1, unofficial].

## Background
Block culling was introduced in the context of sparse volume rendering using hardware ray tracing [S1, unofficial]. It was developed to address the performance challenges of rendering large, sparse RGBA volumes by leveraging the hierarchical block structure to skip irrelevant data [S1, unofficial]. The technique is part of a broader set of optimizations including block walking and beam optimization [S1, unofficial].

## Sources
[S1] Sparse Volume Rendering using Hardware Ray Tracing and Block ..., https://gpuopen.com/download/SA2021_BlockWalk.pdf, fetched 2026-10-26 (unofficial)
