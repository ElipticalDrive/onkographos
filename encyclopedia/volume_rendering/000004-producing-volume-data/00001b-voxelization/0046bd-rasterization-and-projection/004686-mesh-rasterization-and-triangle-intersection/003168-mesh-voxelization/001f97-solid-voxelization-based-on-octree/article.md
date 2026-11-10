# Solid Voxelization based on Octree

Also called: Polygon-to-Octree Rasterization, RFVoxelOctreeBuilder, voxelization via bounding box overlap

## Overview
Solid Voxelization based on Octree converts a polygon mesh (specifically OBJ format) into a hierarchical octree representation to determine the solid/empty status and distance to surface for each leaf node [S1, S2, unofficial]. It differs from fixed-grid voxelization by using an octree structure for adaptive resolution and from ray-casting-only methods by combining spatial partitioning with ray casting for sign determination [S2, unofficial]. It builds on [Mesh Voxelization](6514-mesh-voxelization.md) by providing the specific octree-based implementation for mesh inputs.

## How it works
**Inputs:** A triangular mesh in OBJ format (vertices, faces) and a target dimension $D$ (default 64) [S2, unofficial].
**Outputs:** An octree structure where leaf nodes store distance to surface and solid/empty classification [S2, unofficial].

**Data Structures:**
*   **Octree Node (`RFVoxelOctreeNode`):**
    *   `m_kWorldAabb`: World-space axis-aligned bounding box (`RFAABB3d`).
    *   `m_kBuildAabb`: Build-space axis-aligned bounding box with integer min/max (`RFAABB3d` with integer coordinates).
    *   `m_pkChildren`: Array of 8 pointers to child nodes.
    *   `m_eNodeType`: Enum indicating `NT_EMPTY_LEAF`, `NT_LEAF`, or `NT_INTERNAL`.
    *   `m_fDistanceToSurface`: Double precision distance to the closest triangle surface.
*   **Octree (`RFVoxelOctree`):**
    *   `m_uiOctreeDim`: Dimension of the voxelization grid (power of two).
    *   `m_fVoxelSize`: Size of a single voxel.
    *   `m_kOctreeOrigin`: Origin of the octree in world space.
    *   `m_pkRoot`: Pointer to the root node.
    *   `m_uiNumberOfLeafs`, `m_uiNumberOfEmptyLeafs`, `m_uiNumberOfInternalNodes`: Counters.
*   **Voxel (`vx_voxel`):** (Used in alternative implementation [S1, unofficial])
    *   `normals[3]`: `int8_t`.
    *   `colors[4]`: `uint8_t`.
    *   Padding and padding2 fields.
*   **Node Header (`vx_node`):** (Used in alternative implementation [S1, unofficial])
    *   `child_pointers[8]`: `uint32_t`.
    *   `valid_masks`: `uint8_t`.
    *   `leaf_mask`: `uint8_t`.

**Steps:**

1.  **Parse Mesh:** Read the OBJ file. Count vertices, UVs, normals, and faces. Track the bounding box min/max coordinates. Convert 1-based face indices to 0-based by decrementing by 1 [S1, S2, unofficial].
2.  **Calculate Dimensions:**
    *   Determine the next power of two for the octree dimension $D$. If input is 0, return 2; otherwise, decrement, then bitwise OR with right shifts of 16, 8, 4, 2, and 1 bits, then increment [S2, unofficial].
    *   Find the longest axis of the mesh AABB: $kDim = kMax - kMin$.
    *   Set $m\_uiOctreeDim$ to the next power of two of $kOctreeDim$ [S2, unofficial].
    *   Calculate voxel size: $m\_fVoxelSize = fMaxDim / m\_uiOctreeDim$.
    *   Calculate world size: $fOctreeWorldSize = m\_uiOctreeDim \times m\_fVoxelSize$.
    *   Set origin: $m\_kOctreeOrigin = kMeshAabb.GetCenter() - RFVector3d(fOctreeWorldSize / 2.0f)$ [S2, unofficial].
    *   Alternatively, normalize model into unit cube $[0,1]^3$: $offset = -min$, $scale = 1.0 / max(width, height, depth)$ [S1, unofficial].
