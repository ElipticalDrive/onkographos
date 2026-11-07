# Multiresolution Volume Rendering

Also called: Direct rendering of multiresolution decomposed volume data, Multiresolution Volume Rendering with Adaptive Integration, Wavelet/Laplacian Pyramid Volume Rendering

## Overview
Multiresolution Volume Rendering is a direct volume ray casting technique that utilizes a hierarchical decomposition of volume data (such as wavelets or Laplacian pyramids) to perform adaptive integration and early termination [S1, unofficial]. By allowing the renderer to sample at lower resolutions where accuracy requirements are relaxed or occlusion is high, it reduces computational cost compared to fixed-resolution ray casting [S1, unofficial]. It sets itself apart from standard direct ray casting by dynamically selecting the resolution level for shading and occlusion calculations along the ray path, trading off precision for speed based on the local contribution to the final pixel color [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset decomposed into a multiresolution hierarchy (e.g., $k$ levels, where level 0 is the highest resolution and level $k-1$ is the lowest) [S1, unofficial]. A camera viewpoint and projection parameters [S1, unofficial]. A transfer function mapping scalar values to color and opacity [S1, unofficial].
*   **Output:** A 2D image where each pixel color is the result of ray integration through the volume [S1, unofficial].

**Data Structures:**
The volume data is stored as a set of coefficient arrays corresponding to the multiresolution decomposition. For a wavelet decomposition, these are approximation and detail coefficients at each level. For a Laplacian pyramid, these are the difference images between levels [S1, unofficial]. The exact layout depends on the chosen decomposition (e.g., 3D wavelet coefficients or pyramid layers) [S1, unofficial].

**Numbered Steps:**

1.  **Ray Generation:** For each pixel in the image plane, generate a ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the viewpoint and $\mathbf{d}$ is the direction vector [S1, unofficial].
2.  **Intersection:** Compute the entry point $t_1$ and exit point $t_2$ of the ray with the volume's bounding box [S1, unofficial].
3.  **Initialization:** Initialize the accumulated color $C_{acc} = 0$ and accumulated opacity $\alpha_{acc} = 0$ (or transmittance $W(t) = 1$) [S1, unofficial].
4.  **Ray Tracing Loop:** Iterate along the ray from $t_1$ to $t_2$ with a step size $\Delta t$. The step size and resolution level may vary based on the current occlusion and desired accuracy [S1, unofficial].
    *   **Sample Value:** At the current position $t$, retrieve the scalar value $d(t)$ from the multiresolution data structure. This involves reconstructing the value from the coefficients of the selected level $m$ [S1, unofficial].
    *   **Classification:** Map $d(t)$ to local color $C(t)$ and opacity $\alpha(t)$ using the transfer function [S1, unofficial].
    *   **Shading:** Compute the shading value $C(t)$ based on the local color and illumination model [S1, unofficial].
    *   **Occlusion Update:** Update the occlusion (transmittance) $W(t)$. In the multiresolution domain, $W(t)$ can be computed using coefficients from a different level $j$ than the shading level $m$ [S1, unofficial].
    *   **Compositing:** Update the accumulated color:
        $$ C_{acc}^{new} = C_{acc}^{old} + W(t) \cdot C(t) \cdot \Delta t $$
        $$ W(t+\Delta t) = W(t) \cdot (1 - \alpha(t) \cdot \Delta t) $$
        (Note: The exact integration formula is an inner product of shading and occlusion, approximated by summation [S1, unofficial]).
    *   **Early Termination:** If the current occlusion $W(t)$ falls below a predefined threshold $\epsilon$ (e.g., $W(t) < \epsilon$), terminate the ray tracing loop early [S1, unofficial]. This is valid because $W(t)$ approaches zero as $t$ increases, meaning subsequent samples contribute negligibly to the final color [S1, unofficial].
5.  **Final Color:** The final pixel color is $C_{acc}$ [S1, unofficial].

**Formulas:**
*   **Wavelet Reconstruction (1D):** To reconstruct a value $v(i)$ at point $i$ using kernel size $q$ and coefficients $c_{u,w}(l)$:
    $$ v(i) = \sum_{l=0}^{q-1} c_{u,w}(l) \phi(i-l) $$
    This is extended to 3D by summing over the kernel dimensions [S1, unofficial].
*   **Laplacian Pyramid Reconstruction:** Calculated from coefficients $c_i^k$ at level $k$:
    $$ c_i^k = c_{i}^{k+1} + \sum_{n} w(n - 2k) c_{i}^{k+1} $$
    where $w$ is the reconstruction kernel [S1, unofficial].
*   **Integration in Multiresolution Domain:** The integration using shading color $C_m(t)$ from level $m$ and occlusion $W_j(t)$ from level $j$ is denoted as $I_{m,j}$:
    $$ I_{m,j} = \langle C_m(t), W_j(t) \rangle $$
    Level 0 corresponds to the highest resolution. Using $m > 0$ or $j > 0$ implies using lower resolution approximations [S1, unofficial].

**Parameters:**
*   **Threshold $\epsilon$:** A small positive value for early termination. Not given in the sources.
*   **Number of Levels $k$:** The depth of the multiresolution decomposition. In experiments, $k=5$ was used [S1, unofficial].
*   **Step Size $\Delta t$:** Can be larger for lower resolution levels, reducing computational cost [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision Loss:** Using lower resolution data for shading or occlusion reduces accuracy. Mixing levels (e.g., high-res shading with low-res occlusion) can introduce errors. Table 4.2 in [S1, unofficial] shows errors ranging from -0.027605 to 0.031589 depending on the combination of levels [S1, unofficial].
*   **Occlusion Handling:** If early termination is not implemented, non-zero shading values at large $t$ (where occlusion is near zero) waste computation [S1, unofficial].
*   **Boundary Conditions:** The reconstruction kernels must handle boundaries correctly to avoid artifacts at the volume edges [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiresolution_volume_rendering(volume_data, camera, transfer_function, epsilon=1e-4):
    """
    volume_data: A structure containing multiresolution coefficients (e.g., wavelet or pyramid levels).
    camera: Contains viewpoint o, direction d, and image plane parameters.
    transfer_function: Maps scalar value to (color, opacity).
    epsilon: Threshold for early termination.
    """
    image = empty_image(camera.width, camera.height)
    
    for y in range(camera.height):
        for x in range(camera.width):
            # 1. Ray Generation
            ray_origin = camera.get_origin(x, y)
            ray_direction = camera.get_direction(x, y)
            
            # 2. Intersection
            t_start, t_end = intersect_volume(ray_origin, ray_direction, volume_data.bounding_box)
            
            if t_start is None:
                continue
                
            # 3. Initialization
            accumulated_color = vec3(0, 0, 0)
            transmittance = 1.0
            
            # 4. Ray Tracing Loop
            t = t_start
            step_size = get_initial_step_size() # [G] Initial step size, typically based on highest resolution
            
            while t < t_end:
                # Determine resolution level based on current transmittance and position
                # [G] Logic for selecting level m (shading) and j (occlusion) is not fully specified in sources.
                # A simple heuristic might be to use lower levels when transmittance is low.
                level_shading = get_level_for_shading(t, transmittance, volume_data.levels)
                level_occlusion = get_level_for_occlusion(t, transmittance, volume_data.levels)
                
                # 5. Sample Value
                # Reconstruct scalar value from multiresolution data
                scalar_value = reconstruct_value(volume_data, level_shading, t)
                
                # 6. Classification
                color, opacity = transfer_function(scalar_value)
                
                # 7. Shading
                # [G] Illumination model details are not provided. Assuming color is already shaded or simple diffuse.
                shading_color = color 
                
                # 8. Occlusion Update
                # Compute transmittance contribution using level j
                # [G] The exact formula for updating transmittance from level j coefficients is not explicitly given.
                # Standard front-to-back compositing uses opacity.
                # W(t + dt) = W(t) * (1 - alpha * dt)
                # Here, alpha might be derived from level j.
                alpha_from_occlusion_level = get_opacity_from_level(volume_data, level_occlusion, t)
                transmittance *= (1.0 - alpha_from_occlusion_level * step_size)
                
                # 9. Compositing
                accumulated_color += transmittance * shading_color * step_size
                
                # 10. Early Termination
                if transmittance < epsilon:
                    break
                
                # 11. Advance
                t += step_size
                
            image.set_pixel(x, y, accumulated_color)
            
    return image

def reconstruct_value(volume_data, level, t):
    """
    Reconstructs a scalar value at position t using coefficients from the specified level.
    For wavelets: sum of coefficients * basis functions.
    For Laplacian pyramid: sum of pyramid layers.
    """
    # [G] Specific reconstruction kernel implementation depends on the decomposition type.
    # This is a placeholder for the actual reconstruction logic.
    pass

def get_level_for_shading(t, transmittance, levels):
    """
    Heuristic to select shading resolution level.
    """
    # [G] Not specified in sources.
    return levels[0]

def get_level_for_occlusion(t, transmittance, levels):
    """
    Heuristic to select occlusion resolution level.
    """
    # [G] Not specified in sources.
    return levels[0]

def get_opacity_from_level(volume_data, level, t):
    """
    Retrieves opacity value for a sample point from a specific resolution level.
    """
    # [G] Not specified in sources.
    pass

def get_initial_step_size():
    """
    Returns the initial step size for ray tracing.
    """
    # [G] Not specified in sources.
    return 1.0
```

## Subtopics

### Wavelet Volume Rendering
Uses wavelet decomposition for multiresolution representation. It allows for efficient compression and adaptive sampling. Choose when data has significant spatial frequency content that can be captured by wavelets. [Link to Wavelet Volume Rendering](link)

### Laplacian Pyramid Volume Rendering
Uses Laplacian pyramids for multiresolution representation. Similar to wavelets but may have different reconstruction properties. Choose when pyramid-based processing is preferred. [Link to Laplacian Pyramid Volume Rendering](link)

## Operating characteristics
*   **Time Complexity:** Reduced compared to fixed-resolution ray casting due to adaptive integration and early termination. The complexity depends on the number of levels and the effectiveness of early termination [S1, unofficial].
*   **Memory Complexity:** Depends on the multiresolution decomposition. Wavelet and pyramid representations can be more compact than the original volume data, but storing multiple levels may increase memory usage if not compressed [S1, unofficial].
*   **Measured Performance:** In experiments with a CT data set decomposed into 5 levels, using lower resolution data for shading or occlusion reduced computational cost [S1, unofficial]. However, accuracy was affected, with errors observed when mixing levels (e.g., error of -0.027605 for $C_2/W_2$ vs 0.011057 for $C_0/W_0$) [S1, unofficial].
*   **Strengths:** Reduced computational cost, adaptive integration, early termination.
*   **Limitations:** Potential loss of accuracy, complexity of implementation, sensitivity to level mixing.
*   **When to Use:** Large volume datasets where performance is critical and some loss of accuracy is acceptable.
*   **When Not to Use:** When high precision is required, or when the volume data does not decompose well into multiresolution hierarchies.

## Background
*   **Origin:** The method was proposed in the context of direct rendering of multiresolution decomposed volume data [S1, unofficial].
*   **History:** The technique leverages the hierarchical structure of multiresolution decompositions (wavelets or Laplacian pyramids) to facilitate adaptive integration and early termination [S1, unofficial].
*   **Usage:** Used in volume rendering applications where performance optimization is needed, such as medical imaging (CT data sets) [S1, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
