# Linear Level Octree (LLO)

## Overview
Linear Level Octree (LLO) is a hierarchical data structure for volume data that stores only leaf nodes, implicitly encoding location, size, and path from root to nodes to reduce memory and processing cost [S1]. It extends the linear octree scheme by using a specific labeling system that allows efficient estimation of spatial relations between points and nodes [S1]. Unlike its parent [Linear Octree Hashing](11794-linear-octree-hashing.md), which relies on hash tables for lookup, LLO uses a deterministic coordinate-based key system that facilitates direct geometric calculations without hashing overhead [S1].

## How it works
**Inputs:** A classified volume dataset of size $2^n \times 2^n \times 2^n$ voxels, where $n$ is the resolution exponent. A user-defined intensity threshold $T \neq 0$.
**Outputs:** A list of leaf nodes, each containing a code key and eight vertex voxel values.

**Data Structures:**
Each leaf node is stored as a tuple or struct containing:
1.  **Code Key:** A tuple $(L_i, x_i, y_i, z_i)$.
    *   $L_i$: The level of the node (integer).
    *   $x_i, y_i, z_i$: The level-coordinates (integers).
2.  **Vertex Voxels:** An array of 8 values representing the intensity/color of the voxels at the eight corners of the octant.

**Coordinate System:**
The volume is placed in a coordinate system with the back-bottom-left corner at the origin $(0,0,0)$. Edges are aligned with the axes. The distance between adjacent voxels is one unit [S1].

**Formulas and Properties:**
For an arbitrary node $A$ with code key $(L, x, y, z)$:
*   **Location:** The absolute coordinates of the back-bottom-left corner voxel are:
    $$ \text{Location} = (x \cdot 2^{l_L}, y \cdot 2^{l_L}, z \cdot 2^{l_L}) $$
    where $l_L = n - L$ [S1].
*   **Size:** The side length of the node in voxels is:
    $$ \text{Size} = 2^{l_L} $$
    where $l_L = n - L$ [S1].
*   **Subnode Generation:** The subnode of $A$ at level $L_{i0} = L_i + \Delta L$ (where $\Delta L = 1, 2, \dots$) has level-coordinates:
    $$ x_{i0} = x_i \cdot 2^{\Delta L}, \quad y_{i0} = y_i \cdot 2^{\Delta L}, \quad z_{i0} = z_i \cdot 2^{\Delta L} $$
    The specific subnode index $k \in [0, 7]$ determines the offset in level-coordinates relative to the parent's base. For example, the child at index 1 has level-coordinates $(x_i \cdot 2^{\Delta L} + 2^{\Delta L-1}?, \dots)$? No, the source defines the keys directly. The source lists the keys for children at level $L_i + \Delta L$ as:
    *   Child 0: $(L_i+\Delta L, x_i 2^{\Delta L}, y_i 2^{\Delta L}, z_i 2^{\Delta L})$
    *   Child 1: $(L_i+\Delta L, x_i 2^{\Delta L}, y_i 2^{\Delta L} + 2^{\Delta L-1}?, \dots)$?
    Actually, the source provides the explicit keys for the 8 children at level $L_i+1$ (where $\Delta L=1$) relative to parent $(L_i, x_i, y_i, z_i)$:
    *   0: $(L_i+1, 2x_i, 2y_i, 2z_i)$
    *   1: $(L_i+1, 2x_i, 2y_i, 2z_i+1)$?
    The source text shows:
    $$
    \begin{aligned}
    & (L_{i0}, x_{i0}, y_{i0}, z_{i0}) && 0, 0, 0, 0 \\
    & (L_{i1}, x_{i1}, y_{i1}, z_{i1}) && 0, 1, 0, 0 \\
    & \dots
    \end{aligned}
    $$
    This notation in the source seems to describe the *offsets* or the *relative* keys if the parent was at $(0,0,0)$. However, the general formula $x_{i0} = x_i \cdot 2^{\Delta L}$ implies that the parent's level-coordinates are scaled. The specific children keys are derived by adding offsets to the scaled parent coordinates. For $\Delta L=1$, the children keys are $(L_i+1, 2x_i + b_x, 2y_i + b_y, 2z_i + b_z)$ where $b_x, b_y, b_z \in \{0, 1\}$ correspond to the octant index. The source explicitly lists the keys for the children of a node at level $L_i$ assuming the node is at origin for the offset calculation, but the general rule is scaling.
    *   Ancestor Key: The code key of an ancestor at level $L_w < L$ is:
        $$ (L_w, x/2^{L-L_w}, y/2^{L-L_w}, z/2^{L-L_w}) $$
        where the division is integer division [S1].