3.  **Initialize Root:**
    *   Create root node.
    *   Set `m_kWorldAabb` from `m_kOctreeOrigin` to `m_kOctreeOrigin + RFVector3d(fOctreeWorldSize)`.
    *   Set `m_kBuildAabb.kMin` to $(0,0,0)$ and `kMax` to $(m\_uiOctreeDim, m\_uiOctreeDim, m\_uiOctreeDim)$.
    *   Set `m_kDataDimension` to `RFVector3i(m_uiOctreeDim)` [S2, unofficial].
    *   In the alternative implementation, initialize with allocated size 4096, size 1, and masks set to 0 [S1, unofficial].
4.  **Subdivide Recursively:** For a parent node, create 8 children. For each child $(x,y,z)$ where $x,y,z \in \{0,1\}$:
    *   Set `m_kBuildAabb.kMin` = `parent.kMin` + $(iAabbDim \cdot x, iAabbDim \cdot y, iAabbDim \cdot z)$.
    *   Set `kMax` = `kMin` + $iAabbDim$.
    *   Calculate world AABB for the child using `m_kOctreeOrigin` and `m_fVoxelSize` [S2, unofficial].
5.  **Assign Triangles:** For each triangle in the parent's list:
    *   Check if `TriangleAabbIntersects(kTriangleVerts, kWorldAabb)`.
    *   If true, add the triangle index to the child's list [S2, unofficial].
    *   In the alternative implementation, expand the triangle bounding box by one step size ($1.0 / (1 \ll detail)$) to handle edge cases, then check for overlap using `triBoxOverlap` [S1, unofficial].
6.  **Classify Node:**
    *   If child triangle list is empty, set type to `NT_EMPTY_LEAF` [S2, unofficial].
    *   If `iAabbDim == 1`, set type to `NT_LEAF` and call `UpdateLeafNodeDistanceValue` [S2, unofficial].
    *   Otherwise, set type to `NT_INTERNAL` and recursively call `Subdivide` [S2, unofficial].
7.  **Update Leaf Distance (Solid/Empty Determination):**
    *   For `NT_LEAF` nodes:
        *   Initialize $fMinDist$ to max double.
        *   For each triangle, compute normal $kNormal = VectorCross(kTriangleVerts[2] - kTriangleVerts[0], kTriangleVerts[1] - kTriangleVerts[0])$.
        *   Find $kClosestPoint = ClosestPointOnTriangle(kTriangleVerts, leaf\_center)$.
        *   Compute $kDelta = kClosestPoint - leaf\_center$.
        *   Compute $fSign = (VectorDot(kDelta, kNormal) < 0.0f) ? -1.0f : 1.0f$.
        *   Compute $fDist = VectorLength(kDelta)$.
        *   If $fDist < abs(fMinDist)$, set $fMinDist = fDist \times fSign$.
        *   Store in `m_fDistanceToSurface` [S2, unofficial].
    *   For `NT_INTERNAL` nodes (treated as empty leaves for classification purposes in some contexts, or recursively processed):
        *   Cast rays from node center using precomputed sample directions (200 directions generated via Hammersley2D and UniformSphericalSampling).
        *   Count $uiCoverage$ (hits) and $uiMisses$.
        *   Stop if $uiMisses \ge uiMaxMisses$ (size/4).
        *   If $fCoverage = uiCoverage / size \ge fMinCoverage$ (default 0.8-0.9), set `m_fDistanceToSurface` = -1.0f (solid), else 1.0f (empty) [S2, unofficial].
8.  **Generate Sample Directions:**
    *   Loop $i$ from 0 to 199.
    *   Compute $kRand = Hammersley2D(i, 200)$.
    *   Compute direction = `UniformSphericalSampling(kRand.tX, kRand.tY)` [S2, unofficial].

