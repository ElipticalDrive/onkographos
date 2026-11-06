# Ray-Cast Style Voxel Filling

Also called: Front-to-Back Scanline Fill, Z-Direction Interior Fill

## Overview
Ray-Cast Style Voxel Filling is a method for determining the interior voxels of a mesh by scanning along a principal axis (typically Z) and filling intervals between front-facing and back-facing surface voxels [S1]. It differs from [Stochastic Ray-Casting Voxelization](1067-stochastic-ray-casting-voxelization.md) by using deterministic scanline logic rather than probabilistic sampling, and from [Mesh Winding Number Voxelization](2328-mesh-winding-number-voxelization.md) by relying on explicit face orientation flags rather than topological winding counts [S1].

## How it works
**Inputs:**
- A 3D voxel grid `volume` of dimensions `width` $\times$ `height` $\times$ `depth`.
- Each voxel `volume[x, y, z]` contains:
  - `IsEmpty()`: boolean, true if the voxel is outside the mesh bounds or not hit by the initial ray-cast pass.
  - `IsFrontFace()`: boolean, true if the voxel represents a surface facing the positive Z direction (entry point).
  - `IsBackFace()`: boolean, true if the voxel represents a surface facing the negative Z direction (exit point).
  - `fill`: integer, 0 for empty, 1 for interior.
  - `position`: vector, center of the voxel.

**Outputs:**
- The same `volume` grid, with `fill` set to 1 for all voxels strictly inside the mesh along the scan lines.

**Algorithm:**
The algorithm iterates over every $(x, y)$ column and scans along the Z-axis from $0$ to `depth`-1. It identifies intervals where the ray enters the mesh (front face) and exits the mesh (back face), filling the voxels between them.

1. Iterate $x$ from $0$ to `width`-1.
2. Iterate $y$ from $0$ to `height`-1.
3. Initialize $z = 0$.
4. While $z < \text{depth}$:
   a. If `volume[x, y, z].IsEmpty()` is true, increment $z$ and continue to step 4.
   b. Set `ifront` = $z$.
   c. Advance `ifront` while `ifront` < `depth` AND `volume[x, y, ifront].IsFrontFace()` is true.
   d. If `ifront` >= `depth`, break the loop (end of column).
   e. Set `iback` = `ifront`.
   f. Advance `iback` while `iback` < `depth` AND `volume[x, y, iback].IsEmpty()` is true.
   g. If `iback` >= `depth`, break the loop.
   h. If `volume[x, y, iback].IsBackFace()` is true:
      i. Advance `iback` while `iback` < `depth` AND `volume[x, y, iback].IsBackFace()` is true.
   i. Fill voxels from $z_2 = \text{ifront}$ to $z_2 < \text{iback}$:
      1. Set `volume[x, y, z2].fill` = 1.
      2. Set `volume[x, y, z2].position` = center of voxel $(x, y, z_2)$.
      3. Update `volume[x, y, z2]`.
   j. Set $z = \text{iback}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_cast_style_fill(volume, width, height, depth):
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
                    volume[x, y, z2].fill = 1
                    volume[x, y, z2].position = get_voxel_center(x, y, z2)
                    volume[x, y, z2] = volume[x, y, z2] # Commit update
                
                # Continue scanning from the exit point
                z = iback
```

**Edge Cases and Pitfalls:**
- **Non-watertight meshes:** If the mesh is not watertight, the algorithm may fail to find a corresponding back face, leaving interior voxels unfilled or incorrectly filled. This method assumes the input voxelization correctly identifies front/back faces for a closed surface [S1].
- **Precision:** The logic relies on discrete voxel flags. If the initial ray-cast pass misclassifies a surface voxel as empty or misidentifies front/back orientation, the fill logic will produce artifacts.
- **Boundaries:** The loop checks `ifront < depth` and `iback < depth` to prevent out-of-bounds access.

## Subtopics

### [Slice Filling (Simple)](1691-slice-filling-simple.md)
A variant that processes 2D slices along one axis, scanning from both ends toward the center. It differs by using bidirectional scanning within a slice rather than unidirectional Z-axis traversal, and serves as the base for BFS propagation variants.

### [Solid Voxelization](1695-solid-voxelization.md)
Builds on this method by using a two-level grid and bitsets to ensure watertightness via flood-fill or projection, handling non-watertight inputs better than the basic scanline approach.

## Operating characteristics
- **Time Complexity:** $O(W \cdot H \cdot D)$ in the worst case, as each voxel is visited a constant number of times during the scan. The inner loops advance indices, ensuring no voxel is processed more than once per column.
- **Memory Complexity:** $O(W \cdot H \cdot D)$ to store the voxel grid. No additional significant memory is required.
- **Strengths:** Simple to implement, deterministic, and fast for regular grids. It avoids the overhead of global connectivity checks or complex topological calculations.
- **Limitations:** Fails on non-watertight meshes or meshes with complex topology where front/back face identification is ambiguous. It is sensitive to the accuracy of the initial surface voxelization pass.
- **When to use:** When the input mesh is known to be watertight and the voxel grid resolution is moderate. It is suitable for real-time applications where speed is critical and mesh integrity is guaranteed.
- **When not to use:** For non-watertight meshes or when high topological accuracy is required without post-processing.

## Background
The method originates from real-time GPU-based voxelization techniques, specifically described in the context of Unity graphics programming [S1]. It is a fundamental technique for converting triangle meshes into volumetric data by leveraging the directional properties of surface normals relative to the scan axis.

## Sources
[S1] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