**Algorithm: Volume to LLO Conversion**
1.  Initialize an empty list `leaf_nodes`.
2.  Define the root node at level $L=0$ with key $(0, 0, 0, 0)$ and size $2^n$.
3.  Define a recursive function `ProcessOctant(level, x, y, z, voxel_data)`:
    a.  Calculate current size $S = 2^{n - level}$.
    b.  If $S < 2$ (i.e., size is $1 \times 1 \times 1$):
        i.  Check transparency. If the single voxel is transparent, return (do not store).
        ii. If non-transparent, create leaf node with key $(level, x, y, z)$ and vertex voxels = [voxel_data]. Add to `leaf_nodes`.
        iii. Return.
    c.  If $S = 2$:
        i.  Check leaf criteria:
            1.  Compute $Max$ and $Min$ of the 8 voxels in this $2 \times 2 \times 2$ octant.
            2.  If $Max - Min > T$, the octant is not homogeneous enough. Recursively call `ProcessOctant` for each of the 8 children (level $L+1$, size 1).
            3.  If $Max - Min \le T$, the octant is homogeneous.
        ii. Check transparency:
            1.  If all 8 voxels are transparent, return (white node, not stored).
            2.  If any voxel is non-transparent, create leaf node with key $(level, x, y, z)$ and vertex voxels = [8 corner voxels]. Add to `leaf_nodes`.
        iii. Return.
    d.  If $S > 2$:
        i.  Check leaf criteria:
            1.  Compute $Max$ and $Min$ of all voxels in the current octant.
            2.  If $Max - Min > T$, split the octant into 8 children. For each child $k \in [0,7]$:
                *   Calculate child level $L_{child} = level + 1$.
                *   Calculate child level-coordinates:
                    *   $x_{child} = 2x + (k \gg 2) \& 1$
                    *   $y_{child} = 2y + (k \gg 1) \& 1$
                    *   $z_{child} = 2z + k \& 1$
                    *   *(Note: The bit order of $k$ mapping to x,y,z is not explicitly defined in the source text snippet, but standard Morton/Z-order or similar is implied. The source lists keys 0-7 with offsets 0/1. We assume standard binary decomposition where $k$'s bits determine the offset.)*
                *   Extract the sub-volume of voxels for the child.
                *   Recursively call `ProcessOctant(L_{child}, x_{child}, y_{child}, z_{child}, child_voxels)`.
            3.  If $Max - Min \le T$, proceed to transparency check.
        ii. Check transparency:
            1.  If all voxels are transparent, return.
            2.  If non-transparent, create leaf node with key $(level, x, y, z)$ and vertex voxels = [8 corner voxels of the current octant]. Add to `leaf_nodes`.
        iii. Return.
4.  Call `ProcessOctant(0, 0, 0, 0, full_volume_voxels)`.

**Edge Cases and Pitfalls:**
*   **Transparent Octants:** Treated as "white" nodes and are not stored in the structure [S1].
*   **Non-Transparent Octants:** Treated as "black" nodes and are stored [S1].
*   **Threshold $T$:** Must be non-zero. If $T=0$, the condition $Max - Min = T$ is extremely strict, potentially leading to excessive subdivision or no leaves if data is perfectly constant [S1].
*   **Smallest Octant:** The recursion stops at size $2 \times 2 \times 2$ for the homogeneous check, but individual voxels (size 1) are also handled as base cases for transparency [S1].

## Operating characteristics
*   **Memory:** Only leaf nodes are stored, reducing memory consumption compared to full octrees [S1]. The implicit encoding of location and size further reduces overhead [S1].
*   **Processing:** Fewer nodes to process during rendering because internal nodes are not stored [S1].
*   **Spatial Queries:** Easier to estimate relations between arbitrary spatial points and LLO nodes with lower computing cost due to the explicit coordinate formulas [S1].
*   **Performance:** Specific measured performance numbers are not given in the sources. The method is designed for PC-based medical simulation [S1].

## Background
*   **Origin:** The Linear Level Octree (LLO) was introduced in the paper "Shear-Warp Volume Rendering Algorithms Using Linear Level Octree for PC-Based Medical Simulation" [S1].
*   **Development:** It is an extension of the linear octree scheme, inheriting its advantages while providing a labeling system that simplifies spatial calculations [S1].
*   **Application:** Primarily used in volume rendering, specifically for medical simulation on PC-based systems [S1].

## Sources
[S1] Shear-Warp Volume Rendering Algorithms Using Linear Level Octree for PC-Based Medical Simulation, https://doi.org/10.1007/3-540-45787-9_76, fetched 2026-10-26
