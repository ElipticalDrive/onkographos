# Ray Casting
Also called: Primary Ray Tracing

## Overview
Ray casting is the fundamental image synthesis technique that generates primary rays through pixels to determine visible objects and accumulates contributions along the ray to compute light distribution [S2, unofficial]. It is distinguished from full ray tracing by considering only primary (viewing) rays, excluding secondary rays for shadows, reflections, or refractions [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A camera model defining the ray origin $\mathbf{O}$ and direction $\mathbf{D}$, a scene representation (primitives or volume data), and image dimensions.
*   **Output:** A rendered image where each pixel contains the accumulated color or intensity.

**Data Structures**
*   **Ray:** Defined by origin $\mathbf{O}$ and direction $\mathbf{D}$. The direction is typically normalized.
*   **Scene:** A collection of primitives or a volumetric grid.

**Algorithm Steps**
1.  Iterate over all pixels in the image plane.
2.  For each pixel, generate a primary ray $R(t)$ using the camera model.
3.  Enumerate primitives or sample the volume along the ray $R(t)$.
4.  Accumulate the contribution of the intersected primitives or sampled volume data.

**Formulas**
The ray is defined parametrically as [S2, unofficial]:
$$ R(t) = \mathbf{O} + \mathbf{D} \cdot t $$
Where:
*   $R(t)$ is the ray at parameter $t$.
*   $\mathbf{O}$ is the ray origin (camera eye position).
*   $\mathbf{D}$ is the ray direction.
*   $t$ is the parameter along the ray ($t > 0$).

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_cast(scene, camera, width, height):
    image = empty_array(width, height, 3)
    
    # Iterate over all pixels
    for y in range(height):
        for x in range(width):
            # Generate primary ray for this pixel
            # Note: Exact pixel-to-ray mapping depends on the specific camera model
            # defined in the parent article [770-ray-generation-and-camera-models.md]
            origin = camera.get_origin(x, y)
            direction = camera.get_direction(x, y)
            
            # Enumerate primitives along the ray
            # In a full ray tracer, this involves intersection tests.
            # In volume rendering, this involves ray marching through the volume.
            contributions = []
            
            # For each primitive/volume sample along the ray:
            # 1. Find intersection or sample point
            # 2. Compute color/intensity contribution
            # 3. Add to contributions
            
            # Accumulate contributions
            pixel_color = accumulate(contributions)
            
            image[y, x] = pixel_color
            
    return image
```

**Edge Cases and Pitfalls**
*   **Precision:** Floating-point precision errors in ray origin or direction can cause artifacts, especially at large distances or small scales.
*   **Boundaries:** Rays may miss the scene entirely or intersect outside valid data ranges. Proper culling or clamping is required.
*   **Degenerate Input:** Zero-length rays or undefined directions must be handled.

## Subtopics

### [Camera Ray Generation](7456-camera-ray-generation.md)
Defines how primary rays are computed from camera parameters.

### [Grid Ray Sampler](3930-grid-ray-sampler.md)
Efficiently samples volumetric data along rays.

### [Compute Shader Ray Marching Setup](3487-compute-shader-ray-marching-setup.md)
Parallelizes ray casting on GPUs.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of primitives or samples per ray.
*   **Memory Complexity:** Depends on the scene representation.
*   **Strengths:** Simple implementation, good for primary visibility, parallelizable.
*   **Limitations:** No global illumination effects (shadows, reflections) without secondary rays.
*   **When to use:** Basic visibility determination, primary ray generation for further processing.
*   **When not to use:** When realistic lighting effects like shadows or reflections are required.

## Background
Ray casting was introduced as the basic form of ray tracing, focusing solely on primary rays [S1, unofficial]. It is a physically-based image synthesis technique known for excellent image quality [S2, unofficial]. While historically considered too slow for interactive applications, advancements have enabled real-time performance for both polygonal scenes and volume data sets [S2, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
