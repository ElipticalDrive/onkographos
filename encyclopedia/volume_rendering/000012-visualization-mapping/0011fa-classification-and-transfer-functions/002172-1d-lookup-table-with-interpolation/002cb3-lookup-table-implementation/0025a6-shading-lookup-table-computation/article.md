# Shading Lookup Table Computation

Also called: Internal texture format selection, vpShadeTable

## Overview
Shading Lookup Table Computation is the process of generating a 1D lookup table (LUT) where each entry contains the final shaded color and opacity for a specific scalar value, derived from current lighting, material, and view parameters [S2]. It solves the problem of decoupling the volumetric shading calculation from the rendering loop to enable interactive manipulation of visual quantities [S1, S3]. Unlike methods that store pre-shaded colors, this approach stores color indices (or raw scalar mappings) and computes the shading dynamically during texture sampling, allowing real-time updates to lighting and material properties without regenerating the entire volume texture [S1, S3].

## How it works
The method operates in **Color Index Mode**, where the LUT stores indices or unshaded values that are transformed into color values using texture lookup tables during rendering [S3]. This contrasts with pre-shaded values, which are static and lack interactivity [S3].

**Inputs and Outputs:**
*   **Input:** Current lighting parameters (ambient, diffuse, specular, shinyness), material properties, view transformation matrix, and the base transfer function mapping scalar values to extinction coefficients or base colors [S1, S2].
*   **Output:** A 1D texture or buffer containing 4-component vectors $(R, G, B, \alpha)$, where $R, G, B$ are the shaded colors and $\alpha$ is the opacity [S2, S4, unofficial].

**Data Structures:**
The LUT is a contiguous array of 4-component vectors. In VolPack, this is managed internally by the context `vpc` [S2]. The entries are typically 32-bit floats [S2, S4, unofficial].

**Steps:**
1.  **Configure Shading Parameters:** Set ambient, diffuse, specular, and shinyness values for the material using the context [S2].
2.  **Set View Transformation:** Define the current view matrix and lighting direction relative to the view [S2].
3.  **Compute Table Entries:** Call the computation routine (e.g., `vpShadeTable`) to iterate over all entries in the defined color table [S2].
    *   For each entry $i$, retrieve the base scalar value or extinction coefficient $map[C_i]$ [S1].
    *   Apply the current lighting model to compute the RGB color components [S2].
    *   Compute the opacity $\alpha_i$ based on the sample slice distance $\Delta$ and the extinction coefficient [S1]. The opacity is computed as:
        $$ \alpha_i = 1 - e^{-map[C_i] \Delta} $$
        where $\Delta$ is the sample slice distance between consecutive cutting planes [S1].
    *   Store $(R_i, G_i, B_i, \alpha_i)$ in the LUT [S2].
4.  **Finalize:** Ensure the table is uploaded to GPU memory if using texture lookup [S2].

**Parameters:**
*   `vpShadeTable` context pointer `vpc` [S2].
*   Sample slice distance $\Delta$: Initially chosen with respect to the width of texture elements [S1]. If $\Delta$ changes, opacity values in the color table must be corrected [S1].

**Edge Cases and Pitfalls:**
*   **Timing:** `vpShadeTable()` must be called after any changes to shading or viewing parameters, but *before* calling any rendering routines [S2]. Failure to do so results in stale shading data.
*   **Precision:** Opacity correction is required when the sample slice distance changes to maintain visual consistency across different resolution levels [S1].
*   **Boundary Errors:** Projection of adjoining voxels at boundaries between different resolution levels may result in erroneously drawn pixels; this requires opacity correction by polygon clipping [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vpShadeTable(vpc):
    """
    Recomputes the color lookup table entries based on current lighting,
    material, and view transformation parameters.
    """
    # 1. Retrieve current shading and view parameters from vpc [S2]
    ambient = vpc.get_lighting(VP_AMBIENT)
    diffuse = vpc.get_material(VP_MATERIAL0, VP_DIFFUSE)
    specular = vpc.get_material(VP_MATERIAL0, VP_SPECULAR)
    shinyness = vpc.get_material(VP_MATERIAL0, VP_SHINYNESS)
    view_transform = vpc.get_view_matrix()
    
    # 2. Get the base transfer function mapping [S1]
    # map[Ci] defines the mapping from color indices to extinction coefficients
    base_map = vpc.get_transfer_function_map()
    
    # 3. Get the current sample slice distance [S1]
    delta = vpc.get_sample_slice_distance()
    
    # 4. Iterate over all entries in the color table [S2]
    num_entries = vpc.get_lookup_table_size()
    for i in range(num_entries):
        # Get the base scalar/extinction value for this index [S1]
        extinction_coeff = base_map[i]
        
        # Compute opacity using Beer-Lambert law [S1]
        # alpha = 1 - exp(-extinction_coeff * delta)
        alpha = 1.0 - math.exp(-extinction_coeff * delta)
        
        # Compute shaded color based on lighting model [S2]
        # This step involves standard Phong or similar shading calculations
        # using ambient, diffuse, specular, and shinyness [S2]
        color = compute_shaded_color(
            base_color=base_map.get_color(i), # Assuming base color retrieval
            ambient=ambient,
            diffuse=diffuse,
            specular=specular,
            shinyness=shinyness,
            view_transform=view_transform
        )
        
        # Store the result in the LUT [S2]
        vpc.set_lookup_table_entry(i, color.r, color.g, color.b, alpha)
```

## Subtopics

### Color Index Mode with LUT
Color Index Mode stores textures as color indices that are transformed to color values using texture lookup tables, rather than storing pre-shaded color values [S3]. This mode is chosen because it allows interactive manipulation of visual quantities (such as lighting and material changes) in real-time [S3]. In contrast, pre-shaded color values are often favored in practical applications for performance but lack this interactivity [S3]. This method builds on [Shading Lookup Table Computation](3589-shading-lookup-table-computation.md) by ensuring the LUT contains indices or unshaded data that can be dynamically shaded [G].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of entries in the lookup table. The computation is performed once per shading parameter change [S2].
*   **Memory Complexity:** $O(N)$ for storing the 4-component vector table [S2].
*   **Performance:** Enables interactive manipulation of visual quantities, which is the primary advantage over pre-shaded methods [S3]. However, it requires recomputation of the entire table when lighting or view parameters change [S2].
*   **Strengths:** Real-time interactivity with lighting and materials; hierarchical decomposition benefits [S1, S3].
*   **Limitations:** Requires careful handling of opacity correction when sample slice distances change [S1]; potential boundary errors between resolution levels [S3].
*   **When to use:** When interactive lighting and material manipulation is required during volume rendering [S3].
*   **When not to use:** When static, pre-shaded textures are sufficient for performance and interactivity is not needed [S3].

## Background
The concept originates from Level-of-Detail Volume Rendering via 3D Textures, which introduced the use of color index mode with texture lookup tables to allow interactive manipulation of visual quantities [S1]. The VolPack library formalized this with the `vpShadeTable` routine, which computes lookup table entries based on current lighting and view parameters [S2]. The approach was further discussed in the context of hierarchical decomposition and brick boundaries [S3].

## Sources
[S1] Level-of-Detail Volume Rendering via 3D Textures, https://www.semanticscholar.org/paper/05e7d88387ac7484397e329456535b0c8c665a16, fetched 2026-11-03
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S3] PDF Level-Of-Detail Volume Rendering via 3D Textures, https://publications.sci.utah.edu/publications/weiler00/IEEE-VolVis-2000.pdf, fetched 2026-10-27
