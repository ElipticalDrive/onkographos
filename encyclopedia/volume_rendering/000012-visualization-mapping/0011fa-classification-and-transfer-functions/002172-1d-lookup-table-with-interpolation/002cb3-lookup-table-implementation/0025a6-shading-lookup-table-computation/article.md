# Shading Lookup Table Computation

Also called: Internal texture format selection, vpShadeTable

## Overview
Shading Lookup Table Computation is the process of generating the color and opacity entries for a 1D transfer function lookup table (LUT) by applying a shading model to the scalar field values. It solves the problem of how to map raw scalar data to visually meaningful RGBA values that account for lighting, material properties, and view orientation, rather than using static, pre-shaded colors [S1, S3]. It differs from [Hardcoded Transfer Function](1098-hardcoded-transfer-function.md) by computing values dynamically based on current scene parameters, and from [UpdateOpacityTransferFunction](2489-updateopacitytransferfunction.md) by encompassing the full color/opacity computation rather than just opacity adjustments.

## How it works
The method computes the entries of a color lookup table by evaluating a shading model for each index in the table. The core input is the scalar field value associated with each LUT entry, which is mapped to a color index. The output is a populated 1D texture or array containing $(R, G, B, \alpha)$ values [S1, S3].

**Data Structures:**
The LUT is stored as a contiguous array of 4-component vectors. The standard format is `float4` (32-bit floats per component) or 8-bit integers, depending on the implementation [S1, S2, S3]. The width $N$ must be a power of two [S1, S2, S3].

**Inputs:**
1.  Scalar field values $v_i$ corresponding to each LUT index $i$.
2.  Current lighting parameters (ambient, diffuse, specular coefficients, light positions).
3.  Current material parameters (diffuse color, specular color, shininess).
4.  Current view transformation parameters (view vector, normal vectors if encoded).
5.  Sample slice distance $\Delta$ (used for opacity correction) [S1].

**Steps:**
1.  **Initialize Parameters:** Set lighting and material properties using the shading model (e.g., Phong or Blinn-Phong) [S2].
2.  **Iterate LUT Entries:** For each index $i$ from $0$ to $N-1$:
    a. Retrieve the scalar value $v_i$ or its associated normal vector $n_i$ (if using encoded normals) [S2].
    b. Transform the normal vector $n_i$ into view space using the current view transformation matrix [S2].
    c. Compute the shading contribution (ambient, diffuse, specular) based on the transformed normal, view vector, and light parameters [S2].
    d. Map the scalar value $v_i$ to an extinction coefficient or opacity value $\alpha_i$ using the transfer function mapping $map[C_i]$ [S1].
    e. Apply opacity correction if the sample slice distance $\Delta$ has changed. The opacity entry $\alpha_0$ is computed as:
        $$ \alpha_0 = 1 - e^{-map[C_i] \Delta_0} $$
        If the distance changes to $\Delta_k$, the opacity values must be corrected accordingly [S1].
    f. Store the resulting $(R, G, B, \alpha)$ in the LUT at index $i$ [S2].
3.  **Upload/Commit:** Upload the computed LUT to GPU memory or make it available for rendering [S1, S2].

