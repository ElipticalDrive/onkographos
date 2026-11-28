# Octree Voxel-Triangle Intersection Test

Also called: SAT Box-Triangle Test, Virtual Grid Voxel-Culling

## Overview
A method to determine if a triangle intersects a voxel within a virtual uniform grid defined by an octree, used for culling or traversal in volume rendering [S1, unofficial]. It implements the Separating Axis Theorem (SAT) to test intersection between a triangle and an axis-aligned bounding box (cube) [S1, unofficial]. It serves as the geometric predicate in [Sparse Ray-Octree Intersection (DECIDE)](3006-sparse-ray-octree-intersection-decide.md), replacing generic ray-box tests with a direct voxel-triangle check to handle high-frequency geometry or thin primitives that might be missed by coarse ray stepping [S1, unofficial].

## How it works
**Inputs:**
- Triangle vertices $v_0, v_1, v_2$ (3D points).
- Voxel indices $(x, y, z)$ within the virtual uniform grid.
- Octree level $L$.
- Virtual grid dimension $D_{grid}$ (total number of voxels along one axis at level 0).
- Octree minimum bounds $O_{min} = (O_{minX}, O_{minY}, O_{minZ})$.
- Triangle bounding box $T_{BB}$ with minimum $T_{BBmin}$ and maximum $T_{BBmax}$.

**Outputs:**
- Boolean: `true` if the triangle intersects the voxel, `false` otherwise.

**Data Structures:**
- `DoublePoint`: A struct representing a 3D vector with fields `x`, `y`, `z` of type `double`.
  - Static constants: `UnitX = (1.0, 0.0, 0.0)`, `UnitY = (0.0, 1.0, 0.0)`, `UnitZ = (0.0, 0.0, 1.0)`.
  - Methods:
    - `Sub(p)`: Returns $(x-p.x, y-p.y, z-p.z)$.
    - `Dot(p)`: Returns $x \cdot p.x + y \cdot p.y + z \cdot p.z$.
    - `Cross(p)`: Returns $(y \cdot p.z - p.y \cdot z, z \cdot p.x - x \cdot p.z, x \cdot p.y - y \cdot p.x)$.

**Steps:**

1.  **Calculate Grid Resolution and Multiplier:**
    $$ R = 2^L $$
    $$ M = \frac{1}{R} $$

2.  **Calculate Voxel Bounds:**
    $$ V_{minX} = (x \cdot D_{grid}) \cdot M + O_{minX} $$
    $$ V_{minY} = (y \cdot D_{grid}) \cdot M + O_{minY} $$
    $$ V_{minZ} = (z \cdot D_{grid}) \cdot M + O_{minZ} $$
    $$ V_{maxX} = ((x + 1) \cdot D_{grid}) \cdot M + O_{minX} $$
    $$ V_{maxY} = ((y + 1) \cdot D_{grid}) \cdot M + O_{minY} $$
    $$ V_{maxZ} = ((z + 1) \cdot D_{grid}) \cdot M + O_{minZ} $$

3.  **Calculate Voxel Center and Half-Dimension:**
    $$ V_{dim} = V_{maxX} - V_{minX} $$
    $$ V_{half} = V_{dim} \cdot 0.5 $$
    $$ C_x = V_{minX} + V_{half} $$
    $$ C_y = V_{minY} + V_{half} $$
    $$ C_z = V_{minZ} + V_{half} $$

4.  **Calculate Triangle Bounding Box Center:**
    $$ T_{cx} = T_{BBmin}.x + (T_{BBmax}.x - T_{BBmin}.x) \cdot 0.5 $$
    $$ T_{cy} = T_{BBmin}.y + (T_{BBmax}.y - T_{BBmin}.y) \cdot 0.5 $$
    $$ T_{cz} = T_{BBmin}.z + (T_{BBmax}.z - T_{BBmin}.z) \cdot 0.5 $$

5.  **Fast Bounding Box Cull (Separating Axis on Centers):**
    Calculate distance components between centers, scaled by 2:
    $$ D_x = (C_x - T_{cx}) \cdot 2.0 $$
    $$ D_y = (C_y - T_{cy}) \cdot 2.0 $$
    $$ D_z = (C_z - T_{cz}) \cdot 2.0 $$

    Calculate triangle bounding box dimensions:
    $$ T_{dimX} = T_{BBmax}.x - T_{BBmin}.x $$
    $$ T_{dimY} = T_{BBmax}.y - T_{BBmin}.y $$
    $$ T_{dimZ} = T_{BBmax}.z - T_{BBmin}.z $$

    If any axis separates the boxes:
    $$ \text{if } D_x > V_{dim} + T_{dimX} \text{ or } D_y > V_{dim} + T_{dimY} \text{ or } D_z > V_{dim} + T_{dimZ} $$
    Return `false`.

