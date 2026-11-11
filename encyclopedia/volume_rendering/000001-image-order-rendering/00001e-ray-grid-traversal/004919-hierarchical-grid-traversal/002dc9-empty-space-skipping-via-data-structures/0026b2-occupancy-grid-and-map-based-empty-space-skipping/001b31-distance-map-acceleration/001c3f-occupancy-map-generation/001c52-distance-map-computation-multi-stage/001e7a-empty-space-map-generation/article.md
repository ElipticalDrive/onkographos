# Empty Space Map Generation

Also called: Empty Space Skipping Map, Ray Marching with Empty Space Map, Visibility Map Generation

## Overview
Empty Space Map Generation is a preprocessing technique that creates a boolean map indicating which voxels in a scalar field contain no visible geometry, allowing ray marching to skip empty regions [S1, unofficial]. It reduces the number of density samples and transfer function evaluations by jumping over contiguous blocks of empty space [S1, unofficial]. Unlike [Distance Map Computation (Multi-Stage)](1205-distance-map-computation-multi-stage.md), which computes distances to occupied blocks, this method uses a binary occupancy check based on a transfer function threshold [S1, unofficial].

## How it works
**Inputs:**
- Scalar field data: A 3D array of density values with dimensions $W \times H \times D$.
- Transfer function: A function $TF(v)$ that maps a scalar value $v$ to a density or opacity value.
- Threshold: A tolerance value $T$. If any vertex of a voxel has a transfer function density $> T$, the voxel is considered non-empty [S1, unofficial].

**Outputs:**
- Empty Space Map: A boolean array of size $(W-1) \times (H-1) \times (D-1)$. Each element corresponds to a voxel in the scalar field. If `true`, the voxel is empty; if `false`, it contains visible geometry [S1, unofficial].

**Data Structures:**
- `empty_space_map`: A 1D boolean array or 3D boolean array with dimensions $(W-1) \times (H-1) \times (D-1)$. The index for voxel $(i, j, k)$ is calculated as $i + j(W-1) + k(W-1)(H-1)$ [S1, unofficial].

**Steps:**
1. **Preprocessing (Map Generation):**
   1. Initialize `empty_space_map` to all `false`.
   2. Loop over all voxel indices $(i, j, k)$ where $0 \le i < W-1$, $0 \le j < H-1$, $0 \le k < D-1$.
   3. For each voxel, identify its 8 vertices. The vertex indices relative to the voxel are $(v_i, v_j, v_k)$ where $v_i, v_j, v_k \in \{0, 1\}$.
   4. For each vertex $v$, compute the global scalar field coordinates:
      $$ x = i + v_i, \quad y = j + v_j, \quad z = k + v_k $$
   5. Retrieve the scalar value $s$ from the field at $(x, y, z)$.
   6. Evaluate the transfer function density $d = TF(s)$.
   7. If any vertex has $d > T$, mark the voxel as non-empty. Otherwise, mark it as empty.
      $$ \text{empty\_space\_map}[i, j, k] = \text{is\_empty} $$
      where `is_empty` is `true` if all 8 vertices have $d \le T$, and `false` otherwise [S1, unofficial].

