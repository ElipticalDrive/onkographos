# Multiresolution Volume Rendering

Also called: Direct rendering of multiresolution decomposed volume data, Multiresolution Volume Rendering with Adaptive Integration, Wavelet/Laplacian Pyramid Volume Rendering

## Overview
Multiresolution Volume Rendering is a direct volume ray casting technique that utilizes a hierarchical decomposition of volume data (such as wavelets or Laplacian pyramids) to perform adaptive integration and early termination [S1, unofficial]. By allowing the renderer to switch between resolution levels along a ray, it reduces computational cost by using coarser data for occluded or low-detail regions, while maintaining high fidelity where necessary [S1, unofficial]. It differs from standard direct volume rendering by explicitly managing data resolution as a variable in the integration loop, rather than using a fixed-resolution voxel grid [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Multiresolution volume data structure containing $k+1$ levels ($m=0, \dots, k$), where level 0 is the highest resolution and level $k$ is the lowest [S1, unofficial].
*   Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$) for ray generation.
*   Transfer function mapping scalar values to color $C$ and opacity $\alpha$ [S1, unofficial].
*   Early termination threshold $\epsilon$ for occlusion [S1, unofficial].

**Outputs:**
*   Final pixel color $I$ computed via integration along the ray [S1, unofficial].

### Data Structures
The volume data is stored in a multiresolution hierarchy. Each level $m$ contains coefficients $c_{u,w}$ (or similar notation depending on the decomposition type) representing the data at that resolution [S1, unofficial].
*   **Wavelet Domain:** Coefficients $c_{u,w}$ at level $m$.
*   **Laplacian Pyramid:** Coefficients $c_i^k$ at level $k$, where reconstruction combines details from level $k+1$ [S1, unofficial].

### Steps

1.  **Ray Generation:** For each pixel, define a ray $r(t) = \mathbf{o} + t\mathbf{d}$ [S1, unofficial].
2.  **Boundary Intersection:** Compute the entry parameter $t_1$ and exit parameter $t_2$ where the ray intersects the volume boundary [S1, unofficial].
3.  **Initialization:** Initialize accumulated color $C_{acc} = 0$ and accumulated opacity $\alpha_{acc} = 0$ (transmittance $W(t) = 1$) [S1, S4].
4.  **Adaptive Integration Loop:** Iterate from $t_1$ to $t_2$ with a step size $\Delta t$ [S1, unofficial].
    *   **Level Selection:** Select the resolution level $m$ for the current segment. Lower levels (higher $m$) allow larger step sizes [S1, unofficial].
    *   **Sampling:** Sample the scalar value at the current position $t$ using the selected level's reconstruction function [S1, unofficial].
    *   **Classification:** Map the scalar value to local color $C(t)$ and opacity $\alpha(t)$ using the transfer function [S1, unofficial].
    *   **Compositing:** Update accumulated color and opacity using the front-to-back over operator:
        $$ C_{acc} = C_{acc} + (1 - \alpha_{acc}) \alpha(t) C(t) $$
        $$ \alpha_{acc} = \alpha_{acc} + (1 - \alpha_{acc}) \alpha(t) $$
    *   **Occlusion Check:** Calculate the remaining transmittance $W(t) = 1 - \alpha_{acc}$ [S1, unofficial].
    *   **Early Termination:** If $W(t) < \epsilon$, terminate the ray [S1, unofficial].

### Reconstruction Formulas
The scalar value at a point is reconstructed from the multiresolution coefficients.

**Wavelet Reconstruction (1D):**
$$ v(i) = \sum_{l=0}^{q-1} c_{u,w}(l) \phi(i-l) $$
where $q$ is the kernel size and $\phi$ is the scaling function [S1, unofficial]. This extends to 3D by summing over kernel dimensions [S1, unofficial].

**Laplacian Pyramid Reconstruction:**
$$ c_i^k = c_{i}^{k+1} + \sum_{n} w(n - 2k) c_{i}^{k+1} $$
where $k$ is the level and $w$ is the kernel [S1, unofficial].

