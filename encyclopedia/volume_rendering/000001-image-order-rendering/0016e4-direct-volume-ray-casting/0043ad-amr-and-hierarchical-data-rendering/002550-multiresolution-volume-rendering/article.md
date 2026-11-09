# Multiresolution Volume Rendering

Also called: Direct rendering of multiresolution decomposed volume data, Multiresolution Volume Rendering with Adaptive Integration, Wavelet/Laplacian Pyramid Volume Rendering

## Overview
Multiresolution Volume Rendering is a direct volume ray casting method that utilizes hierarchical data structures, such as wavelet or Laplacian pyramid decompositions, to perform adaptive integration and early termination along rays [S1, unofficial]. It solves the computational cost problem in volume rendering by allowing the use of lower-resolution data for parts of the ray where accuracy requirements are lower or occlusion is high, thereby reducing the number of sampling steps required [S1, unofficial]. It sets itself apart from standard direct volume ray casting by dynamically selecting resolution levels based on the ray's progress and occlusion state, and from sibling methods like Cluster Hierarchy-based Volume Rendering by relying on mathematical decomposition (wavelets/pyramids) rather than spatial clustering of voxels [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A volume dataset decomposed into $k+1$ hierarchical levels ($m = 0, 1, \dots, k$), where level 0 is the highest resolution and level $k$ is the lowest [S1, unofficial].
*   A viewpoint and camera parameters defining the view plane and projection [S1, unofficial].
*   An illumination model to compute shading values [S1, unofficial].
*   A threshold $\epsilon$ for early termination [S1, unofficial].

**Outputs:**
*   A 2D image where each pixel contains the accumulated color and opacity derived from the ray integration [S1, unofficial].

### Data Structures
The method relies on a multiresolution decomposition representation. For a dataset decomposed into $k$ levels:
*   **Level $m$ Data:** Contains coefficients or voxel values at resolution level $m$.
*   **Reconstruction:** Values at any point $t$ along a ray are reconstructed from the coefficients of the selected level(s).

### Reconstruction Formulas
To sample the volume at a point, the method reconstructs the scalar value from the multiresolution coefficients.

**Wavelet Reconstruction (1D):**
To reconstruct a value $v(i)$ at index $i$ using a kernel size $q$:
$$ v(i) = \sum_{l=0}^{q-1} c_{u,w}(l) \phi(i-l) $$
where $c_{u,w}(l)$ are the wavelet coefficients and $\phi$ is the reconstruction kernel [S1, unofficial]. This is extended to 2D and 3D by summing over the respective kernel dimensions [S1, unofficial].

**Laplacian Pyramid Reconstruction:**
Calculated from coefficients $c_i^k$ at level $k$:
$$ c_i^k = c_{i}^{k+1} + \sum_{n} w(n - 2k) c_{i}^{k+1} $$
where $k$ is the level and $w$ is the pyramid kernel [S1, unofficial].

### Integration Model
The rendering equation is viewed as an inner product of the shading value $C(t)$ and the occlusion (accumulated opacity) $W(t)$ along a ray segment from $t_1$ to $t_2$ [S1, unofficial]:
$$ I = \int_{t_1}^{t_2} C(t) W(t) dt = \langle C(t), W(t) \rangle $$
When using data from different resolution levels, let $C_m(t)$ be the shading value derived from level $m$ and $W_j(t)$ be the occlusion derived from level $j$. The integration is denoted as:
$$ I_{m,j} = \langle C_m(t), W_j(t) \rangle $$
Level 0 corresponds to the highest resolution. Using $m > 0$ or $j > 0$ implies using lower-resolution approximations, which allows for larger sampling step sizes [S1, unofficial].

