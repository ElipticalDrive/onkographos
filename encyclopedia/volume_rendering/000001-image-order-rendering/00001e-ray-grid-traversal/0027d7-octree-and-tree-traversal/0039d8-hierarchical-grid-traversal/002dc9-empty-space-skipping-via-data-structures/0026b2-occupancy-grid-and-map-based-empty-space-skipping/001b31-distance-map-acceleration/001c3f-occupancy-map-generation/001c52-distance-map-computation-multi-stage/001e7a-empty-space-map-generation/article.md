# Empty Space Map Generation

Also called: Empty Space Skipping Map, Ray Marching with Empty Space Map, Visibility Map Generation

## Overview
Empty Space Map Generation is a preprocessing optimization for volume ray marching that creates a boolean map indicating which voxels contain no visible geometry based on a transfer function threshold [S1, unofficial]. It allows the renderer to skip entire empty regions during ray traversal, significantly reducing the number of density samples and transfer function evaluations [S1, unofficial]. The method integrates the transfer function's density criteria directly into the skipping logic, ensuring that only voxels contributing to the final image are processed [S1, unofficial].

## How it works
**Inputs:**
- Scalar field data for a volume of dimensions $W \times H \times D$ [S1, unofficial].
- Transfer function definition (control points and evaluation method) [S1, unofficial].
- Threshold density value ($T$) [S1, unofficial].

**Outputs:**
- `empty_space_map`: A boolean array of size $(W-1) \times (H-1) \times (D-1)$ [S1, unofficial].

**Data Structures:**
- `empty_space_map`: 3D boolean array where index $(i, j, k)$ corresponds to the voxel with minimum corner at grid coordinates $(i, j, k)$ [S1, unofficial].
- `tf_vertices`: Temporary array of 8 floats, storing the transfer function output for the 8 vertices of the current voxel [S1, unofficial].

**Steps:**
1. **Initialize Map:** Allocate `empty_space_map` with dimensions $(W-1) \times (H-1) \times (D-1)$ [S1, unofficial].
2. **Iterate Voxels:** Loop over all voxel indices $i$ from $0$ to $W-2$, $j$ from $0$ to $H-2$, and $k$ from $0$ to $D-2$ [S1, unofficial].
3. **Sample Vertices:** For the current voxel $(i, j, k)$, identify its 8 vertices [S1, unofficial]. The vertex indices are encoded as $v = v_i + v_j \cdot 2 + v_k \cdot 4$, where $v_i, v_j, v_k \in \{0, 1\}$ represent the corner offsets [S1, unofficial].
4. **Evaluate Transfer Function:** For each vertex $v$, retrieve the scalar field value at the corresponding grid position $(i + v_i, j + v_j, k + v_k)$ and evaluate the transfer function [S1, unofficial].
   $$ \text{tf\_vertices}[v] = \text{transfer\_function}(\text{field}(i + v_i, j + v_j, k + v_k)) $$
5. **Determine Emptiness:** Check if any vertex has a transfer function density value greater than the threshold $T$ [S1, unofficial].
   $$ \text{is\_empty} = \max(\text{tf\_vertices}) \le T $$
6. **Store Result:** Set the map value for the current voxel [S1, unofficial].
   $$ \text{empty\_space\_map}[\text{Offset}(i, j, k)] = \text{is\_empty} $$

**Ray Marching Integration:**
When rendering, the ray marcher uses this map to skip empty voxels [S1, unofficial]:
1. Compute current 3D voxel indices $(v_i[0], v_i[1], v_i[2])$ from the ray position $p$ [S1, unofficial].
2. Check `empty_space_map` at these indices [S1, unofficial].
3. If empty:
   - Compute the voxel minimum ($v_{min}$) and maximum ($v_{max}$) points in world space [S1, unofficial].
   - Compute the intersection interval $(t_{min}, t_{max})$ of the ray with the voxel bounding box [S1, unofficial].
   - Advance the ray parameter $t$ by the maximum of $(t_{max} - t)$ and a minimum marching delta to ensure progress and avoid precision stalls [S1, unofficial].
   $$ t \leftarrow t + \max(t_{max} - t, \delta_{march}) $$
4. If not empty, proceed with standard volume integration [S1, unofficial].

**Parameters:**
- `tolerance` ($T$): Threshold density value. If any vertex of a voxel has a transfer function density $> T$, the voxel is considered non-empty [S1, unofficial].

