# Empty Space Map Generation

Also called: Empty Space Skipping Map, Ray Marching with Empty Space Map, Visibility Map Generation

## Overview
Empty Space Map Generation is a preprocessing technique that creates a boolean map indicating which voxels in a scalar field contain no visible geometry, allowing ray marching to skip empty regions [S1, unofficial]. It reduces the number of density samples and transfer function evaluations by jumping over contiguous blocks of empty space [S1, unofficial]. Unlike [Distance Map Computation (Multi-Stage)](1205-distance-map-computation-multi-stage.md), which computes distances to occupied blocks, this method uses a binary occupancy check based on a transfer function threshold [S1, unofficial].

## How it works
**Inputs:**
- Scalar field data: A 3D array of density values with dimensions $W \times H \times D$ [S1, unofficial].
- Transfer function: A function $TF(v)$ that maps a scalar value $v$ to a density or opacity value [S1, unofficial].
- Threshold: A tolerance value $T$. If any vertex of a voxel has a transfer function density $> T$, the voxel is considered non-empty [S1, unofficial].

**Outputs:**
- Empty Space Map: A boolean array of size $(W-1) \times (H-1) \times (D-1)$, where each element indicates if the corresponding voxel is empty [S1, unofficial].

**Data Structures:**
- `empty_space_map`: A flat boolean array indexed by a field offset function `field->Offset(i, j, k)` [S1, unofficial].

**Steps:**
1. **Preprocessing (Map Generation):**
   1. Loop over all voxel indices $(i, j, k)$ from $0$ to $\text{dims}[\text{axis}]-2$ for each dimension [S1, unofficial].
   2. For each voxel, evaluate the transfer function at its 8 vertices [S1, unofficial].
   3. Compute the vertex index as $v = v_i + v_j \times 2 + v_k \times 4$, where $v_i, v_j, v_k \in \{0, 1\}$ [S1, unofficial].
   4. Retrieve the scalar value at $(i + v_i, j + v_j, k + v_k)$ and evaluate its density $d = TF(\text{scalar})$ [S1, unofficial].
   5. Mark the voxel as empty if all 8 vertices have $d \le T$; otherwise mark it as non-empty [S1, unofficial].
   6. Store the result: `empty_space_map[field->Offset(i, j, k)] = is_empty` [S1, unofficial].

