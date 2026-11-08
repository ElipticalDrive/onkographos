# Sub-Voxel Volume Clipping


## Overview
Sub-Voxel Volume Clipping is a post-processing step applied to a pre-voxelized grid to handle the intersection of voxel boundaries with arbitrary clipping planes (such as a viewing frustum or a boolean operation boundary). It solves the problem of "stair-stepping" or hard edges that occur when a coarse voxel grid is cut by a plane, by distributing the fill value of a voxel across its internal sub-voxel cells based on the signed distance to the clipping face. It sets itself apart from sibling methods like [Front/Back Face Flagging](1506-front-back-face-flagging.md) by providing a smooth, linear falloff of density rather than a binary inside/outside classification, ensuring seamless reconstruction when clipped pieces are reassembled [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A voxel grid containing a byte array `field` representing fill values (typically 0-255). The voxel geometry is defined by a center position, a `cellSize`, and a subdivision structure `SubVoxels` (containing `Divisions` and `CellCount`).
- **Output:** A modified `field` array where values are quantized based on the fraction of the sub-voxel cell volume that lies inside the clipping boundary.

**Data Structures:**
- `SubVoxels`: A configuration object.
  - `Divisions`: Integer, the number of subdivisions along each axis (e.g., 4, 8).
  - `CellCount`: Integer, total sub-voxels per voxel ($Divisions^3$).
- `field`: Byte array of size `CellCount`.
- `firstCell`: Vector3, the center of the primary voxel in world space.
- `stepX`, `stepY`, `stepZ`: Vector3, the unit step vectors for the sub-voxel grid axes.
- `cellSize`: Float, the physical size of one sub-voxel.

**Parameters:**
- `HalfDiagonal`: Float, the half-diagonal of a unit cube, used as a threshold to determine if a voxel is entirely inside or outside a face. Value: $0.8660254f$ ($\sqrt{3}/2$) [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Create a new byte array `field` of size `SubVoxels.CellCount`. Calculate the voxel center `firstCell` and the reach distance `HalfDiagonal * cellSize * SubVoxels.Divisions + cellSize * 0.5f` [S1, unofficial].
2. **Face Precomputation:** For each intersecting clipping face (defined by a normal $\mathbf{n}$ and a point $\mathbf{p}$ on the plane):
   - Calculate `start`: The signed distance from `firstCell` to the plane.
   - Calculate `alongX`, `alongY`, `alongZ`: The dot products of the face normal $\mathbf{n}$ with the step vectors $\mathbf{stepX}$, $\mathbf{stepY}$, and $\mathbf{stepZ}$ respectively.
   - Store these values in a stack-allocated span or list for efficient access [S1, unofficial].
3. **Sub-Voxel Iteration:** Iterate over all sub-voxel cells indexed by $(sx, sy, sz)$ where $0 \le sx, sy, sz < Divisions$.
   - Retrieve the initial fill value from the source data. If the source is null or empty, default to 255 [S1, unofficial].
   - If the fill value is 0, skip this cell [S1, unofficial].
   - Calculate the position of the current sub-voxel center relative to `firstCell`:
     $$ \mathbf{pos} = firstCell + (sx \cdot \mathbf{stepX} + sy \cdot \mathbf{stepY} + sz \cdot \mathbf{stepZ}) $$
   - Calculate the signed distance $d$ from this sub-voxel center to the clipping plane:
     $$ d = \text{start} + sx \cdot \text{alongX} + sy \cdot \text{alongY} + sz \cdot \text{alongZ} $$
   - Note: The distance calculation assumes the normal points outward or inward consistently. The minimum signed distance to *any* intersecting face is used if multiple planes clip the voxel [S1, unofficial].
4. **Fill Fraction Calculation:** Compute the inside fill fraction $inside$ using the formula:
   $$ inside = \max(0, \min(1, 0.5 + \frac{distance}{cellSize})) $$
   Here, $distance$ is the signed distance $d$ calculated in step 3. The term $0.5$ centers the transition at the sub-voxel center, and $cellSize$ scales the linear falloff across the sub-voxel diameter [S1, unofficial].
5. **Quantization:**
   - If the sub-voxel is considered "inside" (typically determined by a threshold or the sign of the distance relative to the face normal direction), the new fill value is:
     $$ \text{new\_fill} = \text{Quantize}(fill \times inside) $$
   - Otherwise (if "outside"), the new fill value is:
     $$ \text{new\_fill} = \text{Quantize}(fill \times (1 - inside)) $$
   - `Quantize` maps the float result to the nearest integer in the byte range [0, 255] [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sub_voxel_volume_clipping(field, first_cell, step_x, step_y, step_z, cell_size, subvoxels_divisions, faces):
    """
    Clips a voxel's sub-voxel field along clipping faces.
    
    Args:
        field: List[int] or bytearray of size subvoxels_divisions^3, initial fill values.
        first_cell: Tuple[float, float, float], center of the voxel.
        step_x, step_y, step_z: Tuple[float, float, float], unit step vectors for sub-voxel grid.
        cell_size: float, physical size of one sub-voxel.
        subvoxels_divisions: int, number of subdivisions per axis.
        faces: List[Tuple[Tuple[float, float, float], float]], each face is (normal, point_on_plane).
        
    Returns:
        List[int] or bytearray, the modified field.
    """
    divisions = subvoxels_divisions
    cell_count = divisions ** 3
    new_field = [0] * cell_count
    
    # Precompute face parameters
    face_params = []
    for normal, point in faces:
        # Distance from first_cell to plane: dot(normal, first_cell - point)
        # But usually plane equation is dot(n, x) - d = 0. 
        # Based on S1: start is distance from first cell center to face.
        # Assuming normal is normalized.
        start = dot(normal, first_cell) - dot(normal, point)
        along_x = dot(normal, step_x)
        along_y = dot(normal, step_y)
        along_z = dot(normal, step_z)
        face_params.append((start, along_x, along_y, along_z))
        
    half_diagonal = 0.8660254f # sqrt(3)/2
    
    # Iterate over all sub-voxels
    for sx in range(divisions):
        for sy in range(divisions):
            for sz in range(divisions):
                idx = sx * divisions * divisions + sy * divisions + sz
                
                # Get initial fill value
                fill = field[idx]
                if fill == 0:
                    continue
                    
                # Calculate distance to the closest face
                # S1 says "minimum signed distance to any intersecting face"
                min_dist = float('inf')
                for start, ax, ay, az in face_params:
                    dist = start + sx * ax + sy * ay + sz * az
                    # Note: S1 implies using the signed distance directly. 
                    # We need to handle the sign convention. 
                    # If the normal points OUT, inside is negative distance? 
                    # Or if normal points IN, inside is positive?
                    # S1 formula: 0.5 + distance/cell_size. 
                    # If distance is 0 (center on plane), inside = 0.5.
                    # If distance is -cell_size/2 (center half cell inside), inside = 0.25?
                    # Let's assume 'distance' is signed such that positive is inside.
                    # We take the min distance if we want the tightest clip? 
                    # Actually, for a single plane, it's just that plane's distance.
                    # For multiple planes (frustum), we might need the min of the signed distances 
                    # if all normals point inward.
                    if dist < min_dist:
                        min_dist = dist
                        
                distance = min_dist
                
                # Calculate inside fraction
                # S1: inside = max(0, min(1, 0.5 + distance/cell_size))
                inside = max(0.0, min(1.0, 0.5 + distance / cell_size))
                
                # Determine if inside or outside based on the sign of distance?
                # S1 says: "If keeping inside... otherwise..."
                # This implies a binary decision. Usually, if distance > 0 (inside), use inside.
                # If distance < 0 (outside), use 1-inside.
                # However, the formula 0.5 + d/cell_size handles the transition.
                # If d = -cell_size/2, inside = 0.
                # If d = cell_size/2, inside = 1.
                
                if distance >= 0:
                    # Inside
                    new_fill = quantize(fill * inside)
                else:
                    # Outside
                    new_fill = quantize(fill * (1 - inside))
                    
                new_field[idx] = new_fill
                
    return new_field

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def quantize(val):
    return int(round(max(0, min(255, val))))
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in distance calculation can cause artifacts at the boundary. Use consistent normal directions for all clipping planes.
- **Boundary Conditions:** If a sub-voxel center is exactly on the plane, `inside` is 0.5. The quantization step determines whether it is counted as inside or outside.
- **Degenerate Input:** If `cell_size` is 0, division by zero occurs. Ensure `cell_size` is positive.
- **Multiple Faces:** If clipping against a frustum, the "minimum signed distance" logic assumes all face normals point inward. If normals point outward, use maximum signed distance.

## Operating characteristics
- **Time Complexity:** $O(N \cdot F)$, where $N$ is the number of sub-voxels per voxel ($Divisions^3$) and $F$ is the number of clipping faces. Since $N$ is small (e.g., $8^3=512$), this is effectively constant per voxel.
- **Memory Complexity:** $O(N)$ per voxel for the new field array.
- **Performance:** The method is lightweight and suitable for real-time applications. The use of stack-allocated spans for face parameters minimizes memory allocation overhead [S1, unofficial].
- **Strengths:** Provides smooth transitions at clip boundaries, reducing aliasing artifacts. Seamless reconstruction when pieces are reassembled [S1, unofficial].
- **Limitations:** Requires pre-computation of face parameters. The linear falloff may not perfectly represent complex geometric intersections but is a good approximation for volume rendering.

## Background
Sub-Voxel Volume Clipping is a technique used in voxel-based rendering engines to handle clipping of volumetric data. It is implemented in the VoxelTerraformer project, which focuses on efficient voxel generation and manipulation [S1, unofficial]. The method addresses the limitations of binary voxelization by introducing sub-voxel granularity for clipping operations.

## Sources
[S1] GhostlyActive/VoxelTerraformer, https://github.com/GhostlyActive/VoxelTerraformer, fetched 2026-10-27 (unofficial)