### Algorithm Steps
1.  **Ray Casting:** Cast rays from the viewpoint through each pixel on the view plane. Each ray is defined as a parametric line equation with parameter $t=0$ at the viewpoint [S1, unofficial].
2.  **Intersection:** Compute the intersection points $t_1$ and $t_2$ between the ray and the volume boundary [S1, unofficial].
3.  **Ray Tracing Loop:** Trace from $t_1$ to $t_2$ with a sampling step size $\Delta t$. The step size may vary based on the resolution level being used [S1, unofficial].
    *   **Reconstruction:** At the current sample point $t$, reconstruct the density/value using the selected resolution level's coefficients (Wavelet or Laplacian formulas above) [S1, unofficial].
    *   **Shading:** Compute the local color/shading value $C(t)$ using the illumination model and the reconstructed density [S1, unofficial].
    *   **Occlusion:** Compute the accumulated opacity $W(t)$ along the ray path from $t_1$ to $t$ [S1, unofficial].
    *   **Early Termination:** Check if the remaining occlusion $W(t)$ is below a threshold $\epsilon$. If $W(t) < \epsilon$, terminate the ray early [S1, unofficial].
    *   **Accumulation:** Add the contribution $C(t) W(t) \Delta t$ to the pixel color [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiresolution_volume_rendering(volume_levels, viewpoint, camera, epsilon=1e-4):
    """
    volume_levels: List of data structures for levels 0 to k.
                   Level 0 is highest resolution, Level k is lowest.
                   Each level supports a reconstruct(value, point) function.
    viewpoint: 3D point representing the camera center.
    camera: Object providing ray generation for each pixel.
    epsilon: Threshold for early termination.
    """
    image_width, image_height = camera.resolution
    image = zeros((image_height, image_width, 3))

    for y in range(image_height):
        for x in range(image_width):
            ray = camera.get_ray(x, y)
            
            # Step 2: Compute intersection with volume boundary
            t1, t2 = compute_intersection(ray, volume_boundary)
            
            if t1 > t2:
                continue
                
            accumulated_color = [0.0, 0.0, 0.0]
            accumulated_opacity = 0.0
            current_t = t1
            
            # Step 3: Ray tracing loop
            # Note: Step size delta_t may be adjusted based on resolution level
            delta_t = get_initial_step_size() 
            
            while current_t < t2:
                # Determine resolution level based on current_t or strategy
                # For simplicity, assume a fixed level or adaptive selection logic
                level = select_level(current_t, accumulated_opacity)
                
                # Reconstruct density/value at current_t using selected level
                density = volume_levels[level].reconstruct(density_field, ray.origin + ray.direction * current_t)
                
                # Compute shading color C(t)
                color = compute_shading(density, ray.direction, viewpoint)
                
                # Compute opacity alpha for this step
                alpha = compute_opacity(density, delta_t)
                
                # Update accumulated opacity W(t)
                # W(t) represents the transmittance/occlusion factor
                # In standard volume rendering, transmittance T(t) = exp(-sigma_a * t)
                # Here, we track the remaining contribution potential
                remaining_opacity = 1.0 - accumulated_opacity
                
                # Early termination check: if remaining contribution is negligible
                if remaining_opacity < epsilon:
                    break
                
                # Composite: C_out = C_in + C(t) * alpha * T(t)
                # Assuming pre-multiplied alpha or standard blending
                contribution = color * alpha * (1.0 - accumulated_opacity)
                accumulated_color += contribution
                accumulated_opacity += alpha * (1.0 - accumulated_opacity)
                
                current_t += delta_t

            image[y, x] = accumulated_color

    return image
```

### Parameters and Typical Values
*   **$\epsilon$ (Early Termination Threshold):** A small positive value. Not given in the sources.
*   **Kernel Size $q$:** For wavelet reconstruction. Not given in the sources.
*   **Resolution Levels:** In experiments, a CT dataset was decomposed into 5 levels ($k=4$ or similar, source says "five hierarchical levels"), with reconstruction carried out up to levels 2, 1, and 0 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Occlusion Decay:** As $t$ increases, the occlusion function $W(t)$ (or transmittance) approaches zero. If early termination is not implemented, non-zero shading values at large $t$ contribute negligibly to the final color, wasting computation [S1, unofficial].
*   **Accuracy vs. Resolution:** Using lower resolution data for shading or occlusion reduces computational cost because a larger step size can be used. However, this affects accuracy. Mixing levels can introduce errors. For example, in experiments, using shading from level 2 and occlusion from level 2 ($C_2/W_2$) resulted in an error of -0.027605, while using shading from level 0 and occlusion from level 0 ($C_0/W_0$) resulted in an error of 0.011057 [S1, unofficial]. Rendering low-resolution data with high-resolution shading colors can result in better image quality than the reverse [S1, unofficial].

## Subtopics

### Adaptive Integration
A technique within multiresolution rendering where the sampling step size and resolution level are adjusted dynamically along the ray. It is chosen when computational efficiency is prioritized over uniform accuracy. It differs from fixed-step rendering by varying the granularity of the integration based on local data complexity and occlusion. See [Multiresolution Volume Rendering](10962-multiresolution-volume-rendering.md).

### Early Termination
A optimization where ray tracing stops when the accumulated opacity (or remaining transmittance) falls below a threshold. It is chosen to avoid calculating negligible contributions from distant voxels. It differs from full integration by trading off completeness for speed. See [Multiresolution Volume Rendering](10962-multiresolution-volume-rendering.md).

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** Reduced compared to standard direct volume ray casting due to adaptive integration and early termination. The use of lower resolution data allows for larger step sizes, reducing the number of samples per ray [S1, unofficial].
*   **Memory Complexity:** Higher than standard volume rendering due to the storage of multiresolution coefficients (wavelet or pyramid levels) in addition to the base data [S1, unofficial].

### Measured Performance
*   In experiments with a CT data set decomposed into 5 levels, reconstruction was carried out up to levels 2, 1, and 0 [S1, unofficial].
*   Errors were observed when mixing levels. Table 4.2 in the source shows errors for different combinations of shading ($C$) and occlusion ($W$) levels. For instance, $C_2/W_2$ had an error of -0.027605, while $C_0/W_0$ had an error of 0.011057 [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** Reduces computational cost by using larger step sizes at lower resolutions; facilitates early termination; allows for hierarchical data management [S1, unofficial].
*   **Limitations:** Accuracy is affected by the choice of resolution levels; mixing levels can introduce visible errors; requires pre-processing to decompose the volume data [S1, unofficial].

### When to Use
Use when rendering large volume datasets where computational efficiency is critical and some loss of accuracy is acceptable. It is particularly effective when the volume data has natural hierarchical structures or can be efficiently decomposed.

### When Not to Use
Avoid when high-fidelity, uniform accuracy is required across the entire volume, or when the overhead of multiresolution decomposition and level selection outweighs the benefits of adaptive sampling.

## Background

### Origin
The method is described in the work "Direct rendering of multiresolution decomposed volume data" [S1, unofficial].

### History and Development
The approach builds on the principles of multiresolution analysis (wavelets, Laplacian pyramids) applied to volume rendering. It leverages the hierarchical structure of decomposed data to facilitate adaptive integration. The specific analysis of integration in the multiresolution domain, including the formulation of $I_{m,j}$, is detailed in the source [S1, unofficial].

### Applications
Used in the rendering of medical imaging data, such as CT datasets, where large volumes need to be visualized efficiently [S1, unofficial].

## Sources

[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