**Formulas:**
*   **Hammersley2D:** $x = i/N$, $y = RadicalInverseVdC(i)$.
    *   $RadicalInverseVdC$ performs bit reversal on $uiBits$: swap 16-bit halves, swap 8-bit halves, swap 4-bit nibbles, swap 2-bit pairs, swap 1-bit pairs, then multiply by $2.3283064365386963e-10$ [S2, unofficial].
*   **UniformSphericalSampling:** $\phi = v \cdot 2\pi$, $\cos\theta = 1 - u$, $\sin\theta = \sqrt{1 - \cos^2\theta}$. Result: $(\cos\phi \sin\theta, \sin\phi \sin\theta, \cos\theta)$ [S2, unofficial].

**Parameters:**
*   `kOctreeDim`: Requested dimension (default 64) [S2, unofficial].
*   `detail`: Logarithmic resolution (alternative implementation, default 8) [S1, unofficial].
*   `fMinCoverage`: Threshold for ray coverage (default 0.8-0.9) [S2, unofficial].
*   `fBorderSize`: Default 0.0, not used [S2, unofficial].

**Pitfalls:**
*   OBJ face indices are 1-based; must be decremented by 1 [S1, S2, unofficial].
*   Bounding box expansion by 'step' ensures triangles touching voxel edges are not missed [S1, unofficial].
*   Empty leaf classification stopping heuristic (misses > 25%) may lead to inaccurate coverage for nodes near the surface [S2, unofficial].
*   Method only supports OBJ files with vertices ordered in triangles [S2, unofficial].

## Subtopics

### [Octree-based 3D Voxelization with SAT Pruning](2588-octree-based-3d-voxelization-with-sat-pruning.md)
Uses Separating Axis Theorem (SAT) for pruning empty bounding volumes instead of ray casting. Choose for robustness and performance with arbitrary meshes.

### [Robust Inside and Outside Solid Voxelization](3376-robust-inside-and-outside-solid-voxelization.md)
Uses cube map rendering and visibility propagation for inside/outside status, handling non-watertight meshes. Choose for meshes with holes or intersecting geometry.

### [Solid Voxelization via Ray Casting](6829-solid-voxelization-via-ray-casting.md)
Computes solid intervals per pixel via ray-mesh intersection. Choose when explicit interval computation is required.

### [Hierarchical Inside/Outside Propagation](9772-hierarchical-inside-outside-propagation.md)
Post-processing step correcting inside/outside status by propagating boundary crossings. Choose to fix inconsistent solid status in octree structures.

## Operating characteristics
*   **Time Complexity:** Depends on octree depth and triangle distribution. Recursive subdivision and ray casting for sign determination add overhead compared to fixed grids [S2, unofficial].
*   **Memory Complexity:** Adaptive; fewer nodes for sparse scenes, but requires storing pointers and metadata for each node [S2, unofficial].
*   **Performance:** On Apple M1 Pro, building an octree for Armadillo.obj (64 dim) took ~484ms before optimization, ~77ms after optimization for output [S2, unofficial].
*   **Strengths:** Adaptive resolution, handles sparse scenes efficiently, robust sign determination via ray casting [S2, unofficial].
*   **Limitations:** Only supports OBJ format, recursive overhead, potential inaccuracies in coverage calculation near surfaces [S1, S2, unofficial].
*   **When to use:** When adaptive resolution is needed and input is OBJ format.
*   **When not to use:** For non-OBJ formats, or when fixed-grid performance is critical and adaptive resolution is not needed.

## Background
Originated from Peter Trier Mikkelsen's work, presented in a tutorial and video [S2, unofficial]. Implemented in open-source projects like `voxl` and `Voxelizer` [S1, S2, unofficial]. Used in volume rendering pipelines to convert mesh models into voxel data for further processing.

## Sources
[S1] rools/voxl, https://github.com/rools/voxl, fetched 2026-10-26 (unofficial)
[S2] itsyouyi/Voxelizer, https://github.com/itsyouyi/Voxelizer, fetched 2026-10-27 (unofficial)
