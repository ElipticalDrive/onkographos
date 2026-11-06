# Backward Mapping (Ray Tracing)
Also called: Image-order volume rendering, Ray-tracing volume rendering
## Overview
Backward mapping is a volume rendering technique that forms the image by iterating over output pixels and projecting rays through the data volume, where intersecting voxels influence the pixel value [S1, unofficial]. It is distinguished from forward mapping (object-order) methods, which project voxels directly onto the image plane, by its natural handling of hidden surface removal and consistent sampling density relative to the screen [S1, unofficial]. It builds on [Volume Ray Casting](3161-volume-ray-casting.md) by defining the specific image-order traversal strategy.
## How it works
**Inputs and Outputs:**
- Input: Volumetric dataset $V$ (scalar/vector fields), camera parameters (view direction, up vector, position), image dimensions ($W \times H$).
- Output: Image buffer $I$ of size $W \times H$.

**Data Structures:**
- Volume $V$: Stored in its native coordinate system (e.g., voxel grid). No explicit surface representation is required [S1, unofficial].
- Ray: Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.

**Steps:**
1. **Ray Generation:** For each pixel $(u, v)$ in the image plane, compute the corresponding ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ that passes through the pixel center [S1, unofficial].
2. **Intersection:** Determine the entry and exit points of the ray with the volume bounding box.
3. **Sampling:** Discretize the ray segment within the volume into $N$ sample points. At each sample point $x_l$, sample the volume data $c(x_l)$ (color/density) and opacity $\alpha_l$ [S2, unofficial].
4. **Integration:** Accumulate the contribution of each sample to the pixel color using the volume rendering integral approximation [S2, unofficial]:
   $$ I(\hat{x}) = \sum_{l=0}^{L-1} c(x_l) \alpha_l \prod_{j=0}^{l-1} (1 - \alpha_j) $$
   where $L$ is the number of samples along the ray, $c(x_l)$ is the color at sample $l$, and $\alpha_l$ is the opacity [S2, unofficial]. The term $\prod_{j=0}^{l-1} (1 - \alpha_j)$ represents the transmittance from the ray origin to sample $l$.
5. **Blending:** The result for each pixel is the accumulated color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def backward_mapping_volume_rendering(volume, camera, width, height):
    image = zeros((height, width, 3))
    
    # Precompute ray directions for all pixels based on camera parameters
    ray_directions = compute_ray_directions(camera, width, height)
    
    for y in range(height):
        for x in range(width):
            ray_origin = camera.position
            ray_direction = ray_directions[y, x]
            
            # Determine intersection with volume bounding box
            t_min, t_max = intersect_volume(ray_origin, ray_direction, volume.bounds)
            
            if t_max < t_min:
                continue # Ray misses volume
                
            # Discretize ray
            num_samples = compute_num_samples(t_max - t_min, volume.spacing)
            dt = (t_max - t_min) / num_samples
            
            accumulated_color = zeros(3)
            transmittance = 1.0
            
            for i in range(num_samples):
                t = t_min + i * dt
                sample_pos = ray_origin + t * ray_direction
                
                # Sample volume data at sample_pos
                # Returns color c and opacity alpha
                c, alpha = sample_volume(volume, sample_pos)
                
                # Apply volume rendering equation
                # Contribution = color * opacity * transmittance
                contribution = c * alpha * transmittance
                accumulated_color += contribution
                
                # Update transmittance
                transmittance *= (1.0 - alpha)
                
                # Early ray marching termination if fully opaque
                if transmittance < epsilon:
                    break
            
            image[y, x] = accumulated_color
            
    return image
```

**Parameters:**
- `num_samples`: Number of samples per ray. Depends on volume resolution and view distance.
- `epsilon`: Small threshold for early ray termination.

**Edge Cases:**
- Ray misses volume: Skip pixel or assign background color.
- Degenerate volume: Empty volume results in transparent image.

## Operating characteristics
- **Time Complexity:** $O(W \cdot H \cdot N)$, where $W, H$ are image dimensions and $N$ is samples per ray.
- **Memory Complexity:** $O(W \cdot H)$ for the image buffer, plus storage for the volume.
- **Strengths:** Natural handling of occlusion; consistent visual quality across view angles; simple implementation [S1, unofficial].
- **Limitations:** Can be slow for high-resolution volumes due to per-pixel ray casting; sampling artifacts if $N$ is insufficient [S2, unofficial].
- **When to use:** When visual accuracy and simplicity are prioritized over performance, or when hardware acceleration is available.

## Background
Backward mapping was introduced in the late 1980s as a fundamental approach to volume rendering, allowing direct examination of data in its native voxel format without surface simplification [S1, unofficial]. It is widely used in medical imaging and scientific visualization [S1, unofficial].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
