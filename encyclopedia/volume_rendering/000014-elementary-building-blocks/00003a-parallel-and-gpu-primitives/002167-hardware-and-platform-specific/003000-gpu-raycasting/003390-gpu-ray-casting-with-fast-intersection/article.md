# GPU Ray Casting with Fast Intersection

Also called: Single-pass GPU Ray Casting, Fast Intersection Test for Single-Object Rendering

## Overview
This method implements a single-pass volume ray casting algorithm on the GPU, designed to test each ray for intersection with the volume exactly once per rendering pass. It solves the problem of efficiently rendering a single object per pass by utilizing a simplified fast intersection test that avoids complex data structures [S1]. It is distinguished from multi-pass approaches by its single-pass nature and from hierarchical methods by its reliance on a simplified test suitable for single-intersection scenarios [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume data (emission, absorption, reflection, gradient textures), camera parameters (position, rotation, focal length), and projection plane coordinates.
*   **Outputs:** A 2D rendered image where each pixel corresponds to a processed ray.

**Data Structures:**
*   **Rotation Matrix:** A 3x3 matrix where $\vec{x}$, $\vec{y}$, and $\vec{z}$ are the column vectors representing the camera's orientation in the world coordinate frame [S1].
*   **Camera Offset:** A scalar $b$ representing half the total x-offset between two cameras (used for stereo or specific camera configurations) [S1].
*   **Distance:** A scalar $d$ representing the distance from the camera to the volume center [S1].

**Numbered Steps:**
1.  **Coordinate System Setup:** Center the world coordinate frame within the volume. Position the camera along the negative z-axis. Set the projection plane at a distance of 1 from the camera [S1].
2.  **Projection Plane Definition:** Define the projection plane in normalized coordinates ranging from $[-1, -1]$ to $[1, 1]$ [S1].
3.  **Ray Generation:** For each pixel, determine its normalized coordinates $(u, v)$ on the projection plane. Compute the ray origin and direction using the formulas below [S1].
4.  **Parallel Processing:** Assign each ray (representing one pixel) to a separate CUDA thread to achieve high parallelization [S1].
5.  **Intersection Test:** Perform the fast intersection test for the volume. The method supports rendering one object per pass, testing each ray for intersection only once [S1].

**Formulas:**
The ray origin $\text{orig\_ray}$ and direction $\text{dir\_ray}$ are computed as follows [S1]:

$$
\text{orig\_ray} = c_o \cdot \vec{x} + (-1) \cdot d \cdot \vec{z}
$$

$$
\text{dir\_ray} = \frac{u \cdot \vec{x} + v \cdot \vec{y} + f_o \cdot \vec{z}}{\| u \cdot \vec{x} + v \cdot \vec{y} + f_o \cdot \vec{z} \|}
$$

Where:
*   $c_o = \pm b$, where $b$ is half the total x-offset between two cameras [S1].
*   $d$ is the distance from the volume [S1].
*   $\vec{x}, \vec{y}, \vec{z}$ are the column vectors of the rotation matrix $R$ [S1].
*   $u, v$ are the normalized coordinates of the pixel on the projection plane [S1].
*   $f_o$ is the focal length, which can be adjusted [S1].

**Parameters:**
*   **Focal Length ($f_o$):** Adjustable parameter [S1].
*   **Camera Offset ($b$):** User-defined parameter [S1].
*   **Distance ($d$):** Distance from the volume center to the camera [S1].

**Edge Cases and Pitfalls:**
*   **Memory Limitations:** GPU memory is limited and not extendable. The renderer requires multiple textures (emission, absorption, reflection, gradient), which may exceed on-board memory for large datasets [S1].
*   **Single Object Constraint:** The method supports rendering one object per pass. Rendering multiple objects requires multiple passes [S1].
*   **Intersection Assumption:** The simplified fast intersection test is effective for scenarios involving a single intersection per ray. It may not be suitable for complex scenes with multiple intersections or non-rectilinear grids without modification [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CUDA Kernel for Ray Casting
__global__ void rayCastKernel(float* outputImage, int width, int height, 
                              float b, float d, float f_o, 
                              float3 x_vec, float3 y_vec, float3 z_vec) {
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;

    if (x < width && y < height) {
        // 1. Map pixel coordinates to normalized projection plane [-1, 1]
        float u = (2.0f * x / width) - 1.0f;
        float v = (2.0f * y / height) - 1.0f;

        // 2. Compute ray origin
        // orig_ray = co * x_vec + (-1) * d * z_vec
        float3 orig_ray;
        orig_ray.x = b * x_vec.x;
        orig_ray.y = b * x_vec.y;
        orig_ray.z = b * x_vec.z;
        
        // If co is negative, subtract instead of add
        // Note: The source specifies co = +/- b. Assuming positive b for this implementation.
        // If stereo, co would be -b for the second eye.
        orig_ray.x -= d * z_vec.x;
        orig_ray.y -= d * z_vec.y;
        orig_ray.z -= d * z_vec.z;

        // 3. Compute ray direction
        // dir_ray = (u * x_vec + v * y_vec + f_o * z_vec) / norm(...)
        float3 dir_ray;
        dir_ray.x = u * x_vec.x + v * y_vec.x + f_o * z_vec.x;
        dir_ray.y = u * x_vec.y + v * y_vec.y + f_o * z_vec.y;
        dir_ray.z = u * x_vec.z + v * y_vec.z + f_o * z_vec.z;

        float len = sqrt(dir_ray.x*dir_ray.x + dir_ray.y*dir_ray.y + dir_ray.z*dir_ray.z);
        dir_ray.x /= len;
        dir_ray.y /= len;
        dir_ray.z /= len;

        // 4. Perform Fast Intersection Test
        // The source mentions a simplified fast intersection test [37] for single intersection.
        // Implementation details of the specific test are not provided in the source text.
        // [G] Placeholder for the specific intersection logic from reference [37].
        // This step would typically involve checking if the ray intersects the volume bounds
        // and sampling the volume along the ray.
        
        float accumulatedColor = 0.0f; // Placeholder for accumulation result
        
        // 5. Write to output image
        outputImage[y * width + x] = accumulatedColor;
    }
}
```

## Subtopics

### Spectral DVR
An extension of the single-pass raycasting framework that handles refractive material selective absorption and dispersion using the Kubelka–Munk theory [S2]. It is chosen when spectral accuracy for refractive materials is required, addressing the low efficiency of multi-pass pipelines on current hardware [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels, as each pixel is processed by a separate thread [S1].
*   **Memory Complexity:** Requires storage for multiple volume textures (emission, absorption, reflection, gradient) [S1]. Memory is limited by GPU on-board capacity [S1].
*   **Performance:** Achieves high parallelization and fast rendering compared to CPU-based approaches [S1].
*   **Strengths:** Efficient single-pass rendering for single objects; avoids complex data structures [S1].
*   **Limitations:** Limited by GPU memory size; supports only one object per pass; simplified intersection test may not handle complex multi-intersection scenarios [S1].
*   **When to Use:** Rendering single objects with rectilinear 3D grids where memory fits in GPU texture memory [S1, S2].
*   **When Not to Use:** Scenes requiring multiple objects in a single pass or datasets too large for GPU memory without hierarchical subdivision [S1, S2].

## Background
The method is based on a simplified version of the fast intersection test introduced by [37] (not listed in sources) and implemented in a GPU-enabled volume renderer [S1]. It builds on the single-pass raycasting framework developed by Hadwiger et al. [S2]. Stegmaier and colleagues extended this framework to spectral DVR [S2]. The approach was implemented using CUDA to leverage GPU parallelization [S1].

## Sources
[S1] GPU-Enabled Volume Renderer for Use with MATLAB, https://doi.org/10.3390/digital4040049, fetched 2026-10-29
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
