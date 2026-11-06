# Distribution over Levels (MIP-mapping)
Also called: Octree MIP-map filtering, Three-pass MIP-mapping filtering

## Overview
Distribution over Levels propagates radiance values from leaf nodes to internal nodes of a sparse voxel octree to create a MIP-map pyramid for cone tracing [S1, S2]. It distinguishes itself from standard isotropic downsampling by using a three-pass strategy to balance thread cost and explicitly handling shared boundary voxels in a $3 \times 3 \times 3$ brick structure [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse voxel octree with $n$ levels, where leaf nodes contain initial radiance values (e.g., from photon splatting) [S1].
*   **Output:** An octree where internal nodes at levels $1$ to $n-1$ contain filtered radiance values derived from their children [S1].

**Data Structures:**
*   **Octree Node:** Contains a $3 \times 3 \times 3$ brick of vertex-centered voxels [S1].
*   **Voxel:** Stores radiance (energy) and directional information (e.g., Gaussian-Lobes) [S2].
*   **Multiplicity:** Each voxel in a $3 \times 3 \times 3$ brick is shared with up to 8 neighboring bricks, requiring inverse multiplicity weighting [S1].

**Parameters:**
*   **Kernel:** $3 \times 3 \times 3$ Gaussian weighting kernel. This is an optimal reconstruction filter [S1].
*   **Levels:** $n-1$ filtering steps for an octree of $n$ levels [S1].

**Steps:**
1.  **Initialization:** Build the octree structure and write initial values into leaf nodes [S1].
2.  **Iterative Filtering:** For each level from $n-1$ down to $1$:
    a.  Launch threads to compute filtered values for the current level based on the previous (finer) level [S1].
    b.  Execute three distinct passes to balance thread cost:
        i.  **Pass 1 (Center):** Compute the center voxel using the 27 involved voxel values from the lower level [S1].
        ii. **Pass 2 (Face Centers):** Compute half of the filtered response for the 6 voxels situated in the center of the node's faces, involving 18 voxel values from the lower level [S1].
        iii. **Pass 3 (Corners):** Launch threads for the 8 corner voxels, computing a partial filtering from a single octant (9 values) [S1].
    c.  **Value Transfer:** Apply the "Value Transfer to Neighboring Bricks" step to sum partial values across adjacent bricks, finalizing the filtered result [S1].
3.  **Weighting:** When averaging values from the previous level, weight each voxel by the inverse of its multiplicity because boundary voxels reappear in neighboring bricks [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distribute_over_levels(octree_levels):
    # octree_levels: list of lists, index 0 is leaves, index n-1 is root
    n_levels = len(octree_levels)
    # Compute n-1 internal levels from leaves up to root
    for level in range(1, n_levels):
        current_nodes = octree_levels[level]

        # Pass 1: Center voxel (index 1,1,1)
        for node in current_nodes:
            source_voxels = [v for child in node.children for v in child.voxels]
            node.voxels[1, 1, 1].value = weighted_average(source_voxels)

        # Pass 2: Face centers (6 voxels)
        face_indices = [(1, 1, 0), (1, 1, 2), (1, 0, 1), (1, 2, 1), (0, 1, 1), (2, 1, 1)]
        for node in current_nodes:
            for f in face_indices:
                source_voxels = [v for child in node.children for v in child.voxels if not is_in_opposite_octant(v, f)]
                node.voxels[f].value = 0.5 * weighted_average(source_voxels)

        # Pass 3: Corners (8 voxels)
        corner_indices = [(x, y, z) for x in (0, 2) for y in (0, 2) for z in (0, 2)]
        for node in current_nodes:
            for c in corner_indices:
                child_idx = c[0] * 4 + c[1] * 2 + c[2]
                source_voxels = node.children[child_idx].voxels
                node.voxels[c].value = weighted_average(source_voxels)

    # Finalize by summing partial values across brick boundaries
    # Apply Value Transfer to Neighboring Bricks (see parent article)
    apply_value_transfer(octree_levels)

def weighted_average(voxels):
    total_w = 0.0
    total_v = 0.0
    for v in voxels:
        w = 1.0 / v.multiplicity
        total_v += v.value * w
        total_w += w
    return total_v / total_w

def is_in_opposite_octant(voxel, face_idx):
    # Returns True if voxel coordinate is in the octant opposite to the face center
    # face_idx maps to a normal direction; opposite octant has coordinate 0 if face is at 2, etc.
    # Simplified coordinate check
    return False

def apply_value_transfer(octree_levels):
    # Sums partial values across adjacent bricks to finalize filtering
    for level_nodes in octree_levels:
        for node in level_nodes:
            for v in node.voxels:
                v.value += sum(n.value for n in v.neighbors)
```

**Edge Cases and Pitfalls:**
*   **Sparse Octrees:** If the octree is sparse, launching threads for all nodes (including empty ones) is inefficient. The method assumes a complete structure for the base algorithm; for sparse structures, thread launch must be optimized (e.g., per light-view-map pixel) to avoid filtering zero values [S1].
*   **Shared Voxels:** Failure to account for multiplicity (shared boundary voxels) leads to incorrect energy conservation. The Gaussian kernel and inverse multiplicity weighting are critical [S1].

## Subtopics

### Gaussian-Lobes Representation for Radiance Filtering
A compact representation used to store the filtered distribution of incoming light directions in the octree MIP-map levels. It allows for view-dependent treatment of the Normal Distribution Function (NDF) and Bidirectional Reflectance Distribution Function (BRDF) [S2]. Instead of storing single scalar values, it stores directional distributions, enabling more accurate cone tracing and indirect illumination calculation [S2].

## Operating characteristics
*   **Performance:** The three-pass approach balances thread cost, ensuring that all threads have roughly the same computational load, which improves GPU scheduling efficiency [S1].
*   **Not given in the sources:** Time complexity, memory complexity, strengths, limitations, and usage recommendations.

## Background
The method originates from the work on Interactive Indirect Illumination Using Voxel Cone Tracing by Jaroslav Krivánek et al. (2011) [S1, S2]. The three-pass MIP-mapping strategy was introduced to address the imbalance in thread costs and the redundancy of computing shared boundary voxels multiple times [S1]. The Gaussian-Lobes representation was introduced to handle directional lighting data more accurately than scalar values [S2].

## Sources
[S1] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
[S2] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.02063.x, fetched 2026-10-26
