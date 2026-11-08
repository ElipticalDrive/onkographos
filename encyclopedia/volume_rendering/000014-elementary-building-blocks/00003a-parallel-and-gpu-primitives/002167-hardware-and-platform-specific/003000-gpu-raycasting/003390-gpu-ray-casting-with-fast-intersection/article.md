# GPU Ray Casting with Fast Intersection

Also called: Single-pass GPU Ray Casting, Fast Intersection Test for Single-Object Rendering

## Overview
This method implements a single-pass GPU ray casting technique that tests each ray for intersection with the volume exactly once per rendering pass, supporting one object per pass [S1]. It solves the problem of efficiently rendering volumetric data by avoiding complex hierarchical data structures for empty space skipping, relying instead on a simplified fast intersection test effective for single-intersection scenarios [S1]. It differs from its sibling [GPU-based Ray Casting](7131-gpu-based-ray-casting.md) by explicitly optimizing for a single intersection per ray and utilizing a specific camera projection geometry defined in normalized coordinates, whereas general GPU ray casting may employ more complex traversal structures or multiple passes [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric data (emission, absorption, reflection, gradients) stored in GPU memory; camera parameters (distance $d$, offset $b$, rotation matrix $R$, focal length $f_o$); image resolution.
*   **Output:** Rendered 2D image where each pixel corresponds to a processed ray.

**Data Structures:**
*   **Camera Geometry:** Defined by a rotation matrix $R$ with column vectors $\vec{x}, \vec{y}, \vec{z}$, distance $d$, and offset $b$.
*   **Projection Plane:** Normalized coordinates $(u, v)$ ranging from $[-1, -1]$ to $[1, 1]$ [S1].
*   **Ray Representation:** Origin $\text{orig\_ray}$ and direction $\text{dir\_ray}$ [S1].

**Formulas:**
The ray origin and direction are computed as follows [S1]:
$$
\text{orig\_ray} = \text{co} \cdot \vec{x} + (-1) \cdot d \cdot \vec{z}
$$
$$
\text{dir\_ray} = \frac{u \cdot \vec{x} + v \cdot \vec{y} + f_o \cdot \vec{z}}{\|u \cdot \vec{x} + v \cdot \vec{y} + f_o \cdot \vec{z}\|}
$$
Where:
*   $\text{co} = \pm b$ represents half the total x-offset between two cameras [S1].
*   $d$ is the distance from the volume [S1].
*   $f_o$ is the focal length [S1].
*   $u, v$ are the normalized coordinates of the pixel on the projection plane [S1].
*   $\vec{x}, \vec{y}, \vec{z}$ are the column vectors of the rotation matrix $R$ [S1].

**Steps:**
1.  **Camera Setup:** Position the camera along the negative z-axis with the projection plane at a distance of 1 from the camera in the world coordinate frame centered within the volume [S1].
2.  **Thread Mapping:** Assign one CUDA thread to each pixel (ray) to maximize parallelization [S1].
3.  **Ray Generation:** For each thread, calculate the normalized coordinates $(u, v)$ corresponding to the pixel. Compute the ray origin $\text{orig\_ray}$ and direction $\text{dir\_ray}$ using the formulas above [S1].
4.  **Intersection Test:** Perform a single intersection test between the generated ray and the volume. The method supports only one object per pass and assumes a single intersection per ray [S1].
5.  **Rendering:** Accumulate light intensity as the ray traverses the volume based on the intersection result.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for a single CUDA thread processing one pixel
def process_pixel(u, v, d, b, f_o, R):
    # Extract column vectors from rotation matrix R
    x_vec = R.column(0)
    y_vec = R.column(1)
    z_vec = R.column(2)
    
    # Calculate co (half x-offset)
    co = b # Note: sign depends on specific camera configuration, typically ±b
    
    # Calculate ray origin
    # orig_ray = co * x_vec + (-1) * d * z_vec
    orig_ray = scale_add(x_vec, co, z_vec, -d)
    
    # Calculate ray direction
    # numerator = u * x_vec + v * y_vec + f_o * z_vec
    numerator = scale_add(scale_add(x_vec, u, y_vec, v), z_vec, f_o)
    
    # Normalize direction
    dir_ray = normalize(numerator)
    
    # Perform single intersection test with volume
    # This step uses a simplified fast intersection test effective for single intersections
    # [S1] mentions avoiding complex data structures
    intersection_hit = fast_intersection_test(orig_ray, dir_ray, volume_data)
    
    if intersection_hit:
        # Accumulate light intensity along the ray through the volume
        # [S1] mentions emission, absorption, reflection, and gradient volumes
        color = integrate_volume(orig_ray, dir_ray, volume_data)
    else:
        color = background_color
        
    return color
```

**Parameters:**
*   $d$: Distance from the volume. Not given in the sources.
*   $b$: Half the total x-offset between two cameras. User-defined [S1].
*   $f_o$: Focal length. Adjustable [S1].
*   $R$: Rotation matrix. User-defined [S1].

**Edge Cases and Pitfalls:**
*   **Single Object Limitation:** The method supports only one object per pass [S1].
*   **Single Intersection Assumption:** The fast intersection test is effective only for scenarios involving a single intersection per ray [S1].
*   **Memory Limitations:** GPU memory is limited, requiring careful management of multiple volume textures (emission, absorption, reflection, gradients) [S1].

## Subtopics

### Spectral DVR
Spectral DVR is an extension of the single-pass raycasting framework that handles refractive material selective absorption and dispersion using the Kubelka–Munk theory [S2]. It is chosen when spectral accuracy for refractive materials is required, differing from the standard method by incorporating spectral physics into the single-pass traversal [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels, as each pixel is processed by a separate thread [S1].
*   **Memory Complexity:** Requires storage for multiple volume textures (emission, absorption, reflection, gradients) in GPU memory [S1].
*   **Performance:** Achieves high parallelization and fast rendering compared to CPU-based approaches [S1].
*   **Strengths:** Efficient for single-object rendering with single intersections; avoids complex data structures [S1].
*   **Limitations:** Limited to one object per pass; assumes single intersection per ray; GPU memory constraints [S1].
*   **When to Use:** When rendering a single volumetric object with simple intersection properties on a GPU [S1].
*   **When Not to Use:** For multi-object scenes or complex intersection scenarios requiring hierarchical traversal [S1].

## Background
The method was implemented as part of a GPU-enabled volume renderer for use with MATLAB [S1]. It builds on the single-pass raycasting framework introduced by Hadwiger et al. [S2], which exploited a two-level hierarchical grid for empty space skipping. However, this specific implementation simplifies the intersection test to avoid complex data structures, focusing on efficiency for single-object scenarios [S1, S2]. The approach leverages CUDA for parallel processing [S1].

## Sources
[S1] GPU-Enabled Volume Renderer for Use with MATLAB, https://doi.org/10.3390/digital4040049, fetched 2026-10-29
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