2. **Ray Marching (Skipping):**
   1. Compute the current 3D voxel indices $(v_i, v_j, v_k)$ from the current ray position $p$ [S1, unofficial].
   2. Check if the current voxel is marked as empty in `empty_space_map` using the computed indices and field offset [S1, unofficial].
   3. If the voxel is empty:
      a. Compute the voxel minimum $V_{\min}$ and maximum $V_{\max}$ points in world space [S1, unofficial].
      b. Compute the intersection interval $[t_{\min}, t_{\max}]$ of the ray with the voxel bounding box [S1, unofficial].
      c. Advance the ray parameter $t$ by the maximum of $(t_{\max} - t)$ and `marching_delta` to skip the empty voxel [S1, unofficial].
      d. Update the ray position $p = O + t \cdot D$ and recompute voxel indices [S1, unofficial].
      e. Repeat the check for the new voxel [S1, unofficial].
   4. If the voxel is not empty, proceed with standard ray marching sampling within the voxel [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_empty_space_map(field, transfer_function, threshold, width, height, depth):
    """Generates a boolean map indicating empty voxels based on a transfer function threshold."""
    w, h, d = width - 1, height - 1, depth - 1
    empty_space_map = [False] * (w * h * d)
    
    for i in range(w):
        for j in range(h):
            for k in range(d):
                is_empty = True
                # Check all 8 vertices of the voxel (i, j, k)
                for vi in range(2):
                    for vj in range(2):
                        for vk in range(2):
                            # Global coordinates of the vertex
                            x, y, z = i + vi, j + vj, k + vk
                            # Evaluate transfer function density
                            density = transfer_function(field[x][y][z])
                            # If any vertex exceeds threshold, voxel is not empty
                            if density > threshold:
                                is_empty = False
                                break
                        if not is_empty: break
                    if not is_empty: break
                # Store using a flat offset (equivalent to field->Offset(i, j, k))
                empty_space_map[i + j * w + k * w * h] = is_empty
    return empty_space_map

def ray_march_skip(ray_origin, ray_direction, empty_space_map, voxel_size, marching_delta):
    """Advances a ray through empty space using the precomputed map."""
    t = 0.0
    # Convert ray origin to voxel coordinates
    vi = int(ray_origin[0] / voxel_size)
    vj = int(ray_origin[1] / voxel_size)
    vk = int(ray_origin[2] / voxel_size)
    w, h, d = len(empty_space_map), 0, 0
    # Derive map dimensions from flat array size (assuming cubic or known layout)
    # For generality, we assume w, h, d are passed or derived; here we clamp to bounds
    # In practice, w, h, d are known from map generation.
    
    while True:
        # Clamp indices to map bounds
        vi = max(0, min(vi, w - 1))
        vj = max(0, min(vj, h - 1))
        vk = max(0, min(vk, d - 1))
        
        if empty_space_map[vi + vj * w + vk * w * h]:
            # Compute voxel bounds in world space
            vmin = [vi * voxel_size, vj * voxel_size, vk * voxel_size]
            vmax = [(vi + 1) * voxel_size, (vj + 1) * voxel_size, (vk + 1) * voxel_size]
            
            # Compute intersection interval [t_min, t_max] of ray with voxel bounding box
            t_min = -math.inf
            t_max = math.inf
            for axis in range(3):
                if ray_direction[axis] == 0:
                    if ray_origin[axis] < vmin[axis] or ray_origin[axis] > vmax[axis]:
                        return ray_origin + [ray_direction[0]*t, ray_direction[1]*t, ray_direction[2]*t]
                else:
                    t1 = (vmin[axis] - ray_origin[axis]) / ray_direction[axis]
                    t2 = (vmax[axis] - ray_origin[axis]) / ray_direction[axis]
                    t_min = max(t_min, min(t1, t2))
                    t_max = min(t_max, max(t1, t2))
            
            if t_min > t_max:
                return ray_origin + [ray_direction[0]*t, ray_direction[1]*t, ray_direction[2]*t]
            
            # Advance t: max(t_max - t, marching_delta) to ensure progress
            t = t + max(t_max - t, marching_delta)
            
            # Update ray position and recompute voxel indices
            ray_pos = [ray_origin[0] + ray_direction[0]*t, 
                       ray_origin[1] + ray_direction[1]*t, 
                       ray_origin[2] + ray_direction[2]*t]
            vi = int(ray_pos[0] / voxel_size)
            vj = int(ray_pos[1] / voxel_size)
            vk = int(ray_pos[2] / voxel_size)
        else:
            break
            
    return ray_pos
```

**Parameters:**
- `threshold`: The density threshold for the transfer function. Typical values depend on the specific transfer function and data normalization. Not given in the sources.
- `marching_delta`: The minimum step size to ensure progress. Not given in the sources.

**Edge Cases and Pitfalls:**
- Precision problems may occur when computing voxel intersections; the travel value is clamped using `FMax` to ensure progress [S1, unofficial].
- The map size is $(W-1) \times (H-1) \times (D-1)$, so the scalar field must have at least 2 voxels in each dimension [S1, unofficial].

## Subtopics

### [Chebyshev Distance Map Empty Space Skipping](3833-chebyshev-distance-map-empty-space-skipping.md)
Uses a pre-computed Chebyshev distance map to skip multiple empty voxels in any dimension along a ray. It replaces the binary occupancy check with a metric that quantifies the distance to the nearest occupied block, allowing the ray to jump directly to the next potential intersection. Unlike the occupancy map method, which resumes sampling after a fixed number of empty voxels, this method dynamically calculates the skip distance based on local geometry.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** The method reduces the number of density samples and transfer function evaluations [S1, unofficial].
- **Strengths:** Simple to implement, effective for volumes with large empty regions.
- **Limitations:** Binary check may skip over thin structures if the threshold is not carefully chosen. Precision issues can occur at voxel boundaries [S1, unofficial].

## Background
This method is implemented in the SRaimondi/VolumeRender repository as a preprocessing step to accelerate ray marching in volume rendering [S1, unofficial].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
