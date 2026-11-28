# Linear Level Octree (LLO)


## Overview
Linear Level Octree (LLO) is a hierarchical data structure for volume data that stores only leaf nodes, implicitly encoding location, size, and path from root to nodes to reduce memory and processing cost [S1]. It extends the linear octree scheme by using a deterministic coordinate-based labeling system that allows efficient estimation of spatial relations between arbitrary points and nodes [S1]. Unlike its parent [Linear Octree Hashing](11794-linear-octree-hashing.md), which relies on hash tables for lookup, LLO uses explicit coordinate formulas that facilitate direct geometric calculations without hashing overhead [S1].

## How it works
**Inputs:** A classified volume dataset of size $2^n \times 2^n \times 2^n$ voxels, where $n$ is the resolution exponent [S1]. A user-defined intensity threshold $T \neq 0$ [S1].
**Outputs:** A list of leaf nodes, each containing a code key and eight vertex voxel values [S1].

**Data Structures:**
Each stored leaf node contains two fields [S1]:
1. **Code Key:** A tuple $(L_i, x_i, y_i, z_i)$, where $L_i$ is the level and $x_i, y_i, z_i$ are the level-coordinates [S1].
2. **Vertex Voxels:** An array of 8 values representing the intensity of the voxels at the eight corners of the octant [S1].

**Coordinate System:**
The volume is placed in a coordinate system with the back-bottom-left corner at the origin $(0,0,0)$ and edges aligned with the axes [S1]. The distance between adjacent voxels is defined as one unit [S1].

**Formulas and Properties:**
For an arbitrary node $A$ with code key $(L, x, y, z)$:
* **Location:** The absolute coordinates of the back-bottom-left corner voxel are $\text{Location} = (x \cdot 2^{l_L}, y \cdot 2^{l_L}, z \cdot 2^{l_L})$, where $l_L = n - L$ [S1].
* **Size:** The side length of the node in voxels is $\text{Size} = 2^{l_L}$, where $l_L = n - L$ [S1].
* **Subnode Generation:** A subnode of $A$ at level $L_{i0} = L_i + \Delta L$ (where $\Delta L = 1, 2, \dots$) has base level-coordinates $x_{i0} = x_i \cdot 2^{\Delta L}$, $y_{i0} = y_i \cdot 2^{\Delta L}$, $z_{i0} = z_i \cdot 2^{\Delta L}$ [S1]. The eight children at level $L_i + \Delta L$ are distinguished by offsets $(b_x, b_y, b_z) \in \{0, 1\}^3$, yielding keys $(L_i + \Delta L, x_i \cdot 2^{\Delta L} + b_x, y_i \cdot 2^{\Delta L} + b_y, z_i \cdot 2^{\Delta L} + b_z)$ [S1].
* **Ancestor Key:** The code key of an ancestor at level $L_w < L$ is $(L_w, \lfloor x/2^{L-L_w} \rfloor, \lfloor y/2^{L-L_w} \rfloor, \lfloor z/2^{L-L_w} \rfloor)$, where the division is integer division [S1].

**Algorithm: Volume to LLO Conversion**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_llo(volume_data, n, T):
    leaf_nodes = []
    
    def process_octant(level, x, y, z, voxels):
        size = 2**(n - level)
        max_val = max(voxels)
        min_val = min(voxels)
        homogeneous = (max_val - min_val <= T)
        transparent = all(v == 0 for v in voxels) # [G] Assumes 0 represents transparent background

        if homogeneous and not transparent:
            # Create leaf node with code key and 8 corner voxels
            corners = []
            for i in (0, size - 1):
                for j in (0, size - 1):
                    for k in (0, size - 1):
                        corners.append(voxels[i * size * size + j * size + k])
            leaf_nodes.append({"key": (level, x, y, z), "vertex_voxels": corners})
            return

        if size == 2:
            # Smallest octant size reached; stop recursion
            return

        # Split into 8 children
        for k in range(8):
            bx, by, bz = (k >> 2) & 1, (k >> 1) & 1, k & 1
            child_x, child_y, child_z = 2 * x + bx, 2 * y + by, 2 * z + bz
            step = size // 2
            child_voxels = []
            for i in range(step):
                for j in range(step):
                    for p in range(step):
                        child_voxels.append(voxels[(i * step + bz) * size * size + (j * step + by) * size + (p * step + bx)])
            process_octant(level + 1, child_x, child_y, child_z, child_voxels)

    process_octant(0, 0, 0, 0, volume_data)
    return leaf_nodes
```

**Edge Cases and Pitfalls:**
* **Transparent Octants:** Treated as "white" nodes and are not stored [S1].
* **Non-Transparent Octants:** Treated as "black" nodes and are stored [S1].
* **Threshold $T$:** Must be non-zero; the homogeneity criterion requires $Max - Min \le T$ [S1].
* **Smallest Octant:** The recursion stops at size $2 \times 2 \times 2$ voxels; individual voxels (size 1) are not generated as separate leaves [S1].

## Operating characteristics
* **Memory:** Only leaf nodes are stored, which greatly reduces memory consumption compared to full octrees [S1]. The implicit encoding of location and size further reduces structural overhead [S1].
* **Processing:** Internal nodes are not stored, reducing the number of nodes required to be processed during rendering [S1].
* **Spatial Queries:** The explicit coordinate formulas allow easier estimation of relations between arbitrary spatial points and LLO nodes with lower computing cost [S1].
* **Performance:** Specific measured performance numbers are not given in the sources [S1]. The method is applied to medical simulation [S1].

## Background
* **Origin:** The Linear Level Octree (LLO) was introduced in the paper "Shear-Warp Volume Rendering Algorithms Using Linear Level Octree for PC-Based Medical Simulation" [S1].
* **Development:** It extends the linear octree scheme, inheriting its advantages while providing a labeling system that simplifies spatial calculations [S1].
* **Application:** It is used in volume rendering, specifically within medical simulation workflows [S1].

## Sources
[S1] Shear-Warp Volume Rendering Algorithms Using Linear Level Octree for PC-Based Medical Simulation, https://doi.org/10.1007/3-540-45787-9_76, fetched 2026-10-26
