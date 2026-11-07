# Ray-Cast Style Voxel Filling

Also called: Front-to-Back Scanline Fill, Z-Direction Interior Fill

## Overview
Ray-Cast Style Voxel Filling is a method for determining the interior voxels of a mesh by scanning along a principal axis (typically Z) and filling intervals between front-facing and back-facing surface voxels [S1]. It differs from [Stochastic Ray-Casting Voxelization](1067-stochastic-ray-casting-voxelization.md) by using deterministic scanline logic rather than probabilistic sampling [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D voxel grid `volume` of dimensions `width` $\times$ `height` $\times$ `depth` [S1]. A secondary grid `boxes` storing geometric metadata for each voxel coordinate [S1].
- **Output:** The modified `volume` grid, with the `fill` field set to 1 for all voxels strictly inside the mesh along the scan lines [S1].

**Data Structures:**
Each voxel `volume[x, y, z]` contains:
- `is_empty()`: boolean, true if the voxel is outside the mesh bounds or not hit by the initial surface pass [S1].
- `is_front_face()`: boolean, true if the voxel represents a surface facing the positive Z direction (entry point) [S1].
- `is_back_face()`: boolean, true if the voxel represents a surface facing the negative Z direction (exit point) [S1].
- `fill`: integer, 0 for empty, 1 for interior [S1].
- `position`: vector, center of the voxel [S1].

**Formulas and Index Bounds:**
The algorithm operates within the bounds $0 \le x < \text{width}$, $0 \le y < \text{height}$, and $0 \le z < \text{depth}$ [S1]. The fill interval for a column $(x, y)$ is defined as $z_2 \in [\text{ifront}, \text{iback})$, where $\text{ifront}$ is the first front-facing index and $\text{iback}$ is the index immediately following the last consecutive back-facing index [S1].

**Parameters:**
Typical values for grid dimensions and resolution are Not given in the sources.

**Numbered Steps:**
1. Iterate $x$ from $0$ to `width`-1 [S1].
2. Iterate $y$ from $0$ to `height`-1 [S1].
3. Initialize $z = 0$ [S1].
4. While $z < \text{depth}$:
   a. If `volume[x, y, z].is_empty()` is true, increment $z$ and continue to step 4 [S1].
   b. Set `ifront` = $z$ [S1].
   c. Advance `ifront` while `ifront` < `depth` AND `volume[x, y, ifront].is_front_face()` is true [S1].
   d. If `ifront` >= `depth`, break the loop (end of column) [S1].
   e. Set `iback` = `ifront` [S1].
   f. Advance `iback` while `iback` < `depth` AND `volume[x, y, iback].is_empty()` is true [S1].
   g. If `iback` >= `depth`, break the loop [S1].
   h. If `volume[x, y, iback].is_back_face()` is true, advance `iback` while `iback` < `depth` AND `volume[x, y, iback].is_back_face()` is true [S1].
   i. Fill voxels from $z_2 = \text{ifront}$ to $z_2 < \text{iback}$: set `volume[x, y, z2].fill` = 1, update `position`, and commit the voxel [S1].
   j. Set $z = \text{iback}$ to continue scanning from the exit point [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_cast_style_fill(volume, width, height, depth, boxes):
    for x in range(width):
        for y in range(height):
            z = 0
            while z < depth:
                # Skip empty voxels
                if volume[x, y, z].is_empty():
                    z += 1
                    continue
                
                # Find first front-facing voxel
                ifront = z
                while ifront < depth and volume[x, y, ifront].is_front_face():
                    ifront += 1
                
                # If we reached the end, stop
                if ifront >= depth:
                    break
                
                # Find the exit point (back face or end of empty space)
                iback = ifront
                while iback < depth and volume[x, y, iback].is_empty():
                    iback += 1
                
                # If we reached the end, stop
                if iback >= depth:
                    break
                
                # If the exit voxel is a back face, skip all consecutive back faces
                if volume[x, y, iback].is_back_face():
                    while iback < depth and volume[x, y, iback].is_back_face():
                        iback += 1
                
                # Fill the interior interval [ifront, iback)
                for z2 in range(ifront, iback):
                    volume[x, y, z2].position = boxes[x, y, z2].center
                    volume[x, y, z2].fill = 1
                    volume[x, y, z2] = volume[x, y, z2]  # Commit update
                
                # Continue scanning from the exit point
                z = iback
```

**Edge Cases and Pitfalls:**
- **Non-watertight meshes:** If the mesh is not watertight, the algorithm may fail to find a corresponding back face, leaving interior voxels unfilled or incorrectly filled [S1]. This method assumes the input voxelization correctly identifies front/back faces for a closed surface [S1].
- **Precision:** The logic relies on discrete voxel flags. If the initial ray-cast pass misclassifies a surface voxel as empty or misidentifies front/back orientation, the fill logic will produce artifacts [G].
- **Boundaries:** The loop checks `ifront < depth` and `iback < depth` to prevent out-of-bounds access [G].

## Subtopics

### [Slice Filling (Simple)](1691-slice-filling-simple.md)
Slice Filling (Simple) is a voxelization method that fills the interior of a mesh by processing 2D slices of the voxel grid along one of the three principal axes. It solves the problem of determining which voxels inside the mesh bounds are interior by scanning each slice from both ends toward the center, marking voxels as filled until an existing solid voxel is encountered. It differs from [Solid Voxelization](1695-solid-voxelization.md) by relying on linear edge-to-center scanning within slices rather than global connectivity or flood-fill algorithms, and it serves as the base case for [Slice Filling with BFS Propagation](1909-slice-filling-with-bfs-propagation.md), which extends this linear scan with neighbor propagation.

### [Solid Voxelization](1695-solid-voxelization.md)
Solid Voxelization fills the interior of a mesh with voxels (solid/bulk) rather than just the surface, using a two-level grid model. It builds on [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md) by determining interior voxels via a two-pass approach with bitsets and a final inversion step. It differs from [Slice Filling (Simple)](1691-slice-filling-simple.md) by explicitly handling volume connectivity via flood-fill or axis-aligned projection strategies to ensure watertightness, even if the input mesh is not strictly watertight.

## Operating characteristics
- **Time Complexity:** $O(W \cdot H \cdot D)$ in the worst case, as each voxel is visited a constant number of times during the scan [G]. The inner loops advance indices, ensuring no voxel is processed more than once per column [G].
- **Memory Complexity:** $O(W \cdot H \cdot D)$ to store the voxel grid [G]. No additional significant memory is required [G].
- **Strengths:** Simple to implement, deterministic, and fast for regular grids [G]. It avoids the overhead of global connectivity checks or complex topological calculations [G].
- **Limitations:** Fails on non-watertight meshes or meshes with complex topology where front/back face identification is ambiguous [G]. It is sensitive to the accuracy of the initial surface voxelization pass [G].
- **When to use:** When the input mesh is known to be watertight and the voxel grid resolution is moderate [G]. It is suitable for real-time applications where speed is critical and mesh integrity is guaranteed [G].
- **When not to use:** For non-watertight meshes or when high topological accuracy is required without post-processing [G].

## Background
The method is described in the context of CPU-based voxelization for Unity graphics programming [S1]. It is a fundamental technique for converting triangle meshes into volumetric data by leveraging the directional properties of surface normals relative to the scan axis [S1].

## Sources
[S1] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
