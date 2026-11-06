# Subvolume Tracing

Also called: Ray Marching Subvolume

## Overview
Subvolume Tracing renders a localized region of a volume by walking a fixed number of voxels deep along a ray and compositing the data from back to front [S1]. It differs from full volume ray casting by limiting the trace depth to a small, configurable `maxDepth` rather than calculating the full path through the volume cube [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Object position in volume space (`objPosStart`), camera position in volume space (`cameraPosVolSpace`), `maxDepth`, and `numLoops` [S1].
*   **Output:** A final color and alpha value for the pixel [S1].

**Data Structures:**
*   `curColor`: A 4-component vector (RGBA) initialized to transparent black `(0, 0, 0, 0)` [S1].
*   `curCoord`: A 3D coordinate in volume space representing the current sampling position [S1].
*   `stepCoord`: A 3D vector representing the step size between samples [S1].

**Parameters:**
*   `maxDepth`: Depth in volume space to trace. Typical value is `0.15` [S1].
*   `numLoops`: Number of sampling steps. Typical value is `10` (can be `400` on high-end PCs) [S1].

**Algorithm Steps:**
1.  **Coordinate Transformation:** Calculate the object's position in volume space using a `_WorldToVolume` matrix. Calculate the camera's position in volume space using the same matrix [S1].
2.  **Boundary Calculation:** Determine the front and back coordinates of the subvolume segment. `frontCoord` is `objPosStart`. `backCoord` is calculated as `frontCoord + (normalized camera direction * maxDepth)` [S1]. The camera direction is `normalize(cameraPosVolSpace - frontCoord)` [S1].
3.  **Step Vector Calculation:** Compute the step vector:
    $$stepCoord = \frac{frontCoord - backCoord}{numLoops}$$
    [S1]
4.  **Initialization:** Set `curCoord` to `backCoord` [S1].
5.  **Aliasing Prevention:** Add a per-pixel random offset to `curCoord` to avoid layer aliasing. The offset is `stepCoord * RandomFromPositionFast(objPosStart)` [S1].
6.  **Loop:** Iterate `numLoops` times:
    *   Sample the volume at `curCoord` to get intensity.
    *   Shade the intensity to get a `src` color/alpha pair.
    *   Composite `src` into `curColor` using alpha blending.
    *   Increment `curCoord` by `stepCoord` [S1].
7.  **Return:** Return `curColor` [S1].

**Alpha Blending Formula:**
The blending operation combines the source color (`src`) and destination color (`dst`) as follows [S1]:
$$res.rgb = (src.rgb \times src.a) + (dst.rgb - dst.rgb \times src.a)$$
$$res.a = src.a + (dst.a - dst.a \times src.a)$$
[S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def random_from_position_fast(pos):
    # Standard hash-based pseudo-random function returning a value in [0, 1) [G]
    x, y, z = pos
    n = int(math.sin(x * 12.9898 + y * 78.233 + z * 45.164) * 43758.5453)
    return (n - int(n)) % 1

def sample_vol(coord):
    # Standard 3D texture fetch operation [G]
    # Returns a scalar intensity value for the given volume coordinate
    return 0.5 + 0.5 * math.sin(coord[0] + coord[1] + coord[2])

def shade_vol(intensity):
    # Standard transfer function mapping intensity to RGBA [G]
    return (intensity, intensity, intensity, intensity)

def alpha_blend(dst, src):
    res_r = (src[0] * src[3]) + (dst[0] - dst[0] * src[3])
    res_g = (src[1] * src[3]) + (dst[1] - dst[1] * src[3])
    res_b = (src[2] * src[3]) + (dst[2] - dst[2] * src[3])
    res_a = src[3] + (dst[3] - dst[3] * src[3])
    return (res_r, res_g, res_b, res_a)

def subvolume_trace(obj_pos_start, camera_pos_vol_space, max_depth=0.15, num_loops=10):
    if num_loops <= 0:
        return (0.0, 0.0, 0.0, 0.0)
        
    front_coord = obj_pos_start
    cam_dir = (camera_pos_vol_space[0] - front_coord[0],
               camera_pos_vol_space[1] - front_coord[1],
               camera_pos_vol_space[2] - front_coord[2])
    cam_len = math.sqrt(cam_dir[0]**2 + cam_dir[1]**2 + cam_dir[2]**2)
    if cam_len == 0:
        cam_dir = (0.0, 0.0, 1.0)
    else:
        cam_dir = (cam_dir[0]/cam_len, cam_dir[1]/cam_len, cam_dir[2]/cam_len)
        
    back_coord = (front_coord[0] + cam_dir[0] * max_depth,
                  front_coord[1] + cam_dir[1] * max_depth,
                  front_coord[2] + cam_dir[2] * max_depth)
    
    step_coord = ((front_coord[0] - back_coord[0]) / num_loops,
                  (front_coord[1] - back_coord[1]) / num_loops,
                  (front_coord[2] - back_coord[2]) / num_loops)
    
    cur_color = (0.0, 0.0, 0.0, 0.0)
    cur_coord = back_coord
    
    random_offset = random_from_position_fast(obj_pos_start)
    cur_coord = (cur_coord[0] + step_coord[0] * random_offset,
                 cur_coord[1] + step_coord[1] * random_offset,
                 cur_coord[2] + step_coord[2] * random_offset)
    
    for _ in range(num_loops):
        intensity = sample_vol(cur_coord)
        shaded = shade_vol(intensity)
        cur_color = alpha_blend(cur_color, shaded)
        cur_coord = (cur_coord[0] + step_coord[0],
                     cur_coord[1] + step_coord[1],
                     cur_coord[2] + step_coord[2])
        
    return cur_color
```

## Subtopics

### Whole Volume Rendering
Extends subvolume tracing to render the entire volume by calculating the full path length through the volume cube [S1]. It differs by dynamically determining `numLoops` based on the distance through the volume rather than a fixed `maxDepth` [S1]. It sets `maxDepth` to `1.73` ($\sqrt{3}$), the maximum distance from a point on a unit cube to any other point [S1]. It trims front and back positions to be within the volume cube bounds [S1]. The number of loops is calculated as:
$$distanceInVoxels = length(UnitVolumeToIntVolume(frontPos - backPos))$$
$$numLoops = min(distanceInVoxels, maxSamples)$$
[S1]
where `maxSamples` is typically `400` to keep within bounds [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to `numLoops` [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Typical settings use `numLoops=10` for interactive performance, scalable to `400` on high-end PCs [S1].
*   **Strengths:** Fast rendering of local details; avoids aliasing with random offset [S1].
*   **Limitations:** Does not render the full volume; depth is fixed by `maxDepth` rather than volume bounds [S1].

## Background
Originating from Microsoft's Mixed Reality development documentation, this method is designed for GPU-based volume rendering in shaders [S1]. It is implemented in HLSL for vertex and pixel shaders to walk voxels and layer data back to front [S1].

## Sources
[S1] Volume rendering - Mixed Reality | Microsoft Learn, https://learn.microsoft.com/en-us/windows/mixed-reality/develop/native/volume-rendering, fetched 2026-10-27