**Pitfalls:**
- **Precision Problems:** Computing voxel intersections can suffer from floating-point precision issues, potentially causing the ray to stall or skip incorrectly [S1, unofficial]. The travel value must be clamped using `FMax` to ensure strict progress along the ray [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_empty_space_map(field, transfer_function, width, height, depth, threshold, control_points):
    """
    Generates a boolean map indicating empty voxels based on a transfer function threshold.
    """
    # Initialize map with dimensions (W-1) x (H-1) x (D-1)
    empty_space_map = [[[False for _ in range(depth - 1)] 
                        for _ in range(height - 1)] 
                       for _ in range(width - 1)]
    
    # Iterate over all voxels
    for i in range(width - 1):
        for j in range(height - 1):
            for k in range(depth - 1):
                # Evaluate transfer function at 8 vertices
                tf_values = [0.0] * 8
                for vi in (0, 1):
                    for vj in (0, 1):
                        for vk in (0, 1):
                            v = vi + vj * 2 + vk * 4
                            # Get scalar value at vertex and evaluate TF
                            scalar_val = field[i + vi][j + vj][k + vk]
                            tf_values[v] = transfer_function(scalar_val, control_points)
                
                # Determine if voxel is empty
                # If max tf value <= threshold, the voxel is empty
                is_empty = max(tf_values) <= threshold
                
                # Store result
                empty_space_map[i][j][k] = is_empty
                
    return empty_space_map

def ray_march_step_with_empty_space(ray_origin, ray_direction, empty_space_map, voxel_size, marching_delta):
    """
    Single ray marching step using empty space map for skipping.
    Assumes uniform voxel size and axis-aligned grid.
    """
    t = 0.0
    
    # Compute current voxel indices from ray position
    x = ray_origin[0] + ray_direction[0] * t
    y = ray_origin[1] + ray_direction[1] * t
    z = ray_origin[2] + ray_direction[2] * t
    
    vi = int(x / voxel_size)
    vj = int(y / voxel_size)
    vk = int(z / voxel_size)
    
    # Check bounds
    if not (0 <= vi < len(empty_space_map) and 
            0 <= vj < len(empty_space_map[0]) and 
            0 <= vk < len(empty_space_map[0][0])):
        return t, False  # Out of bounds
        
    # Check if empty
    if empty_space_map[vi][vj][vk]:
        # Compute voxel bounds in world space
        v_min_x = vi * voxel_size
        v_max_x = (vi + 1) * voxel_size
        v_min_y = vj * voxel_size
        v_max_y = (vj + 1) * voxel_size
        v_min_z = vk * voxel_size
        v_max_z = (vk + 1) * voxel_size
        
        # Compute intersection interval of ray with voxel bounding box
        t_enter = t
        t_exit = float('inf')
        
        if ray_direction[0] != 0:
            t1 = (v_min_x - ray_origin[0]) / ray_direction[0]
            t2 = (v_max_x - ray_origin[0]) / ray_direction[0]
            t_enter = max(t_enter, min(t1, t2))
            t_exit = min(t_exit, max(t1, t2))
        
        if ray_direction[1] != 0:
            t1 = (v_min_y - ray_origin[1]) / ray_direction[1]
            t2 = (v_max_y - ray_origin[1]) / ray_direction[1]
            t_enter = max(t_enter, min(t1, t2))
            t_exit = min(t_exit, max(t1, t2))
            
        if ray_direction[2] != 0:
            t1 = (v_min_z - ray_origin[2]) / ray_direction[2]
            t2 = (v_max_z - ray_origin[2]) / ray_direction[2]
            t_enter = max(t_enter, min(t1, t2))
            t_exit = min(t_exit, max(t1, t2))
            
        # Advance t by the maximum of (t_exit - t) and marching_delta
        # This ensures progress and avoids precision stalls
        delta = max(t_exit - t, marching_delta)
        t = t + delta
        return t, True  # Skipped
    else:
        # Not empty, proceed with standard volume integration step
        t += marching_delta
        return t, False  # Processed
```

## Subtopics

### [Chebyshev Distance Map Empty Space Skipping](3833-chebyshev-distance-map-empty-space-skipping.md)
This method accelerates volume ray casting by pre-computing a Chebyshev distance map from an occupancy map, allowing rays to skip multiple empty blocks in any dimension. It improves upon standard occupancy map skipping by enabling larger, dimension-independent skips, though it requires more complex generation and higher memory overhead.

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Performance:** Significant speedup in sparse volumes where large empty regions exist, as it reduces the number of density samples and transfer function evaluations [S1, unofficial].
- **Strengths:** Simple to implement, low memory overhead compared to distance maps, effective for sparse data [S1, unofficial].
- **Limitations:** Only skips entire voxels; does not skip partial empty regions within a voxel [S1, unofficial].
- **When to use:** Sparse volumes with large empty regions, real-time rendering where preprocessing cost is acceptable [S1, unofficial].
- **When not to use:** Dense volumes with little empty space, or when partial voxel skipping is required [S1, unofficial].

## Background
Not given in the sources. The specific implementation details described here are derived from open-source CUDA volume rendering projects [S1, unofficial].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
