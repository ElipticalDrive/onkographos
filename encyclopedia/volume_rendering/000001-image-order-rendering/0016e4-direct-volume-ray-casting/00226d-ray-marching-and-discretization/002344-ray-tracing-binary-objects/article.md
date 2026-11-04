# Ray Tracing (Binary Objects)
Also called: Binary Object Ray Tracing, Binary Ray Casting, First-hit Ray Casting for Binary Volumes, Tuy's binary ray casting

## Overview
A direct volume rendering method for visualizing opaque surfaces contained in binary volumetric data by determining only the first intersection points with a surface [S2, unofficial]. It differs from gray-level volume rendering (e.g., [Front-to-Back Volume Rendering](2350-front-to-back-volume-rendering.md)) by stopping the ray traversal immediately upon hitting the object boundary, rather than compositing transmittance and color along the entire ray path [S1, unofficial].

## How it works
**Inputs:** Binary volumetric dataset $V$ where $V(x,y,z) \in \{0, 1\}$; camera parameters (viewing direction, image plane resolution); lighting model (light direction, depth shading function).
**Outputs:** 2D image where each pixel holds the color of the first surface point hit by the viewing ray.

**Data Structures:**
*   **Volume $V$:** 3D array of integers. $V[i,j,k] = 1$ if the voxel belongs to the object, $0$ otherwise.
*   **Ray:** Defined by origin $O$ (camera center or pixel center) and direction $D$.

**Steps:**
1.  **Ray Casting:** For each pixel in the image plane, construct a ray $R(t) = O + tD$ traveling from the screen toward the object (viewing point) [S1, unofficial]. This is the inverse of light-source-to-object tracing [S1, unofficial].
2.  **Sampling:** Resample the volume along the viewing ray at evenly spaced sample points [S2, unofficial]. The sample value at a point is determined by the nearest voxel in $V$ [S2, unofficial].
3.  **Intersection Detection:** Test voxels sequentially along the ray to identify the transition from exterior ($0$) to interior ($1$) [S1, unofficial].
4.  **Termination:** Find the first sample with a value of $1$ [S2, unofficial]. Stop traversal immediately [S1, unofficial].
5.  **Shading:** Determine the pixel color by shading the intersected surface point [S2, unofficial]. Shading uses depth, light direction, and an estimate of the surface normal [S1, unofficial].

**Formulas:**
*   **Ray Equation:** $R(t) = O + tD$, where $t > 0$ [S1, unofficial].
*   **Sample Value:** $S(t) = V(\text{nearest\_voxel}(R(t)))$ [S2, unofficial].
*   **Intersection Condition:** The first $t$ such that $S(t) = 1$ [S2, unofficial].
*   **Normal Estimation:** Not explicitly defined in sources, but described as an "unnormalized estimate of the gradient" or derived from the binary transition [S1, unofficial]. [G] In practice, the normal is often approximated by the difference between the first hit voxel and its neighbors, or by the ray direction if surface orientation is assumed perpendicular to the view.

**Parameters:**
*   **Sampling Step:** Evenly located sample points [S2, unofficial]. Size not given in the sources.
*   **Shading Model:** Depth shading, light direction, surface normal estimate [S1, unofficial].

**Edge Cases:**
*   **Precision:** Boundary detection does not require interpolation, avoiding interpolation artifacts [S1, unofficial].
*   **Degenerate Input:** Binary data must be strictly 0 or 1 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_trace_binary_volume(volume, camera, image_width, image_height):
    image = empty array of size (image_height, image_width, 3)
    
    for y in range(image_height):
        for x in range(image_width):
            # Construct ray from pixel to viewing point
            ray_origin = camera.get_pixel_origin(x, y)
            ray_direction = camera.get_pixel_direction(x, y)
            
            # Traverse ray from screen toward object
            t = 0.0
            step_size = camera.get_step_size() # Not given in sources; must be defined by implementation
            
            hit = False
            hit_point = None
            
            while t < camera.max_distance:
                # Sample volume at current ray position
                sample_value = volume.get_nearest_voxel(ray_origin + ray_direction * t)
                
                if sample_value == 1:
                    hit = True
                    hit_point = ray_origin + ray_direction * t
                    break
                
                t += step_size
            
            if hit:
                # Estimate normal at hit point
                normal = estimate_normal(volume, hit_point)
                
                # Shade based on depth, light direction, and normal
                color = shade(hit_point, normal, camera.light_direction, t)
                image[y, x] = color
            else:
                image[y, x] = background_color
                
    return image

def estimate_normal(volume, point):
    # [G] Approximate normal using finite differences on binary volume
    # This is a standard implementation detail not explicitly in sources
    dx = volume.get_nearest_voxel(point + vec3(1,0,0)) - volume.get_nearest_voxel(point - vec3(1,0,0))
    dy = volume.get_nearest_voxel(point + vec3(0,1,0)) - volume.get_nearest_voxel(point - vec3(0,1,0))
    dz = volume.get_nearest_voxel(point + vec3(0,0,1)) - volume.get_nearest_voxel(point - vec3(0,0,1))
    return normalize(vec3(dx, dy, dz))

def shade(point, normal, light_dir, depth):
    # Simple depth and diffuse shading
    diffuse = max(dot(normal, normalize(light_dir)), 0.0)
    # Depth shading: darker further away
    depth_factor = 1.0 - (depth / max_depth)
    return vec3(diffuse * depth_factor)
```

## Subtopics

### Binary Object Ray Tracing
Same method, emphasizing the binary nature of the data.

### Tuy's Binary Ray Casting
The original 1984 method by Tuy et al., characterized by screen-to-object traversal [S1, S2, unofficial].

### First-hit Ray Casting
Emphasizes the termination condition upon first intersection [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of pixels and the average distance to the first surface hit. Worst case is $O(N \cdot L)$ where $N$ is pixels and $L$ is ray length in voxels.
*   **Memory Complexity:** $O(V)$ for the volume data. No accumulation buffers needed.
*   **Strengths:** No interpolation or boundary detection required [S1, unofficial]; efficient for opaque objects; direct surface visualization [S2, unofficial].
*   **Limitations:** Only works for binary data; no internal structure visualization; shading quality depends on normal estimation [S1, unofficial].
*   **When to use:** Visualizing opaque surfaces in binary volumetric data (e.g., segmented medical masks) [S2, unofficial].
*   **When not to use:** Gray-level volumes, transparent objects, or when internal structure is needed [S1, unofficial].

## Background
Originated in 1984 with Tuy et al. [S1, S2, unofficial]. It was proposed as a method for visualizing surfaces in binary volumetric data, distinct from cuberille methods that recover shading from binary objects [S1, unofficial]. It contrasts with Levoy's 1988 raycasting algorithm for gray-level volumes, which composites along the entire ray [S2, unofficial]. Used in medical data visualization for surface rendering [S1, S2, unofficial].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