2. **Ray Marching (Skipping):**
   1. For a given ray with origin $O$ and direction $D$, compute the current voxel indices $(v_i, v_j, v_k)$ from the ray position $p$.
   2. Check if the current voxel is marked as empty in `empty_space_map`.
   3. If the voxel is empty:
      a. Compute the voxel minimum $V_{min}$ and maximum $V_{max}$ points in world space.
      b. Compute the intersection interval $[t_{min}, t_{max}]$ of the ray with the voxel bounding box.
      c. Advance the ray parameter $t$ by the maximum of $(t_{max} - t)$ and a minimum marching step $\Delta t$ to ensure progress and avoid precision issues.
         $$ t \leftarrow t + \max(t_{max} - t, \Delta t) $$
      d. Update the ray position $p = O + t \cdot D$.
      e. Repeat the check for the new voxel [S1, unofficial].
   4. If the voxel is not empty, proceed with standard ray marching sampling within the voxel [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_empty_space_map(field, transfer_function, threshold, width, height, depth):
    """
    Generates a boolean map indicating empty voxels.
    
    Args:
        field: 3D array of scalar values, shape (width, height, depth).
        transfer_function: Function that takes a scalar value and returns density.
        threshold: Density threshold. Voxels with all vertices below this are empty.
        width, height, depth: Dimensions of the scalar field.
        
    Returns:
        empty_space_map: 3D boolean array of shape (width-1, height-1, depth-1).
    """
    w, h, d = width - 1, height - 1, depth - 1
    empty_space_map = [[ [False for _ in range(d)] for _ in range(h) ] for _ in range(w) ]
    
    for i in range(w):
        for j in range(h):
            for k in range(d):
                is_empty = True
                # Check all 8 vertices of the voxel (i, j, k)
                for vi in range(2):
                    for vj in range(2):
                        for vk in range(2):
                            # Global coordinates of the vertex
                            x = i + vi
                            y = j + vj
                            z = k + vk
                            
                            # Get scalar value
                            scalar_value = field[x][y][z]
                            
                            # Evaluate transfer function
                            density = transfer_function(scalar_value)
                            
                            # If any vertex exceeds threshold, voxel is not empty
                            if density > threshold:
                                is_empty = False
                                break
                        if not is_empty:
                            break
                    if not is_empty:
                        break
                empty_space_map[i][j][k] = is_empty
                
    return empty_space_map

def ray_march_with_empty_space(ray_origin, ray_direction, empty_space_map, voxel_size, marching_delta):
    """
    Advances a ray through empty space using the precomputed map.
    
    Args:
        ray_origin: 3D vector (x, y, z).
        ray_direction: 3D vector (x, y, z), assumed normalized.
        empty_space_map: 3D boolean array of shape (W-1, H-1, D-1).
        voxel_size: Size of a single voxel (assumed uniform).
        marching_delta: Minimum step size to ensure progress.
        
    Returns:
        New ray position after skipping empty space.
    """
    t = 0.0
    # Assume ray_origin is in world space
    # Convert ray origin to voxel coordinates
    current_voxel = [
        int(ray_origin[0] / voxel_size),
        int(ray_origin[1] / voxel_size),
        int(ray_origin[2] / voxel_size)
    ]
    
    # Ensure indices are within bounds
    w, h, d = len(empty_space_map), len(empty_space_map[0]), len(empty_space_map[0][0])
    current_voxel[0] = max(0, min(current_voxel[0], w - 1))
    current_voxel[1] = max(0, min(current_voxel[1], h - 1))
    current_voxel[2] = max(0, min(current_voxel[2], d - 1))
    
    while True:
        i, j, k = current_voxel
        
        # Check if voxel is empty
        if empty_space_map[i][j][k]:
            # Compute voxel bounds in world space
            voxel_min = [
                i * voxel_size,
                j * voxel_size,
                k * voxel_size
            ]
            voxel_max = [
                (i + 1) * voxel_size,
                (j + 1) * voxel_size,
                (k + 1) * voxel_size
            ]
            
            # Compute intersection of ray with voxel bounding box
            # Ray: P = O + t * D
            # For each axis, find t_min and t_max
            t_min = -math.inf
            t_max = math.inf
            
            for axis in range(3):
                if ray_direction[axis] == 0:
                    if ray_origin[axis] < voxel_min[axis] or ray_origin[axis] > voxel_max[axis]:
                        # Ray is parallel and outside voxel
                        return ray_origin # Or handle as no intersection
                else:
                    t1 = (voxel_min[axis] - ray_origin[axis]) / ray_direction[axis]
                    t2 = (voxel_max[axis] - ray_origin[axis]) / ray_direction[axis]
                    t_min = max(t_min, min(t1, t2))
                    t_max = min(t_max, max(t1, t2))
            
            if t_min > t_max:
                # No intersection
                return ray_origin
            
            # Advance t to the exit of the voxel
            # Use FMax to ensure progress
            t_advance = max(t_max - t, marching_delta)
            t += t_advance
            
            # Update ray position
            ray_origin = [
                ray_origin[0] + ray_direction[0] * t_advance,
                ray_origin[1] + ray_direction[1] * t_advance,
                ray_origin[2] + ray_direction[2] * t_advance
            ]
            
            # Update voxel indices
            current_voxel = [
                int(ray_origin[0] / voxel_size),
                int(ray_origin[1] / voxel_size),
                int(ray_origin[2] / voxel_size)
            ]
            current_voxel[0] = max(0, min(current_voxel[0], w - 1))
            current_voxel[1] = max(0, min(current_voxel[1], h - 1))
            current_voxel[2] = max(0, min(current_voxel[2], d - 1))
        else:
            # Voxel is not empty, stop skipping
            break
            
    return ray_origin
```

**Parameters:**
- `threshold`: The density threshold for the transfer function. Typical values depend on the specific transfer function and data normalization. Not given in the sources.
- `marching_delta`: The minimum step size to ensure progress. Not given in the sources.

**Edge Cases and Pitfalls:**
- Precision problems may occur when computing voxel intersections. The travel value is clamped using `FMax` to ensure progress [S1, unofficial].
- The map size is $(W-1) \times (H-1) \times (D-1)$, so the scalar field must have at least 2 voxels in each dimension [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** Preprocessing is $O(W \cdot H \cdot D)$, as each voxel requires checking 8 vertices. Ray skipping is $O(1)$ per empty voxel skipped.
- **Memory Complexity:** $O(W \cdot H \cdot D)$ bits for the boolean map.
- **Performance:** The method reduces the number of density samples and transfer function evaluations [S1, unofficial].
- **Strengths:** Simple to implement, effective for volumes with large empty regions.
- **Limitations:** Binary check may skip over thin structures if the threshold is not carefully chosen. Precision issues can occur at voxel boundaries [S1, unofficial].

## Background
This method is part of the SRaimondi/VolumeRender project, an unofficial implementation of volume rendering techniques [S1, unofficial]. It is used as a preprocessing step to accelerate ray marching in volume rendering.

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
