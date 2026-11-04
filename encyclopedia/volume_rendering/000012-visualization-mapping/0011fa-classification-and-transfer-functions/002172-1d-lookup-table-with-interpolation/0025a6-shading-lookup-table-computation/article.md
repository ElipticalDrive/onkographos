# Shading Lookup Table Computation

Also called: Internal texture format selection, vpShadeTable

## Overview
Shading Lookup Table Computation is the process of generating or updating a color lookup table (LUT) that maps scalar data values to RGB colors and opacity, incorporating current lighting, material, and view transformation parameters. It solves the problem of efficiently applying complex shading models to volume data during rendering by pre-computing these effects into a 1D texture, allowing the renderer to fetch shaded colors via simple texture lookups. It is distinguished from its siblings by its reliance on **color index mode** (storing indices and transforming them at render time) rather than pre-shaded color values, which enables interactive manipulation of visual quantities like lighting and material properties [S1, S3].

## How it works
The method operates within the context of a volume rendering system that uses 3D textures and hierarchical decomposition (such as brick-based structures). The core mechanism involves recomputing the entries of a color lookup table based on dynamic scene parameters.

### Inputs and Outputs
*   **Inputs**:
    *   Current lighting parameters (ambient, diffuse, specular coefficients, light positions).
    *   Current material parameters (diffuse, specular, shinyness, reflectivity).
    *   Current view transformation matrix.
    *   The scalar field data values (implicitly, via the mapping function).
    *   Sample slice distance ($\Delta$) for opacity correction [S1].
*   **Outputs**:
    *   A color lookup table (LUT) where each entry contains a color index and potentially associated opacity values. The LUT is stored as color indices, not final RGB values [S1, S3].

### Data Structures
The LUT is typically implemented as a 1D texture or an array.
*   **Entry Layout**: Each entry corresponds to a specific scalar value or color index $C_i$.
*   **Fields**:
    *   `color_index`: An integer or normalized value representing the index into the color map.
    *   `opacity`: The opacity value $\alpha$ associated with the scalar value.
*   **Bit Packing**: Not specified in the sources. The implementation uses "encoded normal vectors" or similar structures depending on the shading model, but the exact bit layout is not detailed [S2].

### Steps
1.  **Initialize Shading Parameters**: Set ambient, diffuse, specular, and shinyness parameters for the materials using the volume context (e.g., `vpSetMaterial`) [S2].
2.  **Set Viewing Parameters**: Define the view transformation and camera parameters [S2].
3.  **Declare Lookup Shader**: Configure the lookup table structure and size using a function like `vpSetLookupShader()` [S2].
4.  **Compute Shading Table**: Call the computation routine (e.g., `vpShadeTable`) to regenerate the LUT entries based on the current lighting, material, and view parameters [S2].
5.  **Opacity Correction (if applicable)**: If the sample slice distance $\Delta$ changes, correct the opacity values in the LUT. For a color index $C_i$, the initial opacity $\alpha_0$ is computed as:
    $$ \alpha_0 = 1 - e^{-\text{map}[C_i] \Delta_0} $$
    where $\text{map}[C_i]$ is the mapping from color index to extinction coefficient. If the distance changes to $\Delta_k$, the opacity must be corrected using the new $\Delta_k$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shading_lookup_table(context):
    """
    Recomputes the color lookup table entries based on current lighting,
    material, and view transformation parameters.
    
    Args:
        context: The volume rendering context containing lighting, material,
                 and view parameters.
    
    Returns:
        None. Updates the internal color lookup table in the context.
    """
    # Step 1: Ensure lighting and material parameters are set
    # (Assumed to be done via vpSetMaterial/vpSetLight calls prior)
    
    # Step 2: Ensure viewing parameters are set
    # (Assumed to be done via vpSetView calls prior)
    
    # Step 3: Call the shading table computation routine
    # This function computes all entries in the currently-defined color table
    # using the current lighting and material parameters and the current view transformation.
    result = vpShadeTable(context)
    
    if result != SUCCESS:
        raise RuntimeError("vpShadeTable failed")

def correct_opacity_for_slice_distance(lut, old_delta, new_delta, map_function):
    """
    Corrects opacity values in the LUT when the sample slice distance changes.
    
    Args:
        lut: The lookup table array.
        old_delta: The previous sample slice distance.
        new_delta: The new sample slice distance.
        map_function: A function that maps color index to extinction coefficient.
    """
    for index in range(len(lut)):
        # Get extinction coefficient for this color index
        extinction = map_function(index)
        
        # Recompute opacity using the new delta
        # Formula: alpha = 1 - exp(-extinction * delta)
        new_opacity = 1.0 - math.exp(-extinction * new_delta)
        
        # Update the LUT entry
        lut[index].opacity = new_opacity
```

### Parameters
*   **Color Index Mode**: The method assumes color index mode. This is a critical parameter/setting that determines whether the LUT stores indices or pre-shaded colors [S1].
*   **Sample Slice Distance ($\Delta$)**: The distance between consecutive cutting planes. It affects opacity calculation [S1].

### Edge Cases and Pitfalls
*   **Timing of Computation**: `vpShadeTable()` must be called **after** any changes to shading or viewing parameters, but **before** calling any rendering routines. Failure to do so results in stale shading data [S2].
*   **Boundary Errors**: Even with opacity correction, projection of adjoining voxels at boundaries between different resolution levels might result in erroneously drawn pixels. This requires additional handling, such as polygon clipping [S1, S3].
*   **Precision**: The sources do not specify precision issues, but the exponential calculation for opacity correction requires careful handling of floating-point values to avoid artifacts [S1].

## Subtopics

### Color Index Mode with LUT
This variant refers to the specific implementation choice of storing textures as color indices that are transformed to color values using texture lookup tables during rendering, rather than storing pre-shaded color values. It is chosen over pre-shaded values because it allows for the interactive manipulation of visual quantities (such as lighting and material properties) [S3]. Pre-shaded values are often favored in practical applications for performance but lack this interactivity [S3]. This mode is a prerequisite for the Shading Lookup Table Computation method described above [S1, S3].

## Operating characteristics
*   **Time Complexity**: The computation of the shading table is performed once per update of lighting/view parameters. The cost is proportional to the size of the LUT. Rendering time is reduced because shading is done via texture lookup.
*   **Memory Complexity**: Requires memory for the LUT. The size depends on the number of color indices.
*   **Performance**: Enables interactive manipulation of visual quantities, which is a key advantage over pre-shaded methods [S3].
*   **Strengths**: Interactivity, efficient rendering via texture lookup.
*   **Limitations**: Requires careful management of LUT updates; potential boundary errors at resolution level transitions [S1, S3].
*   **When to Use**: When interactive lighting and material changes are required during volume rendering.
*   **When Not to Use**: When maximum rendering performance is critical and lighting/materials are static (pre-shaded values may be better) [S3].

## Background
The method originates from the VolPack library and related research on level-of-detail volume rendering via 3D textures. It was developed to support interactive volume rendering by decoupling shading from the rendering loop through the use of color index mode and lookup tables [S1, S2, S3]. The approach is detailed in the VolPack User's Guide and the paper "Level-of-Detail Volume Rendering via 3D Textures" [S1, S2, S3].

## Sources
[S1] Level-of-Detail Volume Rendering via 3D Textures, https://www.semanticscholar.org/paper/05e7d88387ac7484397e329456535b0c8c665a16, fetched 2026-11-03
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S3] PDF Level-Of-Detail Volume Rendering via 3D Textures, https://publications.sci.utah.edu/publications/weiler00/IEEE-VolVis-2000.pdf, fetched 2026-10-27
