# Binary Ray Casting

Also called: Zero-order interpolation ray casting

## Overview
Binary Ray Casting is a direct volume rendering method for generating images of surfaces within binary volumetric data without explicit boundary detection or hidden-surface removal [S1, unofficial][S2]. It distinguishes itself from higher-order interpolation methods by using zero-order interpolation, where voxel values are treated as discrete constants (0 or 1) rather than continuous fields [S1, unofficial]. This makes it suitable for binary datasets where the distinction between object and background is absolute, avoiding the smoothing artifacts inherent in linear or tri-linear interpolation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A binary volumetric dataset $V$ where $V(\mathbf{i}) \in \{0, 1\}$ for voxel indices $\mathbf{i}$.
- Input: Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$, image plane resolution).
- Output: A 2D image where each pixel is colored based on the first surface intersection.

**Data Structures:**
- The volume is stored as a 3D array of integers or bits. No auxiliary structures (like octrees or grids) are required by the core algorithm [S1, unofficial].

**Steps:**
1. For each pixel $(u, v)$ in the image plane, compute the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ in world space.
2. Initialize the ray parameter $t = 0$ and the step size $\Delta t$. The step size $\Delta t$ determines the resolution of the search. A common choice is the diagonal length of a voxel divided by the number of steps per voxel, or simply the voxel spacing if the ray is aligned with axes [G].
3. While $t$ is within the volume bounds:
    a. Compute the current sample position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.
    b. Map $\mathbf{p}$ to the nearest voxel index $\mathbf{i}$ using floor or round operations (zero-order interpolation).
    c. Retrieve the value $v = V(\mathbf{i})$.
    d. If $v = 1$ (object) and the previous value was 0 (background), a surface intersection is detected.
    e. Refine the intersection point if necessary (e.g., linear interpolation between the last 0 and first 1 sample) to reduce stair-stepping artifacts [G].
    f. Break the loop.
4. If an intersection is found, compute the surface normal $\mathbf{n}$ at the hit point. Since the data is binary, the normal is typically derived from the gradient of the discrete field or by assuming the normal is perpendicular to the ray at the transition point [G].
5. Shade the pixel using a local lighting model (e.g., Phong) with the computed normal.
6. If no intersection is found within the volume bounds, color the pixel as background.

**Parameters:**
- Step size $\Delta t$: Controls the trade-off between accuracy and performance. Smaller steps reduce aliasing but increase computation. Not given in the sources.
- Threshold: Values $\ge 1$ are object; values $< 1$ are background [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
- **Aliasing:** Zero-order interpolation causes significant aliasing (stair-stepping) artifacts, especially at oblique angles. This is the primary limitation compared to higher-order methods [S1, unofficial].
- **Precision:** Floating-point errors in ray traversal may cause missed voxels or double-counting if $\Delta t$ is not chosen carefully relative to voxel size [G].
- **Thin Structures:** If the object thickness is smaller than the step size, the surface may be missed entirely.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binary_ray_casting(volume, camera, width, height):
    image = zeros((height, width, 3))
    voxel_size = volume.spacing # Assume uniform spacing
    step_size = voxel_size * 0.5 # [G] Typical step size to ensure coverage
    
    for y in range(height):
        for x in range(width):
            ray_origin, ray_direction = camera.get_ray(x, y)
            t = 0.0
            hit = False
            prev_val = 0
            
            while t < volume.max_distance:
                # Compute sample position
                pos = ray_origin + t * ray_direction
                
                # Map to voxel index (zero-order interpolation)
                voxel_idx = floor(pos / voxel_size)
                
                # Check bounds
                if not volume.is_inside(voxel_idx):
                    break
                
                # Get value
                val = volume.get(voxel_idx)
                
                # Detect transition from background (0) to object (1)
                if prev_val == 0 and val == 1:
                    hit = True
                    # Refine intersection [G]
                    # Linear interpolation between t_prev and t
                    t_hit = t - step_size + step_size * (0 - prev_val) / (val - prev_val)
                    hit_pos = ray_origin + t_hit * ray_direction
                    
                    # Compute normal [G]
                    # Simple approach: gradient of binary field
                    # Or assume normal is perpendicular to ray at hit_pos
                    normal = compute_normal(volume, hit_pos)
                    
                    # Shade
                    image[y, x] = shade(hit_pos, normal, ray_direction)
                    break
                
                prev_val = val
                t += step_size
            
            if not hit:
                image[y, x] = background_color
                
    return image
```

## Subtopics

### [Ray Marching with Adaptive Step Size](19509-ray-marching-with-adaptive-step-size.md)
Uses variable step sizes to improve performance and accuracy. Choose when the object has varying density or complexity.

### [Intersection Refinement and Search](22010-intersection-refinement-and-search.md)
Provides methods to refine the intersection point after a coarse step. Choose when higher precision is needed than zero-order interpolation allows.

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the average number of steps per ray. $S$ depends on the volume depth and step size.
- **Memory Complexity:** $O(V)$, where $V$ is the volume size. No auxiliary structures are needed.
- **Strengths:** Simple implementation; no preprocessing required; fast for binary data; no boundary extraction overhead.
- **Limitations:** High aliasing artifacts due to zero-order interpolation; sensitive to step size; cannot represent smooth surfaces or gradients.
- **When to use:** Binary datasets where speed is critical and visual quality (aliasing) is acceptable.
- **When not to use:** Continuous scalar fields; high-quality rendering requirements; thin structures smaller than step size.

## Background
Binary Ray Casting was developed to generate images of surfaces in binary volumetric data without the need for explicit boundary detection and hidden-surface removal [S1, unofficial]. It is classified as a direct volume rendering algorithm using zero-order interpolation [S1, unofficial]. It is often compared with Discrete Ray Casting, which uses a 3D Bresenham-like algorithm to traverse a discrete representation of the ray [S1, unofficial]. The method is relevant in medical and scientific visualization where data is often thresholded into binary masks (e.g., bone vs. tissue) [S2].

## Sources
[S1] Volume Rendering, https://www.cs.unh.edu/~cs880/visualization/Celebi-Volume%20Rendering.pdf, fetched 2026-10-26 (unofficial)
[S2] graphXDoses/VolumeInspector, https://github.com/graphXDoses/VolumeInspector, fetched 2026-10-28
