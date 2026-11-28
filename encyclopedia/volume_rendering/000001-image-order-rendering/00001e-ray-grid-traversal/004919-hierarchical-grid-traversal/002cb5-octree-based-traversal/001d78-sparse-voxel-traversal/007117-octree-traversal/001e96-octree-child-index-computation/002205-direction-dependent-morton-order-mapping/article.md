# Direction-dependent Morton Order Mapping

Also called: Direction-Dependent Morton Code Permutation, Direction-dependent Morton Order

## Overview
Direction-dependent Morton Order Mapping is a remapping function that converts a standard Octree Morton code into a sorting order aligned with the near-to-far rendering direction of a specific ray [S1]. It solves the problem that standard Morton codes provide a fixed spatial traversal order, which does not guarantee correct alpha blending (back-to-front) for arbitrary viewing angles [S2, unofficial]. It sets itself apart from [Ordered Child Traversal](3008-ordered-child-traversal.md) by operating on pre-computed global Morton codes via bitwise permutation rather than dynamic tree traversal, and from [Voxel Child Address Calculation](2495-voxel-child-address-calculation.md) by focusing on global sorting keys rather than local child indexing [S1].

## How it works
The method permutes the 3-bit level codes within a Morton code based on the signs of the ray direction components $(x, y, z)$ [S2, unofficial]. This ensures that when voxels are sorted by their mapped Morton codes, they appear in the correct back-to-front order for volume rendering or rasterization [S2, unofficial].

**Inputs and Outputs:**
- Input: `octpath` (the Octree Morton code of a voxel), `rd` (ray direction vector).
- Output: `order` (the direction-dependent Morton order integer).

**Data Structures:**
- `PERMUTATION_MAPS`: A lookup table of size 8. Each entry is an array of 8 integers representing a permutation of 0-7. The index is the `raySignBits` (0-7) [S2, unofficial].
- `order_tables`: Equivalent to `PERMUTATION_MAPS`, storing the 8 permutation mappings $f^{(k)}$ [S1].

**Parameters:**
- `raySignBits`: Integer 0-7, derived from ray direction signs [S2, unofficial].
- `MAX_NUM_LEVELS` (or `maxLevel`): The maximum depth of the octree. Default value is 16 [S2, unofficial].

**Step-by-Step Method:**

1.  **Compute Ray Sign Bits:**
    Determine the sign bits from the ray direction vector $\mathbf{rd} = (rd_x, rd_y, rd_z)$ [S1].
    $$ \text{signbits} = 4 \cdot (rd_x < 0) + 2 \cdot (rd_y < 0) + (rd_z < 0) $$
    Here, $(rd_i < 0)$ evaluates to 1 if true, 0 if false [S1]. This maps the 8 possible sign combinations to indices 0-7 [S1].

2.  **Lookup Permutation Table:**
    Retrieve the specific permutation mapping $P$ corresponding to the `signbits` [S2, unofficial].
    $$ P = \text{PERMUTATION\_MAPS}[\text{signbits}] $$
    $P$ is an array where $P[v]$ gives the remapped value for input voxel index $v \in [0, 7]$ [S2, unofficial].

3.  **Iterate and Remap Bits:**
    For each octree level $i$ from $0$ to $\text{MAX\_NUM\_LEVELS} - 1$:
    a. Extract the 3-bit code for the current level from the Morton code [S2, unofficial]. The Morton code interleaves bits for x, y, and z [G]. The bits for level $i$ are located at bit positions $3i, 3i+1, 3i+2$ [G].
    $$ \text{levelBits} = (\text{octpath} \gg 3i) \ \& \ 0\text{b}111 $$
    b. Remap the extracted 3-bit code using the permutation table [S2, unofficial].
    $$ \text{remappedBits} = P[\text{levelBits}] $$
    c. Shift the remapped bits back to their original position and accumulate into the result [S2, unofficial].
    $$ \text{order} \ |= \ (\text{remappedBits} \ll 3i) $$
    d. Shift the input `octpath` right by 3 bits to process the next level [S1].

