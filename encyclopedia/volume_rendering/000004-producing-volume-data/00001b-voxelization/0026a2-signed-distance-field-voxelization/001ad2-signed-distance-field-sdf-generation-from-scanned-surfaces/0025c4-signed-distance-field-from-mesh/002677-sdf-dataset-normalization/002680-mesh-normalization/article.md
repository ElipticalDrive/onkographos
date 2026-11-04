# Mesh Normalization


## Overview
Mesh normalization transforms a 3D mesh's vertex coordinates so that the geometry is centered at the origin and scaled to fit within a unit cube $[-1, 1]^3$ [S1, unofficial]. It differs from Variance Normalization, which scales based on the distribution of points rather than bounding extent [S2, unofficial].

## How it works
**Inputs:** A mesh object containing a list of vertices $V = \{v_1, v_2, \dots, v_N\}$ where each $v_i \in \mathbb{R}^3$ [G].
**Outputs:** The same mesh object with updated vertex coordinates [G].

**Data Structures:**
- `vertices`: Array of 3D vectors representing mesh vertices [S1, unofficial].
- `bounds`: Pair of 3D vectors representing the minimum and maximum coordinates of the mesh [S1, unofficial].
- `extents`: 3D vector of axis-aligned bounding box sizes [S1, unofficial].

**Steps:**
1. **Compute bounds and center:** Determine the axis-aligned bounding box and its center [S1, unofficial].
   $$ center = bounds.mean(axis=0) $$
2. **Compute max extent:** Find the largest dimension of the bounding box [S1, unofficial].
   $$ max\_extent = extents.max() $$
3. **Compute scale factor:** Determine the uniform scale factor required to fit the mesh into a unit cube [S1, unofficial].
   $$ scale\_factor = \frac{1.0}{max\_extent} $$
4. **Translate:** Subtract the center from each vertex to move the mesh to the origin [S1, unofficial].
   $$ v'_i = v_i - center $$
5. **Scale:** Multiply each translated vertex by the scale factor [S1, unofficial].
   $$ v''_i = v'_i \times scale\_factor $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize_mesh(mesh):
    bounds = mesh.bounds
    center = bounds.mean(axis=0)
    extents = mesh.extents
    max_extent = extents.max()
    if max_extent == 0.0:
        return mesh  # Guard against division by zero [G]
    scale_factor = 1.0 / max_extent
    mesh.vertices -= center
    mesh.vertices *= scale_factor
    return mesh
```

**Parameters:**
- None. The method is deterministic based on the input geometry [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Extent:** If the mesh consists of a single point or all vertices are identical, `max_extent` is 0, leading to division by zero. A guard clause is required [G].

## Subtopics

### Variance Normalization
Variance normalization scales a mesh so that the mean squared distance from its center of mass is 1 [S2, unofficial]. Unlike Mesh Normalization which uses the bounding box extent, this method uses the covariance of the vertex positions [S2, unofficial].
**Steps:**
1. Compute the center of mass (COM) of the vertices [S2, unofficial].
2. Translate the mesh to the origin: $v'_i = v_i - COM$ [S2, unofficial].
3. Compute the covariance matrix $C$ of the translated vertices [S2, unofficial].
4. Compute the scale factor $s$ using the trace of the covariance matrix: $s = \frac{1}{\sqrt{\text{tr}(C)}}$ [S2, unofficial].
5. Apply the scale: $v''_i = v'_i \times s$ [S2, unofficial].
6. Translate back to the original center of mass: $v'''_i = v''_i + COM$ [S2, unofficial].
**Formula:**
$$ s = \frac{1}{\sqrt{C_{00} + C_{11} + C_{22}}} $$
where $C_{ij}$ are the diagonal elements of the covariance matrix [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of vertices [G].
- **Memory Complexity:** $O(1)$ auxiliary space if performed in-place [G].
- **Strengths:** Simple, fast, and ensures the model fits within a known canonical space $[-1, 1]^3$ [S1, unofficial].
- **Limitations:** Does not account for the internal distribution of the mesh, only its outer bounds. Thin structures or outliers can dominate the scale factor, potentially shrinking important features [G].
- **When to use:** Standard preprocessing for SDF generation and voxelization where a canonical bounding box is required [G].

## Background
- **Where it is used:** Standard preprocessing step for Signed Distance Field (SDF) generation and voxelization, ensuring consistent scale and position for downstream tasks [G]. The specific implementation described here, centering at the bounding box center and scaling by the max extent, is documented in [S1, unofficial].

## Sources
[S1] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S2] prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, https://github.com/prateekchandan/Advanced-CG-High-Resolution-Sparse-Voxel-DAGs, fetched 2026-10-26 (unofficial)
