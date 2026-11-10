# Dominant-Axis Triangle Voxelization

Also called: 3D Bounding Box Voxelization, Case 7-9 Voxelization, Dominant Axis Voxelization, Multi-Pass Voxelization, Voxelization with Dominant Axis Projection

## Overview
Dominant-Axis Triangle Voxelization is a rasterization-based algorithm that determines which voxels in a sparse voxel DAG are intersected by a triangle by projecting the triangle onto a 2D plane defined by its dominant normal axis [S1, S2, unofficial]. It sets itself apart from general 3D clipping methods by reducing the problem to 2D edge tests along the axis of maximum surface alignment, optimizing column casting for sparse storage [S1, unofficial]. This method is a specific implementation detail within the NGLE Triangle Voxelization framework, handling the 3D bounding box thickness cases (Cases 7-9) where the triangle spans multiple voxels in all three dimensions [S1, S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Triangle vertices in world space: $v_0, v_1, v_2$ [S1, unofficial].
- Triangle bounding box in grid coordinates: $bbox_{grid}.min, bbox_{grid}.max$ [S1, unofficial].
- Voxel side length: $unitlength$ [S1, unofficial].
- Parent bounding box limits: $p_{bbox\_grid}.min, p_{bbox\_grid}.max$ [S1, unofficial].
- Mode flag: `single_pass` or `multi_pass` [S2, unofficial].
- In multi-pass mode: Pre-computed dominant axis buffer, current projection direction $projDir.direction$, and primitive vertex index `vid` [S2, unofficial].

**Outputs:**
- A set of occupied voxel Morton codes in a sparse array [S1, unofficial].
- Voxel data (e.g., color, normal) if non-binary mode is enabled [S1, unofficial].

### Data Structures
- **Voxel Storage**: A sparse array indexed by Morton codes. The base offset is `morton_start`. For non-binary mode, `VoxelData` is pushed to a vector, and the index (`voxel_data.size() - 1`) is stored in the voxels array. Binary mode stores `true` [S1, unofficial].
- **Dominant Axis Buffer**: In multi-pass mode, an array storing the pre-computed dominant axis for each triangle, accessed via `vid / 3` [S2, unofficial].

### Steps

1. **Calculate Plane Constant:**
   Compute the plane constant $d$ using the normal $n$ and vertex $v_0$:
   $$ d = -1.0 \cdot (n_x \cdot v_{0x} + n_y \cdot v_{0y} + n_z \cdot v_{0z}) $$
   [S1, unofficial]

2. **Determine Dominant Axis:**
   Identify the axis with the largest absolute normal component.
   - Initialize `dominant_axis = X`.
   - If $|n_y| > |n_{dominant\_axis}|$, set `dominant_axis = Y`.
   - If $|n_z| > |n_{dominant\_axis}|$, set `dominant_axis = Z`.
   [S1, unofficial]
   
   *Single-Pass Mode:* Compute the triangle's normal by transforming vertices to world space ($pos_0, pos_1, pos_2$). Calculate the absolute cross product of edge vectors: $triNormal = |\text{cross}(pos_1 - pos_0, pos_2 - pos_0)|$ [S2, unofficial].
   
   *Multi-Pass Mode:* Retrieve the pre-computed dominant axis from the buffer using `vid / 3`. If the retrieved axis does not match `projDir.direction`, output a degenerate position `float4(-2,-2,-2,-2)` to cull the primitive [S2, unofficial].

3. **Iterate Based on Dominant Axis:**
   The algorithm projects the triangle onto the plane perpendicular to the dominant axis and iterates over the 2D grid cells within the triangle's bounding box on that plane [S1, unofficial].

   **Case A: Dominant Axis is X**
   - Iterate over $y$ and $z$ within $bbox_{grid}.min/max$ [S1, unofficial].
   - **YZ Plane Projection Test:** For each edge $e_i$ ($i=0,1,2$), compute the 2D edge normal in the YZ plane:
     $$ n_{yz\_ei} = \text{vec2}(-e_{iz}, e_{iy}) $$
     If $n_x < 0.0$, negate $n_{yz\_ei}$ [S1, unofficial].
     Compute the offset:
     $$ d_{yz\_ei} = -1.0 \cdot (n_{yz\_ei} \cdot \text{vec2}(v_{0y}, v_{0z})) + \max(0.0, unitlength \cdot n_{yz\_ei,x}) + \max(0.0, unitlength \cdot n_{yz\_ei,y}) $$
     A voxel at $(y,z)$ is inside the triangle's projection if:
     $$ (n_{yz\_ei} \cdot \text{vec2}(y \cdot unitlength, z \cdot unitlength)) + d_{yz\_ei} \ge 0.0 $$
     for all three edges [S1, unofficial].
   - **X-Range Calculation:** For valid $(y,z)$ columns, determine the range of voxels in the X direction. Define min/max corners in the YZ plane:
     $$ min\_corner = (y \cdot unitlength, z \cdot unitlength) $$
     $$ max\_corner = min\_corner + \text{vec2}(unitlength, unitlength) $$
     Swap components if corresponding normal components are negative: if $n_y < 0$, swap $min\_corner[0]$ and $max\_corner[0]$; if $n_z < 0$, swap $min\_corner[1]$ and $max\_corner[1]$ [S1, unofficial].
     Project YZ corners to X coordinates on the triangle plane:
     $$ x_{min\_world} = \frac{n_y \cdot min\_corner[0] + n_z \cdot min\_corner[1] + d}{-1.0 \cdot n_x} $$
     $$ x_{max\_world} = \frac{n_y \cdot max\_corner[0] + n_z \cdot max\_corner[1] + d}{-1.0 \cdot n_x} $$
     Convert to voxel indices:
     $$ x_{min} = x_{min\_world} / unitlength $$
     $$ x_{max} = x_{max\_world} / unitlength $$
     [S1, unofficial]
   - **Clamping:** Clamp $x_{min}$ and $x_{max}$ to $p_{bbox\_grid}.min/max[X]$ [S1, unofficial].
   - **Overlap Tests:** If the depth range ($x_{max} - x_{min}$) > 1, perform additional overlap tests in the ZX and XY planes for each $x \in [x_{min}, x_{max}]$. Compute edge normals and offsets for these planes, adjusting signs based on normal components (e.g., if $n_y < 0.0$, negate ZX edge normals) [S1, unofficial].
   - **Voxel Marking:** For each $x$, compute the Morton index for $(z, y, x)$. Check if already marked. Test the voxel center against the XY and ZX edge inequalities. If inside both, mark the voxel and increment `nfilled` [S1, unofficial].

   **Case B: Dominant Axis is Y**
   - Iterate over $z$ and $x$ within $bbox_{grid}.min/max$ [S1, unofficial].
   - **XZ Plane Projection Test:** Similar to YZ test, project onto XZ plane [S1, unofficial].
   - **Y-Range Calculation:** Determine Y-range using corner projection:
     $$ y_{min\_world} = \frac{n_z \cdot min\_corner[0] + n_x \cdot min\_corner[1] + d}{-1.0 \cdot n_y} $$
     Clamp $y_{min}$ and $y_{max}$ to $p_{bbox\_grid}.min/max[Y]$. Handle $y_{min} == y_{max}$ special case [S1, unofficial].
   - **Overlap Tests:** For ranges > 1, test YZ and XY planes [S1, unofficial].

   **Case C: Dominant Axis is Z**
   - Iterate over $x$ and $y$ within $bbox_{grid}.min/max$ [S1, unofficial].
   - **XY Plane Projection Test:** Similar to previous tests, project onto XY plane [S1, unofficial].
   - **Z-Range Calculation:** Determine Z-range using corner projection:
     $$ z_{min\_world} = \frac{n_x \cdot min\_corner[0] + n_y \cdot min\_corner[1] + d}{-1.0 \cdot n_z} $$
     Clamp $z_{min}$ and $z_{max}$ to $p_{bbox\_grid}.min/max[Z]$. Handle $z_{min} == z_{max}$ special case [S1, unofficial].
   - **Overlap Tests:** For ranges > 1, test XZ and YZ planes [S1, unofficial].

4. **Write Voxel Data:**
   Write to voxel texture using one of three methods based on function constants [S2, unofficial]:
   - `kUseRWTexture`: Read existing value, perform max blend ($res = \max(res, stored)$), write back [S2, unofficial].
   - `kUseWTexture`: Write directly without blending [S2, unofficial].
   - `Atomic buffer`: Convert result to RGBA8 uint. Use `atomic_compare_exchange_weak_explicit` loop to perform max blend: load `prevValue`, if mismatch, $res = \max(res, rgba8ToVec4(prevValue))$, convert back to `newValue`, retry [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dominant_axis_voxelization(triangle, bbox_grid, unitlength, parent_bbox, mode, dominant_axis_buffer=None, proj_dir=None, vid=None):
    """
    Voxelize a triangle using the dominant axis method.
    triangle: {v0: vec3, v1: vec3, v2: vec3}
    bbox_grid: {min: vec3i, max: vec3i}
    unitlength: float
    parent_bbox: {min: vec3i, max: vec3i}
    mode: 'single_pass' or 'multi_pass'
    dominant_axis_buffer: list of ints (only for multi_pass)
    proj_dir: int (only for multi_pass)
    vid: int (only for multi_pass)
    """
    v0 = triangle['v0']
    v1 = triangle['v1']
    v2 = triangle['v2']
    
    # 1. Calculate Normal and Plane Constant
    edge1 = v1 - v0
    edge2 = v2 - v0
    normal = abs(cross(edge1, edge2))
    d = -dot(normal, v0)
    
    # Guard against degenerate triangle
    if dot(normal, normal) == 0:
        return []

    # 2. Determine Dominant Axis
    if mode == 'single_pass':
        dominant_axis = 'X'
        if abs(normal['Y']) > abs(normal[dominant_axis]):
            dominant_axis = 'Y'
        if abs(normal['Z']) > abs(normal[dominant_axis]):
            dominant_axis = 'Z'
    else: # multi_pass
        precomputed_axis = dominant_axis_buffer[vid // 3]
        if precomputed_axis != proj_dir:
            return [] # Cull primitive
        dominant_axis = precomputed_axis

    occupied_voxels = []

    # 3. Iterate based on Dominant Axis
    if dominant_axis == 'X':
        for y in range(bbox_grid['min']['Y'], bbox_grid['max']['Y'] + 1):
            for z in range(bbox_grid['min']['Z'], bbox_grid['max']['Z'] + 1):
                if not is_inside_yz_projection(triangle, normal, d, y, z, unitlength):
                    continue
                
                min_corner = vec2(y * unitlength, z * unitlength)
                max_corner = min_corner + vec2(unitlength, unitlength)
                
                if normal['Y'] < 0:
                    min_corner[0], max_corner[0] = max_corner[0], min_corner[0]
                if normal['Z'] < 0:
                    min_corner[1], max_corner[1] = max_corner[1], min_corner[1]
                
                x_min_world = (normal['Y'] * min_corner[0] + normal['Z'] * min_corner[1] + d) / (-1.0 * normal['X'])
                x_max_world = (normal['Y'] * max_corner[0] + normal['Z'] * max_corner[1] + d) / (-1.0 * normal['X'])
                
                x_min = x_min_world / unitlength
                x_max = x_max_world / unitlength
                
                x_min = clamp(x_min, parent_bbox['min']['X'], parent_bbox['max']['X'])
                x_max = clamp(x_max, parent_bbox['min']['X'], parent_bbox['max']['X'])
                
                for x in range(x_min, x_max + 1):
                    if x_max > x_min:
                        if not is_inside_zx_projection(triangle, normal, d, x, z, unitlength):
                            continue
                        if not is_inside_xy_projection(triangle, normal, d, x, y, unitlength):
                            continue
                    
                    morton = compute_morton(z, y, x)
                    occupied_voxels.append(morton)
                    
    elif dominant_axis == 'Y':
        for z in range(bbox_grid['min']['Z'], bbox_grid['max']['Z'] + 1):
            for x in range(bbox_grid['min']['X'], bbox_grid['max']['X'] + 1):
                if not is_inside_xz_projection(triangle, normal, d, x, z, unitlength):
                    continue
                
                min_corner = vec2(x * unitlength, z * unitlength)
                max_corner = min_corner + vec2(unitlength, unitlength)
                
                if normal['X'] < 0:
                    min_corner[0], max_corner[0] = max_corner[0], min_corner[0]
                if normal['Z'] < 0:
                    min_corner[1], max_corner[1] = max_corner[1], min_corner[1]
                
                y_min_world = (normal['Z'] * min_corner[0] + normal['X'] * min_corner[1] + d) / (-1.0 * normal['Y'])
                y_max_world = (normal['Z'] * max_corner[0] + normal['X'] * max_corner[1] + d) / (-1.0 * normal['Y'])
                
                y_min = y_min_world / unitlength
                y_max = y_max_world / unitlength
                
                y_min = clamp(y_min, parent_bbox['min']['Y'], parent_bbox['max']['Y'])
                y_max = clamp(y_max, parent_bbox['min']['Y'], parent_bbox['max']['Y'])
                
                for y in range(y_min, y_max + 1):
                    if y_max > y_min:
                        if not is_inside_yz_projection(triangle, normal, d, y, z, unitlength):
                            continue
                        if not is_inside_xy_projection(triangle, normal, d, x, y, unitlength):
                            continue
                    
                    morton = compute_morton(z, y, x)
                    occupied_voxels.append(morton)

    elif dominant_axis == 'Z':
        for x in range(bbox_grid['min']['X'], bbox_grid['max']['X'] + 1):
            for y in range(bbox_grid['min']['Y'], bbox_grid['max']['Y'] + 1):
                if not is_inside_xy_projection(triangle, normal, d, x, y, unitlength):
                    continue
                
                min_corner = vec2(x * unitlength, y * unitlength)
                max_corner = min_corner + vec2(unitlength, unitlength)
                
                if normal['X'] < 0:
                    min_corner[0], max_corner[0] = max_corner[0], min_corner[0]
                if normal['Y'] < 0:
                    min_corner[1], max_corner[1] = max_corner[1], min_corner[1]
                
                z_min_world = (normal['X'] * min_corner[0] + normal['Y'] * min_corner[1] + d) / (-1.0 * normal['Z'])
                z_max_world = (normal['X'] * max_corner[0] + normal['Y'] * max_corner[1] + d) / (-1.0 * normal['Z'])
                
                z_min = z_min_world / unitlength
                z_max = z_max_world / unitlength
                
                z_min = clamp(z_min, parent_bbox['min']['Z'], parent_bbox['max']['Z'])
                z_max = clamp(z_max, parent_bbox['min']['Z'], parent_bbox['max']['Z'])
                
                for z in range(z_min, z_max + 1):
                    if z_max > z_min:
                        if not is_inside_xz_projection(triangle, normal, d, x, z, unitlength):
                            continue
                        if not is_inside_yz_projection(triangle, normal, d, y, z, unitlength):
                            continue
                    
                    morton = compute_morton(z, y, x)
                    occupied_voxels.append(morton)

    return occupied_voxels

def compute_morton(z, y, x):
    # Standard 3D Morton code interleaving
    m = 0
    for i in range(10): # Assumes coordinates fit in 10 bits each
        m |= ((x & (1 << i)) << (2 * i))
        m |= ((y & (1 << i)) << (2 * i + 1))
        m |= ((z & (1 << i)) << (2 * i + 2))
    return m

def is_inside_yz_projection(triangle, normal, d, y, z, unitlength):
    v0 = triangle['v0']
    v1 = triangle['v1']
    v2 = triangle['v2']
    edges = [v1 - v0, v2 - v1, v0 - v2]
    
    for e in edges:
        n_yz_ei = vec2(-e['Z'], e['Y'])
        if normal['X'] < 0.0:
            n_yz_ei = -n_yz_ei
        d_yz_ei = -1.0 * dot(n_yz_ei, vec2(v0['Y'], v0['Z'])) + \
                  max(0.0, unitlength * n_yz_ei[0]) + \
                  max(0.0, unitlength * n_yz_ei[1])
        if dot(n_yz_ei, vec2(y * unitlength, z * unitlength)) + d_yz_ei < 0.0:
            return False
    return True

def is_inside_xz_projection(triangle, normal, d, x, z, unitlength):
    v0 = triangle['v0']
    v1 = triangle['v1']
    v2 = triangle['v2']
    edges = [v1 - v0, v2 - v1, v0 - v2]
    
    for e in edges:
        n_xz_ei = vec2(-e['Z'], e['X'])
        if normal['Y'] < 0.0:
            n_xz_ei = -n_xz_ei
        d_xz_ei = -1.0 * dot(n_xz_ei, vec2(v0['X'], v0['Z'])) + \
                  max(0.0, unitlength * n_xz_ei[0]) + \
                  max(0.0, unitlength * n_xz_ei[1])
        if dot(n_xz_ei, vec2(x * unitlength, z * unitlength)) + d_xz_ei < 0.0:
            return False
    return True

def is_inside_xy_projection(triangle, normal, d, x, y, unitlength):
    v0 = triangle['v0']
    v1 = triangle['v1']
    v2 = triangle['v2']
    edges = [v1 - v0, v2 - v1, v0 - v2]
    
    for e in edges:
        n_xy_ei = vec2(-e['Y'], e['X'])
        if normal['Z'] < 0.0:
            n_xy_ei = -n_xy_ei
        d_xy_ei = -1.0 * dot(n_xy_ei, vec2(v0['X'], v0['Y'])) + \
                  max(0.0, unitlength * n_xy_ei[0]) + \
                  max(0.0, unitlength * n_xy_ei[1])
        if dot(n_xy_ei, vec2(x * unitlength, y * unitlength)) + d_xy_ei < 0.0:
            return False
    return True
```

### Parameters
- `unitlength`: The side length of a voxel, used to convert grid coordinates to world coordinates and vice versa [S1, unofficial].
- `DIST_FACTOR`: A constant multiplier for distance in attenuation calculation. Value: 1.1f [S2, unofficial].
- `CONSTANT`, `LINEAR`, `QUADRATIC`: Coefficients for the attenuation formula. Values: CONSTANT=1, LINEAR=0, QUADRATIC=1 [S2, unofficial].

### Edge Cases and Pitfalls
- **Division by Zero**: If the dominant axis component of the normal is zero, the projection formula divides by zero. The code assumes the dominant axis has the largest magnitude, but does not explicitly check for zero [S1, unofficial].
- **Clamping Typo**: In the Y-dominant case, the source code contains a typo where $y_{max}$ is clamped using $y_{min}$ as the lower bound argument: `clampval<int>(y_min, p_bbox_grid.min[Y], p_bbox_grid.max[Y])` instead of `clampval<int>(y_max, ...)` [S1, unofficial].
- **Degenerate Triangles**: If the triangle normal is zero, the dominant axis cannot be determined. [G] A check for zero-length normal should be added before computing the dominant axis.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Strengths**: Not given in the sources.
- **Limitations**: Not given in the sources.
- **When to Use**: Not given in the sources.
- **When Not to Use**: Not given in the sources.

## Background
This method is part of the NGLE Triangle Voxelization algorithm, which classifies triangle bounding box thickness into 9 cases (1D, 2D, or 3D) and uses plane/projection tests to determine which grid cells are intersected [S1, S2, unofficial]. It is used in high-resolution sparse voxel DAG construction [S1, unofficial] and real-time voxel cone tracing applications [S2, unofficial].

## Sources
[S1] leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, https://github.com/leonzheng2/INF584-High_Resolution_Sparse_Voxel_DAGs, fetched 2026-10-26 (unofficial)
[S2] kakashidinho/VoxelConeTracingMetal, https://github.com/kakashidinho/VoxelConeTracingMetal, fetched 2026-10-26 (unofficial)