**Formula:**
The mapping function $f^{(k)}: [0 \dots 7] \to [0 \dots 7]$ is one of eight permutation mappings applied to each triplet of bits $(b_x b_y b_z)$ from the Octree Morton code [S1]:
$$ \dots b_x b_y b_z \ a_x a_y a_z \mapsto \dots f^{(k)}(b_x b_y b_z) \ f^{(k)}(a_x a_y a_z) $$
where $k$ corresponds to the ray sign bits [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ray_sign_bits(rd):
    signbits = 0
    if rd[0] < 0:
        signbits += 4
    if rd[1] < 0:
        signbits += 2
    if rd[2] < 0:
        signbits += 1
    return signbits

def direction_dependent_morton_order(octpath, signbits, max_level=16):
    # The sources do not provide the exact 8 permutation arrays.
    # Implementers must hard-code the 8x8 permutation matrix derived from the geometric requirements for each sign quadrant.
    order_tables = [[0, 1, 2, 3, 4, 5, 6, 7] for _ in range(8)]
    table = order_tables[signbits]
    order = 0
    for i in range(max_level):
        level_bits = (octpath >> (3 * i)) & 0b111
        remapped_bits = table[level_bits]
        order |= (remapped_bits << (3 * i))
    return order
```

**Edge Cases and Pitfalls:**
- **Optimization:** In practice, the loop-based mapping can be optimized to a single bitwise XOR operation instead of a loop, as the permutation for each sign combination can be represented as a fixed XOR mask applied to the Morton code [S1].
- **Precision:** The implementation uses arbitrary-precision integers (BigInt) to handle large codes, as standard fixed-width integers may overflow for deep trees [S2, unofficial].
- **Boundary Conditions:** The method uses strict `< 0` checks for ray direction components, mapping zero components to the positive sign index [S1].

## Subtopics

### [Direction-dependent Morton Order Sorting](3438-direction-dependent-morton-order-sorting.md)
This method sorts sparse voxels into a near-to-far order for each image tile to ensure correct alpha blending during rasterization. It replaces the primitive z-depth used in Gaussian Splatting with a direction-dependent Morton order, ensuring the rendering order is correct for all ray directions by duplicating voxels for each ray sign bit pattern they cover. Unlike 3DGS, which uses z-depth, this approach uses precomputed Morton codes remapped by ray direction signs to guarantee topological correctness across the entire viewing sphere.

### Proof of Correct Ordering by Induction
A proof by induction that the direction-dependent Morton order aligns with the near-to-far rendering order for all rays with $(+, +, +)$ direction signs, generalizable to other signs by flipping the scene [S1].
- **Base Case ($l=1$):** For the first Octree level, the direction-dependent Morton order of the eight voxels for $(+, +, +)$ rays is correct because the most significant bit corresponds to x, then y, then z, matching the ray direction. The ray renders $-x$ side (000-011) before $+x$ side (100-111), aligned with the MSB [S1].
- **Induction Hypothesis:** Assume $S(l)$ is true for some positive integer $l$, meaning direction-dependent Morton order is correct for all sparse voxel sets in $V_l$ [S1].
- **Induction Step:** For $V_{l+1}$, voxels in $V_l$ evolve by subdividing some voxels and removing others. Subdivision appends three zero bits to the least significant bit, preserving order. Subdivided voxels share the same most significant $3l$ bits, and their 8 children follow the base case Morton order, preserving relative order. Removing voxels does not affect remaining order [S1].

## Operating characteristics
- **Time Complexity:** $O(L)$ where $L$ is the number of octree levels, for the naive loop implementation. Can be reduced to $O(1)$ with bitwise XOR optimization [S1].
- **Memory Complexity:** $O(1)$ auxiliary space for the permutation table (8x8 integers) [S2, unofficial].
- **Strengths:** Guarantees correct sorting order for all viewing angles without per-ray depth calculations. Efficient for rasterization pipelines where voxels are sorted once per tile/ray direction [S1].
- **Limitations:** Requires duplicating voxels if they span multiple sign-bit patterns (as implied by the sorting variant) or requires computing the order per ray direction. The naive implementation is slower than a single XOR [S1].
- **When to use:** When rendering sparse voxel clouds with alpha blending where view direction varies significantly, and z-depth sorting is insufficient or expensive [S1].

## Background
Originated in the context of Sparse Voxel Rasterization for high-fidelity radiance field rendering. The method was introduced to address the limitations of z-depth sorting in Gaussian Splatting by providing a geometrically consistent sorting key for arbitrary ray directions [S1]. It builds on the concept of Morton codes (Z-order curves) for spatial indexing [S1].

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
[S2] Gaussian Splatting Alternative: WebGL Implementation of Nvidia's SVRaster, https://github.com/samuelm2/svraster-webgl, fetched 2026-10-26 (unofficial)
