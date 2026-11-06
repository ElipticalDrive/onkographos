# Shading Lookup Table Computation

Also called: Internal texture format selection, vpShadeTable

## Overview
Shading Lookup Table Computation generates or updates a color lookup table (LUT) that maps scalar data values to colors and opacity using current lighting, material, and view transformation parameters [S2]. It solves the problem of efficiently applying shading models to volume data by pre-computing these effects into a 1D texture, allowing the renderer to fetch shaded colors via simple texture lookups [S2]. It is distinguished from its siblings by its reliance on color index mode (storing indices and transforming them at render time) rather than pre-shaded color values, which enables interactive manipulation of visual quantities like lighting and material properties [S1, S3].

## How it works
### Inputs and Outputs
*   **Inputs**:
    *   Current lighting parameters (ambient, diffuse, specular coefficients, light positions) [S2].
    *   Current material parameters (diffuse, specular, shinyness, reflectivity) [S2].
    *   Current view transformation matrix [S2].
    *   The scalar field data values (implicitly, via the mapping function) [S1].
    *   Sample slice distance ($\Delta$) for opacity correction [S1].
*   **Outputs**:
    *   A color lookup table (LUT) where each entry contains a color index and associated opacity values [S1].
    *   The LUT is stored as color indices, not final RGB values [S1, S3].

### Data Structures
The LUT is typically implemented as a 1D texture or an array [S1]. Each entry corresponds to a specific scalar value or color index $C_i$ [S1]. Fields include `color_index` and `opacity` [S1]. Bit packing is not specified in the sources [S2].

### Steps
1. Initialize shading parameters by setting ambient, diffuse, specular, and shinyness parameters for the materials [S2].
2. Set viewing parameters to define the view transformation and camera parameters [S2].
3. Declare the lookup shader to configure the lookup table structure and size [S2].
4. Compute the shading table to regenerate the LUT entries based on the current lighting, material, and view parameters [S2].
5. Apply opacity correction if the sample slice distance $\Delta$ changes [S1]. For a color index $C_i$, the initial opacity $\alpha_0$ is computed as:
   $$ \alpha_0 = 1 - e^{-\text{map}[C_i] \Delta_0} $$
   where $\text{map}[C_i]$ is the mapping from color index to extinction coefficient [S1]. If the distance changes to $\Delta_k$, the opacity must be corrected using the new $\Delta_k$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_shading_entry(color_index, extinction_map, lighting, material, view_transform):
    """Computes the shaded color and opacity for a single color index."""
    extinction = extinction_map(color_index)
    # The exact shading model (e.g., Phong, Blinn-Phong) is not specified in the sources [G].
    color = apply_shading_model(lighting, material, view_transform, extinction)
    opacity = 1.0 - math.exp(-extinction)
    return color, opacity

def update_shading_lookup_table(lut_size, extinction_map, lighting, material, view_transform):
    """Recomputes the color lookup table entries based on current scene parameters."""
    new_lut = []
    for i in range(lut_size):
        color, opacity = compute_shading_entry(i, extinction_map, lighting, material, view_transform)
        new_lut.append({"color_index": i, "color": color, "opacity": opacity})
    return new_lut

def correct_opacity_for_slice_distance(lut, new_delta, extinction_map):
    """Corrects opacity values in the LUT when the sample slice distance changes."""
    for entry in lut:
        extinction = extinction_map(entry["color_index"])
        entry["opacity"] = 1.0 - math.exp(-extinction * new_delta)
    return lut
```

### Parameters
*   **Color Index Mode**: The method assumes color index mode [S1].
*   **Sample Slice Distance ($\Delta$)**: The distance between consecutive cutting planes [S1].

### Edge Cases and Pitfalls
*   **Timing of Computation**: The computation routine must be called after any changes to shading or viewing parameters, but before calling any rendering routines [S2].
*   **Boundary Errors**: Even with opacity correction, projection of adjoining voxels at boundaries between different resolution levels might result in erroneously drawn pixels [S1].
*   **Precision**: The sources do not specify precision requirements or floating-point handling constraints for the exponential calculation [S1].

## Subtopics

### Color Index Mode with LUT
This variant refers to the specific implementation choice of storing textures as color indices that are transformed to color values using texture lookup tables during rendering, rather than storing pre-shaded color values [S3]. It is chosen over pre-shaded values because it allows for the interactive manipulation of visual quantities (such as lighting and material properties) [S3]. Pre-shaded values are often favored in practical applications for performance but lack this interactivity [S3]. This mode is a prerequisite for the Shading Lookup Table Computation method described above [S1, S3].

## Operating characteristics
*   **Time Complexity**: The computation of the shading table is performed once per update of lighting/view parameters [S2]. The cost is proportional to the size of the LUT [S2]. Rendering time is reduced because shading is done via texture lookup [S2].
*   **Memory Complexity**: Requires memory for the LUT [S1]. The size depends on the number of color indices [S1].
*   **Performance**: Enables interactive manipulation of visual quantities, which is a key advantage over pre-shaded methods [S3].
*   **Strengths**: Interactivity and efficient rendering via texture lookup [S3].
*   **Limitations**: Requires careful management of LUT updates and potential boundary errors at resolution level transitions [S1, S3].
*   **When to Use**: When interactive lighting and material changes are required during volume rendering [S3].
*   **When Not to Use**: When maximum rendering performance is critical and lighting/materials are static, as pre-shaded values may be better [S3].

## Background
The method originates from the VolPack library and related research on level-of-detail volume rendering via 3D textures [S1, S2]. It was developed to support interactive volume rendering by decoupling shading from the rendering loop through the use of color index mode and lookup tables [S1, S3]. The approach is detailed in the VolPack User's Guide and the paper "Level-of-Detail Volume Rendering via 3D Textures" [S1, S2, S3].

## Sources
[S1] Level-of-Detail Volume Rendering via 3D Textures, https://www.semanticscholar.org/paper/05e7d88387ac7484397e329456535b0c8c665a16, fetched 2026-11-03
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S3] PDF Level-Of-Detail Volume Rendering via 3D Textures, https://publications.sci.utah.edu/publications/weiler00/IEEE-VolVis-2000.pdf, fetched 2026-10-27
