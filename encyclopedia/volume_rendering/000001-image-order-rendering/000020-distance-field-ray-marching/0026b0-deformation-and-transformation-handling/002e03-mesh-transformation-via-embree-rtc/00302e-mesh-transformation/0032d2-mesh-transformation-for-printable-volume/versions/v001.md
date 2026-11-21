# Mesh Transformation for Printable Volume


## Overview
Computes a 4x4 transformation matrix to scale, rotate, and translate a mesh so it fits within a specified printable volume, typically aligning the largest dimension with the z-axis [S1, unofficial]. It differs from generic mesh transformations by incorporating volume constraints and automatic axis alignment [S1, unofficial].

## How it works
**Inputs:**
- `mesh`: A mesh object with vertex positions and bounding box.
- `print_size`: A 3D vector `[x, y, z]` defining the maximum printable dimensions.
- `size` (optional): A 3D vector specifying a target size. If provided, it must be $\le$ `print_size` [S1, unofficial].

**Outputs:**
- `to_world`: A 4x4 transformation matrix (scale $\circ$ rotation $\circ$ translation) that maps local mesh coordinates to world/printable coordinates [S1, unofficial].

**Data Structures:**
- `bbox`: Axis-aligned bounding box with fields `min` (3D vector) and `max` (3D vector).
- `extents`: 3D vector where `extents[i] = bbox.max[i] - bbox.min[i]` [S1, unofficial].

**Steps:**

1. **Compute Extents:** Calculate `ext_max = max(bbox.extents())` [S1, unofficial].
2. **Determine Rotation Axis:** Identify the axis of maximum extent:
   $$ \text{axis\_max} = \text{select}(\text{bbox.extents()} == \text{ext\_max}, 1.0, 0.0) $$
   If `axis_max` is not $(0, 0, 1)$, compute the rotation axis:
   $$ \text{rot\_axis} = \text{cross}(\text{axis\_max}, (0, 0, 1)) $$
   Compute the rotation matrix $R$ as a rotation of $90^\circ$ around `rot_axis` [S1, unofficial].
3. **Center in XY Plane:**
   - Mask non-z dimensions of vertex positions to find the 2D footprint in the XY plane.
   - Compute the minimum enclosing circle (miniball) center $C_{xy}$ and radius $r$ for these 2D points [S1, unofficial].
   - Compute translation $T_{xy}$ to center the object at $(0, 0)$ in the XY plane [S1, unofficial].
4. **Compute Scale Factors:**
   - $s_z = \text{print\_size.z} / \text{ext\_max}$
   - $s_{xy} = 0.5 \times \text{print\_size.xy} / r$
   - Let $s = \min(s_z, s_{xy})$ [S1, unofficial].
5. **Assemble Transform:**
   - If `size` is not provided:
     $$ \text{to\_world} = S(s) \circ R \circ T_{xy} $$
     where $S(s)$ is a uniform scale matrix [S1, unofficial].
   - If `size` is provided:
     - Check if any component of `size` > `print_size`. If so, raise an error [S1, unofficial].
     - Compute center $c = 0.5 \times (\text{bbox.min} + \text{bbox.max})$ [S1, unofficial].
     - Compute scale factor $s' = \text{size} / \text{ext\_max}$ (element-wise or uniform? Source implies uniform scaling relative to ext_max, but uses size vector. Note: Source says `scale(size/ext_max)`. Assuming uniform scale based on `ext_max` or per-axis? Source says `scale.z = ... scale.xy = ...` in previous step, but here `scale(size/ext_max)` suggests a uniform scale if `size` is a scalar or vector. Given `ext_max` is a scalar, `size/ext_max` is a vector. This implies non-uniform scaling if `size` is a vector. However, standard practice is uniform. Let's stick to the formula: $S = \text{diag}(\text{size} / \text{ext\_max})$? Or is `size` a scalar? The source says `size` is a 3D vector. Let's assume uniform scaling to fit the bounding box into the target size box, but the source formula `scale(size/ext_max)` is ambiguous. Let's look at `scale.z` and `scale.xy` in step 4. Step 4 computes a uniform $s$. Step 5 uses `size`. If `size` is provided, it overrides the `print_size` constraint. The formula `scale(size/ext_max)` likely means a uniform scale factor $s_{target} = \text{size} / \text{ext\_max}$? No, `size` is 3D. It likely means scaling the object to fit within `size`. Let's assume the source implies a uniform scale if `size` is a scalar, or per-axis if `size` is a vector. Given `ext_max` is a scalar, `size/ext_max` is a vector. This results in a non-uniform scale matrix. Let's write it as such [S1, unofficial].
     $$ \text{to\_world} = S(\text{size} / \text{ext\_max}) \circ T(-c) $$
     Note: The rotation and XY centering might be skipped or applied differently if `size` is provided. The source says `to_world = scale(size/ext_max) @ translate(-c)`. It does not mention rotation or XY centering in this branch. This implies `size` mode assumes the mesh is already oriented or ignores orientation [S1, unofficial].

**Parameters:**
- `print_size`: Typical values depend on the printer (e.g., $200 \times 200 \times 200$ mm) [S1, unofficial].
- `size`: Optional target dimensions [S1, unofficial].

**Edge Cases:**
- If `ext_max` is 0, division by zero occurs. Guard against empty meshes [G].
- If `axis_max` is $(0,0,1)$, no rotation is needed [S1, unofficial].
- If `size` > `print_size`, an error is raised [S1, unofficial].

## Subtopics

### Surface Alignment Factors
Calculates scale and translation factors to align a surface mesh in micron coordinates with a volume defined by voxel size and volume size [S2, unofficial]. It initializes default `surfaceScale` to $0.00161$ and `surfaceTranslation` to $[-313.7, -147.5, -86.5]$ [S2, unofficial]. If `units` is 'micron', it calculates `micronSize` as `volSize` mapped to $e \times \text{voxelSize}[i]$ [S2, unofficial]. If `sizeMax` (max of `micronSize`) > 0, `surfaceScale` is set to $1 / \text{sizeMax}$; otherwise, it is 1 [S2, unofficial]. `surfaceTranslation` is set to `micronSize` mapped to $-e / 2$ for each element [S2, unofficial]. It returns an object `{ surfaceScale: number, surfaceTranslation: [number, number, number] }` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of vertices, due to bounding box and miniball computation [S1, unofficial].
- **Memory Complexity:** $O(N)$ to store vertex positions for miniball calculation [S1, unofficial].
- **Strengths:** Automatically orients the object for optimal printing; handles volume constraints [S1, unofficial].
- **Limitations:** The `size` branch ignores rotation and XY centering, which may lead to suboptimal orientation if the mesh is not already aligned [S1, unofficial].

## Background
Originated in the context of inverse rendering for tomographic volumetric additive manufacturing [S1, unofficial]. It is used to prepare meshes for 3D printing by ensuring they fit within the printer's build volume and are oriented correctly [S1, unofficial]. The surface alignment variant is used in web-based volume viewers for scientific visualization [S2, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
[S2] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
