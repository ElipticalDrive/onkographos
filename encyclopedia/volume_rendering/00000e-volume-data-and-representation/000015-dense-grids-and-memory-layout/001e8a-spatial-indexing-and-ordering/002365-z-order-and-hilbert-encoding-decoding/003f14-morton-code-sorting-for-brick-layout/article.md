# Morton Code Sorting for Brick Layout

Also called: Brick Building (Morton Code Sorting), Morton Code Sorting

## Overview
Morton Code Sorting for Brick Layout organizes sub-bricks into a global buffer to maintain spatial coherence during volume rendering [S1, unofficial]. It sorts sub-bricks according to their Z-order (Morton) codes within hierarchical groups to ensure the buffer remains compact and spatially coherent as construction proceeds one level at a time [S1, unofficial]. It sets itself apart from its siblings by using an enumeration sort that leverages a 1:1 thread-to-sub-brick mapping to guarantee sorted order within each group [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A group of sub-bricks, each identified by its local 3D index $(x, y, z)$ within its parent brick [S1, unofficial].
- Output: Sub-bricks placed into a global output buffer, ordered by their Morton codes relative to the group's local order [S1, unofficial].

**Data Structures:**
- **Sub-brick:** Contains its local 3D index $(x, y, z)$ within the parent brick [S1, unofficial].
- **Global Buffer:** A contiguous array that stores the sorted sub-bricks [S1, unofficial].
- **Thread Mapping:** A 1:1 mapping between execution threads and sub-bricks within a group [S1, unofficial].

**Formulas:**
- Morton code calculation: $M(x, y, z) = \sum_{i=0}^{k-1} (x_i 2^{3i} + y_i 2^{3i+1} + z_i 2^{3i+2})$, where $x_i, y_i, z_i$ are the $i$-th bits of the coordinates and $k$ is the bit depth [G].
- Enumeration sort rank: $r_i = \sum_{j=0}^{N-1} \mathbb{I}(M_j < M_i) + \mathbb{I}(M_j = M_i \land j < i)$, where $N$ is the group size and $\mathbb{I}$ is the indicator function [G].

**Parameters:**
- Maximum number of sub-bricks per parent: 64 [S1, unofficial].

**Steps:**
1. **Calculate Morton Codes:** For each sub-brick, compute its Morton code from its 3D index $(x, y, z)$ by interleaving the bits of the coordinates [G].
2. **Enumerate Sort:** Perform an enumeration sort within the group of sub-bricks [S1, unofficial]. Each thread calculates the sorted index of one brick by counting how many other bricks in the group have a smaller Morton code, or an equal code with a lower original index [S1, unofficial].
3. **Place in Buffer:** Place each sorted sub-brick into the global buffer at the calculated sorted index, relative to the group's local order [S1, unofficial]. This ensures the global buffer remains compact and spatially coherent [S1, unofficial].
4. **Iterate:** Repeat steps 1–3 for each level of the brick hierarchy, guaranteeing the entire buffer is sorted as construction proceeds one level at a time [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def morton_sort_bricks(sub_bricks, global_buffer, group_offset):
    n = len(sub_bricks)
    morton_codes = []
    for brick in sub_bricks:
        morton_codes.append(interleave_bits(brick.x, brick.y, brick.z))
    
    ranks = [0] * n
    for i in range(n):
        rank = 0
        for j in range(n):
            if morton_codes[j] < morton_codes[i]:
                rank += 1
            elif morton_codes[j] == morton_codes[i] and j < i:
                rank += 1
        ranks[i] = rank
    
    for i in range(n):
        global_buffer[group_offset + ranks[i]] = sub_bricks[i]

def interleave_bits(x, y, z):
    m = 0
    for i in range(10):
        m |= ((x >> i) & 1) << (3 * i)
        m |= ((y >> i) & 1) << (3 * i + 1)
        m |= ((z >> i) & 1) << (3 * i + 2)
    return m
```

**Edge Cases and Pitfalls:**
- **Ties in Morton Codes:** If two sub-bricks share the same Morton code, a stable sort or secondary key (like original index) ensures deterministic placement [G].
- **Bit Width:** The Morton code calculation must handle the maximum coordinate range without overflow [G].
- **Group Size:** The method assumes a maximum of 64 sub-bricks per group to keep the enumeration sort efficient [S1, unofficial].

## Subtopics

### Bricking and Z-Order Layout
Bricking and Z-Order Layout orders vertices into a fixed number of small cubes (bricking) or arranges them using a Lebesgue space-filling curve (Z-order) for better sequential access [S2]. It generates the arrangement by creating an out-of-core octree of vertices and traversing them in-order [S2]. Compared to spatial sorting, it provides more contiguous mesh portions on average but exhibits a much larger front width and span [S2]. It is suitable for meshes with a dominant principal direction but less effective for shapes like spheres [S2].

## Operating characteristics
**Time Complexity:**
- Sorting within each group: $O(N^2)$ for enumeration sort, where $N$ is the number of sub-bricks per group (max 64) [S1, unofficial].
- Overall: Linear with respect to the number of bricks, as sorting is performed locally per group [G].

**Memory Complexity:**
- Not given in the sources.

**Performance:**
- Improves cache coherency during rendering by ensuring nearby bricks are adjacent in memory [S1, unofficial].
- Effective for making use of cache during rendering, as nearby threads access adjacent bricks or a thread accesses a neighbouring brick when exiting a brick [S1, unofficial].

**Strengths:**
- Guarantees spatial coherence in the global buffer [S1, unofficial].
- Simple to implement with enumeration sort for small group sizes [S1, unofficial].

**Limitations:**
- Limited to a maximum of 64 sub-bricks per group for efficient sorting [S1, unofficial].
- May not be optimal for meshes without a dominant principal direction [S2].

**When to Use:**
- When spatial coherence and cache efficiency are critical for volume rendering performance [S1, unofficial].
- When the brick hierarchy has a manageable number of sub-bricks per parent [S1, unofficial].

**When Not to Use:**
- For meshes with complex shapes like spheres, where other layouts may be more effective [S2].
- When the number of sub-bricks per group exceeds 64, making enumeration sort inefficient [S1, unofficial].

## Background
Morton Code Sorting for Brick Layout was introduced in the context of real-time rendering and construction of signed distance fields [S1, unofficial]. It builds on the concept of Z-order encoding for spatial data organization [S1, unofficial]. The method is designed to optimize memory access patterns in volume rendering by leveraging the spatial locality of bricks [S1, unofficial].

## Sources
[S1] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
[S2] Streaming Simplification of Tetrahedral Meshes, https://doi.org/10.1109/tvcg.2007.21, fetched 2026-11-04
