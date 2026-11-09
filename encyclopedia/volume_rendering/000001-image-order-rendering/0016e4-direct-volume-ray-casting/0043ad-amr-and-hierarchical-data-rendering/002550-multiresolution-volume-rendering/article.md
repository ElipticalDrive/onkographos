# Multiresolution Volume Rendering

Also called: Direct rendering of multiresolution decomposed volume data, Multiresolution Volume Rendering with Adaptive Integration, Wavelet/Laplacian Pyramid Volume Rendering

## Overview
Multiresolution Volume Rendering is a direct volume ray casting technique that utilizes hierarchical data structures, such as wavelet or Laplacian pyramid decompositions, to perform adaptive integration and early termination along rays [S1, unofficial]. By allowing the renderer to switch between resolution levels based on occlusion or accuracy requirements, it reduces computational cost by using larger step sizes and fewer samples in lower-resolution regions [S1, unofficial]. It sets itself apart from standard direct volume ray casting by dynamically adapting the sampling density and data source resolution per ray, and from [Cluster Hierarchy-based Volume Rendering](10972-cluster-hierarchy-based-volume-rendering.md) by operating on a continuous multiresolution decomposition rather than discrete spatial clusters.

## How it works
The method extends the standard direct volume ray casting loop [G] by integrating a multiresolution data structure and an adaptive sampling strategy.

### Inputs and Outputs
*   **Inputs**: Multiresolution volume data (e.g., wavelet coefficients or Laplacian pyramid levels), camera parameters (viewpoint, projection), transfer function mapping scalar values to color and opacity, and integration parameters (epsilon for early termination).
*   **Outputs**: A 2D image where each pixel contains the accumulated color and opacity.

### Data Structures
The core data structure is a multiresolution decomposition of the volume. Two common forms are cited:
1.  **Wavelet Decomposition**: Stores detail coefficients at multiple levels. Level $k=0$ is the highest resolution (original data), and higher $k$ values represent lower resolutions [S1, unofficial].
2.  **Laplacian Pyramid**: Stores a base approximation and difference details at each level [S1, unofficial].

### Algorithm Steps

1.  **Ray Generation**: For each pixel, generate a ray $r(t) = \mathbf{o} + t\mathbf{d}$ [S1, unofficial]. Compute the intersection interval $[t_{start}, t_{end}]$ with the volume bounding box [S1, unofficial].
2.  **Initialization**: Initialize accumulated color $C_{in} = 0$ and accumulated opacity $\alpha_{in} = 0$ [S1, S15, S46, unofficial]. Set current level $m$ to the highest resolution (e.g., $m=0$) or a starting level [S1, unofficial].
3.  **Ray Tracing Loop**: Iterate along the ray from $t_{start}$ to $t_{end}$:
    *   **Adaptive Step Selection**: Determine the step size $\Delta t$ based on the current resolution level $m$. Lower levels (higher $m$) allow for larger step sizes [S1, unofficial].
    *   **Sample Point**: Calculate sample position $\mathbf{p} = r(t)$ [S1, unofficial].
    *   **Value Reconstruction**: Retrieve the scalar value $V(\mathbf{p})$ from the multiresolution data at level $m$.
        *   *Wavelet Reconstruction*: For a 1D case with kernel size $q$, $v(i) = \sum_{l=0}^{q-1} c_{u,w}(l) \phi(i-l)$, extended to 3D by summing over kernel dimensions [S1, unofficial].
        *   *Laplacian Pyramid Reconstruction*: $c_i^k = c_{i}^{k+1} + \sum_{n} w(n - 2k) c_{i}^{k+1}$, where $k$ is the level [S1, unofficial].
    *   **Classification**: Map $V(\mathbf{p})$ to color $C_{sample}$ and opacity $\alpha_{sample}$ using the transfer function. Correct opacity for step size: $\alpha_{sample} = 1 - e^{-\sigma \Delta t}$, where $\sigma$ is derived from $V(\mathbf{p})$ [S24, S30, unofficial].
    *   **Compositing**: Update accumulated values:
        $$ C_{in} = C_{in} + (1 - \alpha_{in}) \cdot C_{sample} \cdot \alpha_{sample} $$
        $$ \alpha_{in} = \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha_{sample} $$
    *   **Occlusion Check**: Calculate the remaining transmission (occlusion) $W(t) = 1 - \alpha_{in}$ [S1, unofficial].
    *   **Early Termination**: If $W(t) < \epsilon$ (a small positive threshold), terminate the ray [S1, unofficial].
    *   **Level Switching**: Optionally, switch resolution levels based on occlusion or local detail requirements. Using lower resolution data ($m > 0$) reduces cost but may introduce error [S1, unofficial].