**Pitfalls:**
*   **Timing:** `vpShadeTable()` (or equivalent) must be called after any changes to shading or viewing parameters, but *before* calling any rendering routines [S2].
*   **Color Index Mode:** The method assumes color index mode, where textures store color indices that are transformed to color values during rendering, rather than pre-shaded color values [S1, S3].
*   **Precision:** Opacity correction requires accurate handling of the exponential mapping to avoid artifacts at resolution boundaries [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shading_lookup_table(lut_size, scalar_values, lighting_params, material_params, view_matrix, sample_distance):
    """
    Computes the shading lookup table entries.
    
    Args:
        lut_size (int): Number of entries (power of two).
        scalar_values (list): Scalar values for each LUT index.
        lighting_params (dict): Ambient, diffuse, specular coefficients and light positions.
        material_params (dict): Diffuse color, specular color, shininess.
        view_matrix (matrix): Current view transformation matrix.
        sample_distance (float): Current sample slice distance Delta.
        
    Returns:
        list: List of (R, G, B, alpha) tuples for the LUT.
    """
    lut = []
    
    # Pre-calculate view-space light directions if needed
    # [G] Assuming standard Phong shading model
    
    for i in range(lut_size):
        v = scalar_values[i]
        
        # Retrieve or compute normal vector for this scalar value
        # [G] In VolPack, normals are often encoded in the LUT or derived from the scalar field gradient
        # For this generic implementation, assume a normal n is available or computed
        n = get_normal_for_scalar(v) 
        
        # Transform normal to view space
        n_view = transform_vector(view_matrix, n)
        
        # Compute shading
        # [G] Simplified Phong shading
        ambient = lighting_params['ambient'] * material_params['diffuse']
        
        # Diffuse
        light_dir = normalize(lighting_params['light_direction'])
        diff_intensity = max(0, dot(n_view, light_dir))
        diffuse = diff_intensity * lighting_params['diffuse_coeff'] * material_params['diffuse']
        
        # Specular
        view_dir = normalize(-view_matrix[0:3, 3]) # Simplified view vector
        reflect_dir = reflect(-light_dir, n_view)
        spec_intensity = pow(max(0, dot(view_dir, reflect_dir)), material_params['shininess'])
        specular = spec_intensity * lighting_params['specular_coeff'] * material_params['specular']
        
        # Combine color
        R = min(1.0, ambient[0] + diffuse[0] + specular[0])
        G = min(1.0, ambient[1] + diffuse[1] + specular[1])
        B = min(1.0, ambient[2] + diffuse[2] + specular[2])
        
        # Compute opacity from scalar value
        # [G] Assuming a mapping function map[C_i] exists for extinction coefficient
        extinction = map_scalar_to_extinction(v)
        alpha = 1.0 - exp(-extinction * sample_distance)
        
        lut.append((R, G, B, alpha))
        
    return lut
```

## Subtopics

### Color Index Mode with LUT
Color Index Mode with LUT is a variant where the texture stores color indices rather than pre-shaded color values. These indices are transformed to color values using the shading lookup table during rendering [S3]. It is chosen over pre-shaded color values because it allows interactive manipulation of visual quantities (such as lighting and material properties) at runtime, whereas pre-shaded values are static and lack this interactivity [S3]. Pre-shaded values are often favored in practical applications for performance, but Color Index Mode is necessary when dynamic shading is required [S1, S3].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of LUT entries. Each entry requires a shading computation [S2].
*   **Memory Complexity:** $O(N)$ for storing the LUT [S1, S2, S3].
*   **Performance:** The computation is done once per frame or when parameters change, not per voxel, making it efficient for interactive rendering [S1, S2].
*   **Strengths:** Enables dynamic lighting and material changes without re-rendering the entire volume or pre-computing multiple LUTs [S1, S3].
*   **Limitations:** Requires the assumption of color index mode, which may not be supported by all hardware or software pipelines [S1, S3]. Opacity correction adds complexity [S1].

## Background
The method originates from early volume rendering systems that used 3D textures and color lookup tables for interactive visualization. It was notably implemented in VolPack, a software volume rendering library, where `vpShadeTable()` was the routine responsible for recomputing the color lookup table entries based on current lighting, material, and view transformation parameters [S2]. The approach was also discussed in the context of level-of-detail volume rendering via 3D textures, where color index mode was preferred for interactive manipulation of visual quantities [S1, S3].

## Sources
[S1] Level-of-Detail Volume Rendering via 3D Textures, https://www.semanticscholar.org/paper/05e7d88387ac7484397e329456535b0c8c665a16, fetched 2026-11-03
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S3] PDF Level-Of-Detail Volume Rendering via 3D Textures, https://publications.sci.utah.edu/publications/weiler00/IEEE-VolVis-2000.pdf, fetched 2026-10-27