### Parameters
*   **$\epsilon$ (Early Termination Threshold):** A small positive value (e.g., $10^{-3}$ or $10^{-4}$) below which transmittance is considered zero [S1, unofficial].
*   **Step Size $\Delta t$:** Determined by the resolution level. Higher levels (coarser data) use larger $\Delta t$ [S1, unofficial].
*   **Number of Levels $k$:** In experiments, $k=4$ (5 levels) was used for CT data [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision Loss:** Mixing shading colors from high-resolution levels with occlusion from low-resolution levels can introduce errors. Table 4.2 in [S1, unofficial] shows errors such as $-0.027605$ when combining $C_2$ and $W_2$, compared to $0.011057$ for $C_0$ and $W_0$.
*   **Occlusion Handling:** If early termination is not implemented, non-zero shading values at large $t$ (where transmittance is near zero) waste computation without contributing to the final image [S1, unofficial].
*   **Artifact Generation:** Using too coarse a resolution for shading can cause visible degradation in image quality [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiresolution_volume_rendering(volume_data, camera, transfer_function, epsilon=1e-4):
    """
    Renders a pixel using multiresolution volume rendering.
    
    Args:
        volume_data: Object containing levels 0..k of multiresolution data.
        camera: Object with origin 'o' and direction 'd'.
        transfer_function: Function mapping scalar value to (color, opacity).
        epsilon: Threshold for early termination.
        
    Returns:
        Final color (R, G, B).
    """
    # 1. Ray Generation
    origin = camera.o
    direction = camera.d
    
    # 2. Boundary Intersection
    # Compute t1 (entry) and t2 (exit) for the ray against the volume AABB
    t_start, t_end = intersect_aabb(origin, direction, volume_data.bbox)
    
    # 3. Initialization
    acc_color = [0.0, 0.0, 0.0]
    acc_opacity = 0.0
    
    # 4. Adaptive Integration Loop
    t = t_start
    while t < t_end:
        # Determine resolution level based on position or adaptive criteria
        # Lower levels (higher index) allow larger steps
        level = get_resolution_level(t, volume_data)
        
        # Sample scalar value using reconstruction from the selected level
        scalar_val = reconstruct_scalar(volume_data, level, t)
        
        # Classification
        color, opacity = transfer_function(scalar_val)
        
        # Compositing (Front-to-Back)
        # Transmittance at this step
        transmittance = 1.0 - acc_opacity
        
        # Update accumulated color
        acc_color = [
            acc_color[0] + transmittance * opacity * color[0],
            acc_color[1] + transmittance * opacity * color[1],
            acc_color[2] + transmittance * opacity * color[2]
        ]
        
        # Update accumulated opacity
        acc_opacity = acc_opacity + transmittance * opacity
        
        # Early Termination Check
        if (1.0 - acc_opacity) < epsilon:
            break
            
        # Step size depends on resolution level
        step_size = get_step_size(level)
        t += step_size
        
    return acc_color
```

## Subtopics

### Wavelet Decomposition
*   **What it is:** A method of decomposing volume data into frequency bands using wavelet transforms, creating a hierarchy of detail coefficients [S1, unofficial].
*   **Distinction:** Uses orthogonal or biorthogonal wavelets to separate approximation and detail components [S1, unofficial].
*   **When to choose:** When the data has sparse details or when compression is a secondary benefit [S1, unofficial].
*   **Link:** [Wavelet Decomposition](link_to_wavelet_article)

### Laplacian Pyramid
*   **What it is:** A multi-scale representation where each level contains the difference between a smoothed version of the previous level and the original [S1, unofficial].
*   **Distinction:** Simpler reconstruction than wavelets but may lack perfect reconstruction properties depending on the kernel [S1, unofficial].
*   **When to choose:** For simpler implementation or when the data is already in a pyramid format [S1, unofficial].
*   **Link:** [Laplacian Pyramid](link_to_laplacian_article)

## Operating characteristics

*   **Time Complexity:** Reduced compared to fixed-resolution rendering due to adaptive step sizes and early termination. The cost depends on the number of levels and the distribution of opacity along the ray [S1, unofficial].
*   **Memory Complexity:** Higher than fixed-resolution rendering due to the storage of multiple resolution levels [S1, unofficial].
*   **Measured Performance:** In experiments with a CT dataset decomposed into 5 levels, using lower resolution data for shading/occlusion reduced computational cost [S1, unofficial].
*   **Strengths:** Efficient rendering of large volumes; adaptive detail; early termination optimization [S1, unofficial].
*   **Limitations:** Potential for visual artifacts when mixing resolution levels; increased memory usage; complexity of implementation [S1, unofficial].
*   **When to use:** Large datasets where performance is critical and some loss of fidelity in occluded regions is acceptable [S1, unofficial].
*   **When not to use:** When high-fidelity detail is required everywhere, or memory is severely constrained [S1, unofficial].

## Background

*   **Origin:** The method is described in the thesis "Direct rendering of multiresolution decomposed volume data" [S1, unofficial].
*   **History:** It builds on the principles of direct volume ray casting and multiresolution analysis (wavelets/pyramids) [S1, unofficial].
*   **Usage:** Applied in medical imaging (e.g., CT data) and scientific visualization [S1, unofficial].

## Sources

[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
