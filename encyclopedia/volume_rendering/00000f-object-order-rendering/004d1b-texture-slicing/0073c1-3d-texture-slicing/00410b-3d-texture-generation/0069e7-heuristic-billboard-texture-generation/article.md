# Heuristic Billboard Texture Generation

Also called: Billboard Texture Generation, Heuristic Texel Selection for Billboards

## Overview
Heuristic Billboard Texture Generation is a method for creating view-dependent textures for volumetric cells (billboards) by selecting specific texels from input images based on a heuristic that compares cell and pixel transparencies [S1]. It solves the problem of blurring associated with simple projection methods by ensuring that only the most relevant image data contributes to the final texture, while maintaining visual consistency across the volume [S1]. Unlike sibling methods that may use isotropic synthesis or model-based approaches, this technique relies on explicit transparency metrics and a stencil-buffer-based tracking system to determine texel validity [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of RGBA images (RGB and alpha channels), a 3D tree structure of volumetric cells, and camera parameters for each image.
*   **Outputs:** An $M \times M$ RGBA texture for each cell in the tree for each input image.

**Data Structures**
*   **Image Copy ($C_i$):** A copy of the original RGBA image $i$. Each pixel stores:
    *   RGB values.
    *   Alpha value (from the original alpha image).
    *   `age`: An integer field, initially 0, used to prioritize texel selection over time [S1].
*   **Stencil Buffer State:**
    *   `Ncurr`: An integer count of unrendered cells for each pixel, initialized by rendering billboards before processing [S1].
*   **Pixel State Variables:**
    *   `tcurr`: A running transparency value for each pixel, initialized to the pixel's initial transparency $t_p$ [S1].
    *   $t_c$: The transparency of the current cell $c$ [S1].
    *   $t_p$: The transparency of a pixel in the alpha image, calculated as $t_p = 1 - \alpha_p$ [S1].
    *   $N_{curr}$: The number of billboards not yet rendered at a specific pixel [S1].

**Parameters**
*   $M$: The resolution dimension of the square billboard texture ($M \times M$). Typical values are not specified in the sources [S1].
*   $\alpha_c$: The opacity of cell $c$ [S1].

**Algorithm Steps**
1.  **Initialization:** For each input RGBA image $i$, create a copy $C_i$. Render billboards to initialize `Ncurr` at each pixel using the stencil buffer [S1].
2.  **Cell Processing:** Iterate through each cell $c$ in the tree in front-to-back order [S1].
3.  **Projection:** Find the projected region $R$ of cell $c$ in the image copy $C_i$ [S1].
4.  **Resampling:** Create a resampled $M \times M$ texture $T$ from $C_i$ [S1].
5.  **Heuristic Calculation:** For each texel in $T$, calculate the importance metric $H_t$ using the formula:
    $$ H_t = \frac{1}{|t_{curr} - t_c^{N_{curr}}|} $$
    where $t_c^{N_{curr}}$ approximates the influence of the current cell on the cumulative transparency of all cells projecting to this pixel [S1]. Higher values are assigned when $t_{curr}$ is close to $t_c^{N_{curr}}$ [S1].
6.  **Sorting:** Sort the texels of $T$ by `age` (ascending) and then by $H_t$ (descending) [S1].
7.  **Selection:** Select the first $\lfloor \alpha_c \cdot M \cdot M_c \rfloor$ texels to be used on the billboard, where $\alpha_c$ is the cell opacity and $M_c$ is a scaling factor (Note: The source formula uses $M \cdot M_c$, but the context implies $M \times M$ total texels; the exact definition of $M_c$ is not provided in the sources, so it is treated as a given parameter) [S1].
8.  **Update Transparency:** For pixels in $R$ corresponding to selected texels, update `tcurr` by dividing it by $t_c$:
    $$ t_{curr} = \frac{t_{curr}}{t_c} $$
    This "removes" the influence of the current cell as estimated previously [S1].
9.  **Texture Generation:** Replace the selected pixels in $C_i$ using texture generation (interpolating neighboring pixels) [S1].
10. **Age Increment:** For remaining pixels in $R$ (those not selected), increment their `age` field [S1].
11. **Decrement Counter:** For all pixels in region $R$, decrement `Ncurr` by 1 [S1].
12. **Final Texture Assembly:** The final texture for cell $c$ and image $i$ is an $M \times M$ RGBA texture. The RGB values are taken from the modified copy $C_i$, and the Alpha value is taken from the original alpha image [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize image copies and stencil buffer
for each RGBA image i:
    C_i = copy(image_i)
    render_billboards_to_initialize_Ncurr(C_i)

# Process cells front-to-back
for each cell c in front-to-back order:
    # Find projected region R of cell c in image C_i
    R = get_projected_region(c, C_i)
    
    # Create resampled M x M texture T from C_i
    T = resample(C_i, size=M)
    
    # Calculate importance and sort texels
    for each texel t in T:
        # Get corresponding pixel state from C_i
        pixel = get_pixel_at_texel(t, C_i)
        t_curr = pixel.tcurr
        t_c = cell_transparency(c)
        N_curr = pixel.Ncurr
        
        # Calculate importance metric H_t
        # Note: Avoid division by zero if t_curr == t_c^N_curr
        denominator = abs(t_curr - (t_c ** N_curr))
        if denominator == 0:
            t.H_t = infinity
        else:
            t.H_t = 1.0 / denominator
            
    # Sort texels by age (ascending) then by H_t (descending)
    sort_texels(T, key=lambda x: (x.age, -x.H_t))
    
    # Select texels based on cell opacity
    num_texels_to_select = floor(c.opacity * M * M_c) # M_c definition not given in sources
    selected_texels = T[:num_texels_to_select]
    
    # Update transparency and replace pixels for selected texels
    for texel in selected_texels:
        pixel = get_pixel_at_texel(texel, C_i)
        t_c = cell_transparency(c)
        
        # Update running transparency
        pixel.tcurr = pixel.tcurr / t_c
        
        # Replace pixel using texture generation (interpolation)
        replace_pixel_with_interpolation(texel, C_i)
        
    # Increment age for remaining pixels in region R
    for pixel in pixels_in_region(R):
        if pixel not in selected_texels:
            pixel.age += 1
            
    # Decrement Ncurr for all pixels in region R
    for pixel in pixels_in_region(R):
        pixel.Ncurr -= 1
```

**Edge Cases and Pitfalls**
*   **Random Selection:** Random selection of texels yields unsatisfactory results; the heuristic importance metric is required for quality [S1].
*   **Unselected Pixels:** Some pixels may remain unselected, but this does not appear to cause significant artifacts [S1].
*   **Precision:** The formula involves division by the difference between transparencies. If $t_{curr}$ equals $t_c^{N_{curr}}$, the denominator is zero. The pseudocode handles this by assigning infinity to $H_t$, ensuring such texels are prioritized if sorting is stable, or handled by a guard [G].
*   **Parameter $M_c$:** The source formula $\lfloor \alpha_c \cdot M \cdot M_c \rfloor$ includes $M_c$, which is not defined in the provided notes. It is assumed to be a constant or parameter related to the texture resolution or cell size, but its exact value is "Not given in the sources." [S1].

## Operating characteristics
*   **Time Complexity:** The method requires iterating through all cells and all pixels in the projected regions. Sorting texels adds a logarithmic factor relative to the number of texels per cell.
*   **Memory Complexity:** Uncompressed textures require significant memory. The method suggests dissociating alpha from RGB and packing two RGB textures if their alphas do not intersect, reducing memory requirements to about 45% of the original uncompressed size (e.g., 45Mb for a 100Mb tree) [S1].
*   **Strengths:** Avoids blurring associated with simple projection; view-dependent; handles transparency accumulation heuristically.
*   **Limitations:** Requires stencil buffer support; memory intensive without compression; relies on heuristic approximation which may not be perfect for all transparency configurations.
*   **When to use:** When view-dependent rendering of volumetric data (like trees) is required and blurring from simple projection is unacceptable.
*   **When not to use:** When memory is extremely constrained and compression is not an option, or when hardware does not support stencil buffers.

## Background
Heuristic Billboard Texture Generation was introduced in the context of volumetric reconstruction and interactive rendering of trees from photographs [S1]. The method addresses the limitations of earlier texture slicing techniques that suffered from blurring when projecting cell data to image space [S1]. It is part of a pipeline that includes 3D Texture Generation and view-dependent rendering using multi-texturing [S1].

## Subtopics

### AR mapping to 3D texture
Uses augmented reality techniques to map 2D images to 3D structures. Chosen when precise alignment with real-world camera poses is critical. [AR mapping to 3D texture](884-ar-mapping-to-3d-texture.md)

### 3-Pass Texture Mapping with Compositing
A multi-pass approach for compositing textures. Chosen when complex blending operations are needed. [3-Pass Texture Mapping with Compositing](1354-3-pass-texture-mapping-with-compositing.md)

### 3D Texture Generation via Slicing
Generates textures by slicing the 3D volume. Chosen for simpler, non-view-dependent scenarios. [3D Texture Generation via Slicing](4891-3d-texture-generation-via-slicing.md)

### Post-packing Optimisation
Optimizes texture packing after generation. Chosen to reduce memory footprint. [Post-packing Optimisation](11679-post-packing-optimisation.md)

### Texture Slicing Pipeline
The overall pipeline for slicing textures. Chosen as the foundational process. [Texture Slicing Pipeline](13410-texture-slicing-pipeline.md)

### Isotropic Texture Synthesis
Synthesizes textures isotropically. Chosen when view-independence is desired. [Isotropic Texture Synthesis](13939-isotropic-texture-synthesis.md)

### Model-based Solid Texture Synthesis
Uses geometric models for texture synthesis. Chosen for high-fidelity procedural textures. [Model-based Solid Texture Synthesis](21039-model-based-solid-texture-synthesis.md)

## Sources
[S1] Volumetric reconstruction and interactive rendering of trees from photographs, https://doi.org/10.1145/1015706.1015785, fetched 2026-10-26