6.  **Vertex Containment Check:**
    If any triangle vertex lies within the voxel bounds:
    $$ \text{if } (V_{minX} \le v_0.x \le V_{maxX} \land V_{minY} \le v_0.y \le V_{maxY} \land V_{minZ} \le v_0.z \le V_{maxZ}) $$
    $$ \text{or } (V_{minX} \le v_1.x \le V_{maxX} \land V_{minY} \le v_1.y \le V_{maxY} \land V_{minZ} \le v_1.z \le V_{maxZ}) $$
    $$ \text{or } (V_{minX} \le v_2.x \le V_{maxX} \land V_{minY} \le v_2.y \le V_{maxY} \land V_{minZ} \le v_2.z \le V_{maxZ}) $$
    Return `true`.

7.  **Detailed SAT Box-Triangle Intersection:**
    Call `isectboxtri(C_x, C_y, C_z, V_{half}, v_0, v_1, v_2)`.

    **Sub-routine `isectboxtri(centerX, centerY, centerZ, boxHalfDim, v0, v1, v2)`:**
    1.  Generate 8 box vertices relative to center:
        $$ P_i = (centerX \pm boxHalfDim, centerY \pm boxHalfDim, centerZ \pm boxHalfDim) $$
        for all combinations of signs.
    2.  Store triangle vertices: $T_0=v_0, T_1=v_1, T_2=v_2$.
    3.  **Test Box Axes (X, Y, Z):**
        For each axis $A \in \{UnitX, UnitY, UnitZ\}$:
        If `!overlap(P, T, A)`, return `false`.
    4.  **Test Triangle Normal:**
        $$ E_1 = T_1 - T_0 $$
        $$ E_2 = T_2 - T_1 $$
        $$ N = E_1 \times E_2 $$
        If `!overlap(P, T, N)`, return `false`.
    5.  **Test Edge Cross Products:**
        $$ E_3 = T_0 - T_2 $$
        For each box axis $A \in \{UnitX, UnitY, UnitZ\}$ and each triangle edge $E \in \{E_1, E_2, E_3\}$:
        $$ L = A \times E $$
        If `!overlap(P, T, L)`, return `false`.
    6.  Return `true`.

    **Sub-routine `overlap(points, axis)`:**
    1.  Calculate min projection of `points` onto `axis`:
        $$ min_P = \min_{p \in points} (p \cdot axis) $$
    2.  Calculate max projection of `points` onto `axis`:
        $$ max_P = \max_{p \in points} (p \cdot axis) $$
    3.  Calculate min projection of `triangle` onto `axis`:
        $$ min_T = \min_{t \in triangle} (t \cdot axis) $$
    4.  Calculate max projection of `triangle` onto `axis`:
        $$ max_T = \max_{t \in triangle} (t \cdot axis) $$
    5.  If $min_P > max_T$ or $max_P < min_T$, return `false`.
    6.  Return `true`.

**Parameters:**
- `nTHREADS_TRAV_KERNEL`: 128 [S1, unofficial].
- `nTHREADS_DRAW_KERNEL`: 128 [S1, unofficial].
- `nTHREADS_DRAW_SHADOW_KERNEL`: 192 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The method uses `double` for coordinate calculations to maintain precision during grid resolution scaling [S1, unofficial].
- **Degenerate Triangles:** If the triangle is degenerate (area zero), the normal $N$ will be zero. The cross products with box axes will also be zero. The `overlap` test on a zero vector axis is undefined or trivially true/false depending on implementation; standard SAT requires non-zero axes. The source does not explicitly handle degenerate triangles, implying they are either pre-culled or assumed valid [S1, unofficial].
- **Boundary Conditions:** The vertex containment check uses inclusive bounds ($\le$) [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel-triangle pair. The SAT test involves a fixed number of dot products and comparisons (3 box axes + 1 normal + 9 edge cross products = 13 axes).
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Performance:** The method is optimized for GPU execution within CUDA kernels, utilizing shared memory for job queues and texture objects for depth buffers [S1, unofficial].
- **Strengths:** More accurate than ray-box intersection for thin triangles or high-frequency geometry because it tests the actual triangle geometry against the voxel volume, rather than approximating the voxel with a ray.
- **Limitations:** More computationally expensive than simple ray-box intersection due to the 13-axis SAT test. Requires pre-computed triangle bounding boxes for the fast cull step.
- **When to use:** When rendering high-frequency geometry (e.g., skinned meshes, detailed surfaces) where ray-box culling leads to missed intersections or artifacts.
- **When not to use:** For simple, low-frequency geometry where ray-box culling is sufficient and faster.

## Background
The method is part of the Adaptive Sparse Voxel Octree (ASVO) renderer by Dennis K. [S1, unofficial]. It was developed to handle the rendering of skinned animated models using voxelization, where traditional ray tracing or rasterization might struggle with the dynamic nature of the geometry. The implementation is found in the `asvo` repository [S1, unofficial].

## Sources
[S1] denniskb/asvo, https://github.com/denniskb/asvo, fetched 2026-10-26 (unofficial)
