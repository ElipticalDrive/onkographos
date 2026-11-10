# Dye-Advection LIC
Also called: 3D Dye-Advection LIC, Dye Advection, Dye Injection with Backward LIC, Dye Smearing

## Overview
Dye-Advection LIC is a 3D extension of Line Integral Convolution that simulates the advection of colored dye within a vector field by exploiting the "smearing" nature of the LIC algorithm [S1, S2, unofficial]. It solves the problem of visualizing local flow features, such as wavefronts or specific material paths, which are difficult to observe in standard LIC due to the lack of directional correspondence between neighboring streamlines [S2, unofficial]. Unlike standard LIC, which produces symmetric textures, this method allows users to dynamically inject dye at arbitrary locations, ensuring the color propagates only in the forward flow direction [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D vector field $\mathbf{V}$; a base white noise texture $N(x,y,z)$; a dye injection mask $M(x,y,z)$ defining the spatial location and color of the injected dye; convolution length $L$.
*   **Output:** A final rendered image $Final_{out}(x,y)$ combining the base flow structure with the advected dye.

**Data Structures:**
*   **White Noise Texture:** A 3D volume of random values, typically isotropic Gaussian or uniform noise, sampled along streamlines.
*   **Dye Mask:** A 3D volume where non-zero values indicate the presence of dye. The value represents the color/intensity of the dye at that location.
*   **Streamline Data:** Precomputed or on-the-fly streamline paths for each voxel, defined by their start points and integration directions.

**Steps:**
1.  **Compute Base LIC:** Compute a standard LIC image $F_{out}(x,y)$ for the current view. This is done by convolving the white noise texture $N$ along both positive and negative streamline directions for all cells. This establishes the background flow texture [S2, unofficial].
2.  **Identify Affected Cells:** Determine the set of cells $C_{affected}$ that will receive color contributions from the dye. A cell $c$ is in $C_{affected}$ if its backward streamline (negative direction) passes through a region where the dye mask $M$ is non-zero, within the convolution length $L$ [S2, unofficial].
    *   *Note:* Efficient identification of $C_{affected}$ is critical for performance. See [Fast Searching Algorithm for Dye Advection](12136-fast-searching-algorithm-for-dye-advection.md) for an optimized approach using a sparse directed graph of backward neighbors [S1, unofficial].
3.  **Compute Dye LIC:** For each cell $c \in C_{affected}$, compute a dye-specific LIC value $D_{out}(c)$.
    *   Sample the dye mask $M$ along the **backward** streamline direction from cell $c$ for length $L$.
    *   Convolve the sampled values using the standard LIC filter kernel.
    *   Cells not in $C_{affected}$ are assigned $D_{out}(c) = 0$ (or transparent) [S2, unofficial].
4.  **Composite:** Combine the base LIC and the dye LIC. The final output is computed as:
    $$Final_{out}(x,y) = D_{out}(x,y) \circledcirc F_{out}(x,y)$$
    where the operator $\circledcirc$ overwrites the pixel values of $F_{out}$ with $D_{out}$ where $D_{out}$ is non-zero [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dye_advection_lic(vector_field, white_noise, dye_mask, convolution_length, view_params):
    """
    Computes Dye-Advection LIC for a 3D vector field.
    
    Args:
        vector_field: 3D array of vectors V(x,y,z).
        white_noise: 3D array of noise values N(x,y,z).
        dye_mask: 3D array of dye colors M(x,y,z).
        convolution_length: L, the integration length for LIC.
        view_params: Camera and projection parameters.
        
    Returns:
        final_image: 2D image with base LIC and advected dye.
    """
    # Step 1: Compute standard LIC for the base texture
    # Convolve white noise along both forward and backward streamlines
    base_lic = compute_standard_lic(vector_field, white_noise, convolution_length, view_params)
    
    # Step 2: Identify cells affected by dye injection
    # A cell is affected if its backward streamline intersects the dye mask
    affected_cells = find_affected_cells(vector_field, dye_mask, convolution_length)
    
    # Step 3: Compute dye-specific LIC for affected cells only
    # Convolve dye mask along BACKWARD streamlines for affected cells
    dye_lic = compute_dye_lic(vector_field, dye_mask, affected_cells, convolution_length)
    
    # Step 4: Composite results
    # Overwrite base LIC pixels with dye LIC pixels where dye is present
    final_image = composite_lic(base_lic, dye_lic)
    
    return final_image

def compute_standard_lic(vector_field, noise, L, view_params):
    """
    Computes standard LIC by convolving noise along streamlines in both directions.
    """
    image = zeros_like(view_params.screen)
    for pixel in view_params.screen_pixels:
        ray = trace_ray(pixel, vector_field)
        # Integrate forward and backward
        samples_forward = integrate_along_streamline(ray, noise, L, direction=1)
        samples_backward = integrate_along_streamline(ray, noise, L, direction=-1)
        # Combine samples (standard LIC usually averages or sums)
        all_samples = samples_forward + samples_backward
        # Apply LIC filter kernel
        image[pixel] = convolve(all_samples, LIC_KERNEL)
    return image

def find_affected_cells(vector_field, dye_mask, L):
    """
    Identifies cells whose backward streamlines pass through the dye mask.
    Uses efficient search if available, otherwise brute force.
    """
    affected = set()
    # Optimization: Use Fast Searching Algorithm if implemented
    # See: Fast Searching Algorithm for Dye Advection
    # For simplicity, we iterate over all cells and check backward path
    for cell in all_voxels(vector_field):
        ray = trace_ray_from_cell(cell, vector_field, direction=-1)
        if intersects_dye_mask(ray, dye_mask, L):
            affected.add(cell)
    return affected

def compute_dye_lic(vector_field, dye_mask, affected_cells, L):
    """
    Computes LIC using the dye mask along backward streamlines for affected cells.
    """
    dye_image = zeros_like(view_params.screen)
    for cell in affected_cells:
        ray = trace_ray_from_cell(cell, vector_field, direction=-1)
        # Sample dye mask along backward path
        samples = integrate_along_streamline(ray, dye_mask, L, direction=-1)
        # Apply LIC filter
        value = convolve(samples, LIC_KERNEL)
        # Map cell to screen pixel (simplified)
        pixel = project_cell_to_screen(cell)
        if pixel is valid:
            dye_image[pixel] = value
    return dye_image

def composite_lic(base_lic, dye_lic):
    """
    Overwrites base LIC with dye LIC where dye is non-zero.
    """
    final = base_lic.copy()
    mask = dye_lic > 0
    final[mask] = dye_lic[mask]
    return final
```

**Parameters:**
*   **Convolution Length ($L$):** Determines the extent of the smearing effect. Typical values depend on the resolution and flow velocity, often set to a fraction of the domain size or based on the local streamline length [S2, unofficial].
*   **Dye Mask ($M$):** Defines the spatial extent and color of the injected dye. Can be a single point, a plane, or a complex volume [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Upstream Coloring:** If standard LIC (convolving in both directions) is applied directly to the dyed texture, the upstream of the dyed area will be incorrectly colored because positive streamlines in that area pass through the dyed area [S2, unofficial]. The solution is to convolve only in the backward direction for affected cells [S2, unofficial].
*   **Precision:** Integration errors in backward streamline tracing can lead to inaccurate dye placement, especially in high-velocity or turbulent regions [S2, unofficial].
*   **Performance:** Naive identification of affected cells by scanning the entire volume is computationally expensive. Efficient algorithms are required for interactive use [S1, unofficial].

## Subtopics

### [Fast Searching Algorithm for Dye Advection](12136-fast-searching-algorithm-for-dye-advection.md)
This algorithm efficiently identifies the set of grid cells affected by a new dye injection in Dye-Advection LIC without scanning the entire volume or storing exhaustive backward streamline histories. It differs from brute-force neighbor lists by constructing a sparse directed graph of "direct flow-back neighbors" and using Breadth First Search (BFS) to locate the affected region, significantly reducing memory overhead while maintaining visual accuracy.

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the streamline integration and convolution. For each frame, standard LIC is computed for all pixels ($O(N)$ where $N$ is the number of pixels). The dye advection step requires identifying affected cells and computing LIC only for them. With the fast searching algorithm, the number of affected cells is proportional to the dye volume and flow velocity, making it significantly faster than recomputing LIC for the entire volume [S1, unofficial].
*   **Memory Complexity:** Requires storage for the vector field, noise texture, dye mask, and streamline data. The fast searching algorithm reduces memory overhead by avoiding exhaustive neighbor lists [S1, unofficial].
*   **Strengths:** Allows dynamic, interactive injection of dye to highlight specific flow features; preserves the visual quality of standard LIC; provides clear directional information for local flow structures [S1, S2, unofficial].
*   **Limitations:** Computationally more intensive than standard LIC due to the need for two convolution passes (base and dye) and the search for affected cells; sensitive to integration errors in backward tracing [S2, unofficial].
*   **When to use:** When visualizing local flow features, wavefronts, or material advection in 3D vector fields is required, and interactivity is desired [S1, S2, unofficial].
*   **When not to use:** When performance is critical and dynamic dye injection is not needed; standard LIC or particle tracing may be more appropriate [S1, unofficial].

## Background
Dye-Advection LIC was proposed by Shen et al. as a 3D extension of the Line Integral Convolution method [S1, unofficial]. The method leverages the "smearing" nature of LIC to simulate the advection of dye within a flow field, allowing for the visualization of both regular LIC textures and 3D particle traces [S1, unofficial]. The technique was introduced to address the difficulty of observing correspondence between neighboring streamlines in standard LIC, which is crucial for tracking local flow features [S2, unofficial]. The method was presented in the context of flow visualization techniques for CFD using volume rendering [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19970011183: Visualizing Vector Fields Using Line Integral Convolution and Dye Advection, https://archive.org/details/NASA_NTRS_Archive_19970011183, fetched 2026-10-26 (unofficial)
