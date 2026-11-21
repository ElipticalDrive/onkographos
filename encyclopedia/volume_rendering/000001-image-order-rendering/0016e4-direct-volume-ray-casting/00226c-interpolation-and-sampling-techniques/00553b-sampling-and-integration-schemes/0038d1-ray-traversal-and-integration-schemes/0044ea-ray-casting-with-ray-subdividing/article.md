# Ray Casting with Ray Subdividing

Also called: Adaptive Ray Subdivision, Ray Splitting for Perspective Rendering, Ray Subdividing

## Overview
Ray Casting with Ray Subdividing is a technique to maintain adequate spatial sampling rates in volume rendering under perspective projection, where rays diverge as they travel deeper into the scene [S1, S2, unofficial]. It addresses the problem of undersampling distant volume slices by dynamically splitting rays when the local ray density falls below a threshold, ensuring that the footprint of each sample remains consistent with the desired resolution [S1, S2, unofficial]. Unlike fixed-step ray marching, this method adapts the integration step size based on geometric divergence rather than local data features [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A perspective camera ray $r(t) = \mathbf{o} + t\mathbf{d}$, a volume dataset $V$, and a target sampling density or maximum footprint size in world space.
- Output: A color and opacity value for the pixel, computed via front-to-back compositing.

**Data Structures:**
- Ray Queue: A list of active rays, where each ray stores its origin $\mathbf{o}$, direction $\mathbf{d}$, current depth $t$, and accumulated color/opacity.
- Split Threshold: A scalar value defining the maximum allowable distance between adjacent rays at a given depth to prevent aliasing.

**Steps:**
1. **Initialization**: For each pixel, generate an initial ray $r_0$ from the camera center through the pixel center. Add $r_0$ to the Ray Queue.
2. **Ray Divergence Check**: For each ray $r$ in the queue, calculate the distance to its nearest neighbor rays in the screen space at the current depth $t$. If the distance exceeds the Split Threshold, mark $r$ for splitting.
3. **Ray Splitting**: If a ray is marked for splitting, divide it into two or more sub-rays. The new origins are offset slightly in screen space, and the directions are adjusted to maintain perspective consistency. The original ray is removed from the queue, and the new sub-rays are added.
4. **Ray Merging (Optional)**: As proposed by Kreeger et al., rays can be merged when they converge or when the sampling density exceeds the requirement, reducing computational cost [S1, unofficial]. This requires tracking ray ancestry to correctly accumulate color and opacity.
5. **Integration**: For each ray in the queue, perform standard volume ray casting (sampling, interpolation, compositing) up to the next split/merge point or until the volume is exited.
6. **Final Compositing**: Combine the results from all leaf rays contributing to a pixel.

**Formulas:**
- Ray Divergence Metric: Let $d_{screen}(r_i, r_j)$ be the Euclidean distance between the screen-space projections of rays $r_i$ and $r_j$ at depth $t$. Split if:
  $$ d_{screen}(r_i, r_j) > \delta_{max} $$
  where $\delta_{max}$ is the maximum allowed screen-space footprint size.

**Parameters:**
- $\delta_{max}$: Typically set to the pixel size or a fraction thereof, depending on the desired anti-aliasing quality.

**Edge Cases and Pitfalls:**
- **Infinite Splitting**: If the threshold is too small, rays may split indefinitely. A maximum recursion depth or minimum ray length must be enforced.
- **Merging Complexity**: Merging rays requires careful handling of the compositing order to avoid artifacts. Not all implementations include merging; splitting is the primary mechanism [S1, unofficial].
- **Performance**: The overhead of managing the ray queue and checking divergence can be significant. Efficient spatial data structures (e.g., grids) may be needed to find nearest neighbors quickly.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_casting_with_subdivision(camera, volume, threshold):
    ray_queue = []
    for pixel in pixels:
        ray = Ray(camera.origin, pixel.direction)
        ray_queue.append(ray)
    
    while ray_queue:
        current_rays = ray_queue.pop_all()
        next_rays = []
        
        for ray in current_rays:
            # Check divergence against neighbors
            if is_divergent(ray, ray_queue, threshold):
                sub_rays = split_ray(ray, threshold)
                next_rays.extend(sub_rays)
            else:
                # Perform integration for this ray segment
                color, opacity = integrate_ray(ray, volume)
                # Store result for final compositing
                store_pixel_result(ray.pixel_id, color, opacity)
        
        ray_queue = next_rays
    
    return final_image
```

## Subtopics

### Adaptive Ray Subdivision
Focuses on the dynamic adjustment of ray density based on screen-space metrics. Choose when perspective distortion causes significant sampling variation. [Link](adaptive-ray-subdivision.md)

### Ray Splitting for Perspective Rendering
Emphasizes the geometric necessity of splitting in perspective views. Choose when fixed-step methods fail to capture distant details. [Link](ray-splitting-for-perspective-rendering.md)

## Operating characteristics
- **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the average number of splits per ray. Splits increase $S$ in distant regions.
- **Memory Complexity**: $O(N \cdot D)$, where $D$ is the maximum depth of ray splitting. The ray queue grows with the number of active sub-rays.
- **Performance**: Improves image quality in distant regions compared to fixed-step ray marching. However, it is more complex to implement and may have higher overhead due to queue management.
- **Strengths**: Maintains consistent sampling density across the entire volume, reducing aliasing in perspective views.
- **Limitations**: Complex implementation; potential for high memory usage if many rays split; merging is non-trivial.
- **When to Use**: Perspective volume rendering where distant details are important and fixed-step sampling leads to aliasing.
- **When Not to Use**: Parallel projection (where rays do not diverge) or when performance is critical and some aliasing is acceptable.

## Background
The concept of ray subdividing was introduced as a solution to the undersampling problem in perspective volume rendering [S2, unofficial]. It is considered an equivalent approach to Elliptical Weighted Average (EWA) splatting for raycasting, which handles screen-space footprint mapping [S1, unofficial]. Kreeger et al. proposed improvements involving optimal splitting and merging of rays [S1, unofficial]. The method is part of the broader family of image-order rendering techniques that adapt to geometric properties of the view [S1, S2, unofficial].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] Introduction to volume rendering, https://slidetodoc.com/introduction-to-volume-rendering-presented-by-zvi-devir/, fetched 2026-10-29 (unofficial)