4.  **Output**: Write $C_{in}$ and $\alpha_{in}$ to the pixel buffer.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiresolution_volume_rendering(volume_data, camera, transfer_function, epsilon=1e-4):
    """
    volume_data: Object providing access to multiresolution levels (e.g., wavelet/pyramid).
                 Must support reconstruction of scalar value at point p given level m.
    camera: Object with method get_ray(pixel) returning (origin, direction).
    transfer_function: Function mapping scalar value to (color, extinction_coefficient).
    epsilon: Threshold for early termination.
    """
    image_width, image_height = camera.resolution
    output_image = zeros((image_height, image_width, 3))
    output_alpha = zeros((image_height, image_width))

    for y in range(image_height):
        for x in range(image_width):
            origin, direction = camera.get_ray(x, y)
            
            # 1. Ray-Volume Intersection
            t_start, t_end = intersect_volume(origin, direction, volume_data.bounding_box)
            if t_start > t_end:
                continue

            # 2. Initialization
            accumulated_color = [0.0, 0.0, 0.0]
            accumulated_opacity = 0.0
            t = t_start
            
            # Determine initial step size and level based on resolution requirements
            # Not given in sources: Specific logic for choosing initial m and delta_t
            # Assume default to highest resolution (m=0) and base step size
            current_level = 0
            base_step = volume_data.get_base_step_size()
            step_size = base_step 

            # 3. Ray Tracing Loop
            while t < t_end:
                # Calculate sample position
                sample_point = origin + t * direction
                
                # 4. Value Reconstruction
                # Retrieve scalar value V from multiresolution data at current_level
                scalar_value = volume_data.reconstruct(sample_point, current_level)
                
                # 5. Classification
                color_sample, extinction_coeff = transfer_function(scalar_value)
                
                # Correct opacity for step size (Beer-Lambert)
                alpha_sample = 1.0 - exp(-extinction_coeff * step_size)
                
                # 6. Compositing (Front-to-Back)
                # C_out = C_in + (1 - alpha_in) * C_sample * alpha_sample
                # alpha_out = alpha_in + (1 - alpha_in) * alpha_sample
                weight = (1.0 - accumulated_opacity) * alpha_sample
                accumulated_color = [
                    accumulated_color[i] + weight * color_sample[i] for i in range(3)
                ]
                accumulated_opacity += weight
                
                # 7. Early Termination
                # W(t) is the remaining transmission (1 - accumulated_opacity)
                if (1.0 - accumulated_opacity) < epsilon:
                    break
                
                # 8. Advance Ray
                # Not given in sources: Logic for adaptive step size or level switching
                # Standard approach: step_size might increase with lower resolution levels
                t += step_size

            output_image[y, x] = accumulated_color
            output_alpha[y, x] = accumulated_opacity

    return output_image, output_alpha
```

### Parameters and Edge Cases
*   **Epsilon ($\epsilon$)**: Threshold for early termination. Typical values are small positive numbers (e.g., $10^{-4}$) [S1, unofficial].
*   **Resolution Levels**: The number of levels $k$ affects accuracy and performance. Experiments used 5 levels [S1, unofficial].
*   **Mixing Levels**: Combining shading from one level and occlusion from another can introduce errors. Table 4.2 in [S1, unofficial] shows errors ranging from -0.027605 to 0.031589 depending on the combination of levels used for shading ($C_m$) and occlusion ($W_j$).
*   **Precision**: Using lower resolution data reduces computational cost but affects accuracy [S1, unofficial].

## Subtopics

### Multiresolution Rendering with Data Reduction
A variant that exploits data reduction across resolution levels to achieve super-linear performance increases by computing fewer voxels and increasing step lengths for low-resolution blocks [S2, unofficial].
*   **Data Reduction Ratio**: Compression factors applied to volume blocks. Typical values: 8.5:1, 15.7:1, 31.4:1, 92.8:1 [S2, unofficial].
*   **Performance**: Achieves super-linear speedups on GPU-based ray casting [S2, unofficial].
*   **Accuracy**: Mean error increases with higher data reduction: 0.6 at 8.5:1, 1.2 at 15.7:1, 1.6 at 31.4:1 [S2, unofficial].
*   **Sensitivity**: Shadow Volume sensitivity increases at lower resolutions (32³, 64³, 128³, 256³), requiring careful segment length and reduction ratio selection [S2, unofficial].
*   **Usage**: Used in interactive medical visualization and virtual autopsy applications [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Reduced compared to full-resolution ray casting due to adaptive integration and early termination. The complexity depends on the number of samples taken, which varies by ray and resolution level [S1, unofficial].
*   **Memory Complexity**: Requires storage for the multiresolution decomposition (wavelet coefficients or pyramid levels), which is typically more compact than the original volume data [S1, unofficial].
*   **Performance**: Experiments show that using lower resolution data for shading or occlusion reduces computational cost because a larger step size can be used [S1, unofficial]. Super-linear performance increases are reported with data reduction [S2, unofficial].
*   **Strengths**: Efficient rendering of large datasets; adaptive detail; early termination saves computation in opaque regions [S1, S2, unofficial].
*   **Limitations**: Accuracy degrades with lower resolution levels and higher data reduction ratios [S1, S2, unofficial]. Mixing levels can introduce visible artifacts [S1, unofficial].
*   **When to Use**: Interactive visualization of large volumes where real-time performance is critical, and some loss of detail is acceptable [S1, S2, unofficial].
*   **When Not to Use**: Applications requiring high-fidelity detail throughout the volume, or where shadow volume accuracy is critical at low resolutions [S2, unofficial].

## Background
The method originates from research into efficient volume rendering using multiresolution decompositions. Key work includes "Direct rendering of multiresolution decomposed volume data" which analyzes integration in the multiresolution domain and proposes adaptive integration and early termination [S1, unofficial]. Another significant contribution is "GPU-Based Volume Ray-Casting with Advanced Illumination" which demonstrates multiresolution rendering with data reduction on GPUs, achieving interactive frame rates for medical visualization [S2, unofficial]. The technique builds on the fundamental loop of direct volume ray casting [G] and extends it with hierarchical data structures.

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
