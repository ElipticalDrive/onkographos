# Shading Lookup Table Computation

Also called: Internal texture format selection, vpShadeTable

## Overview
Shading Lookup Table Computation is the process of generating a 1D texture that maps scalar volume data values to RGBA colors based on current lighting, material, and view transformation parameters. It solves the problem of dynamic visual feedback in volume rendering by decoupling the shading calculation from the per-voxel rendering pass. Unlike methods that pre-compute static colors, this approach allows for the interactive manipulation of visual quantities (such as lighting intensity or material properties) without re-rendering the entire volume geometry [S1, S3]. It builds on [Lookup Table Implementation](5347-lookup-table-implementation.md) by specifically addressing the generation of the color indices within that table.

## How it works
The method computes the entries of a color lookup table (LUT) by applying a shading model to the transfer function's color mapping.

**Inputs and Outputs**
*   **Input:** Current lighting parameters (ambient, diffuse, specular, shinyness), material properties, view transformation matrix, and the base transfer function mapping scalar values to extinction coefficients or base colors.
*   **Output:** A 1D texture (LUT) where each entry contains the final computed $(R, G, B, \alpha)$ values for a specific scalar input index.

**Data Structures**
The LUT is stored as a contiguous array of 4-component vectors.
*   **Format:** `float4` (R, G, B, A), where each component is a 32-bit float [S5, unofficial].
*   **Size:** Typically 256 entries for 8-bit data [S1] or 4096 entries for higher precision [S3].
*   **Storage:** Uploaded to GPU memory as a 1D CUDA array bound to a texture object [S5, unofficial].

**Algorithm Steps**
1.  **Initialize Parameters:** Set the lighting model (e.g., Phong) and material properties (diffuse, specular, shininess) [S2].
2.  **Iterate LUT Entries:** For each index $i$ in the LUT (from $0$ to $N-1$):
    a.  **Retrieve Base Value:** Get the scalar value $v_i$ associated with index $i$. In color index mode, this is often the index itself or a mapped extinction coefficient [S1].
    b.  **Compute Normal:** Determine the surface normal $\mathbf{n}$ for this scalar value. In VolPack, this is often encoded or derived from the transfer function's gradient or a default assumption [S2, G].
    c.  **Transform Normal:** Apply the current view transformation to the normal $\mathbf{n}$ to get the view-space normal $\mathbf{n}'$ [S2].
    d.  **Shade:** Calculate the color $C_i$ and opacity $\alpha_i$ using the shading model:
        $$ C_i = \text{ShadeModel}(v_i, \mathbf{n}', \text{LightingParams}, \text{MaterialParams}) $$
        $$ \alpha_i = \text{TransferFunctionOpacity}(v_i) $$
        *Note: In VolPack, `vpShadeTable` uses the current lighting and material settings to compute these values [S2].*
    e.  **Store:** Write $(C_i, \alpha_i)$ to the LUT at index $i$.
3.  **Upload:** Bind the updated LUT to the texture unit for rendering [S5, unofficial].

**Parameters**
*   **Lighting Model:** Typically Phong or Blinn-Phong [S2].
*   **Shininess:** Controls specular highlight size [S2].
*   **View Transformation:** Must be updated before calling the computation routine [S2].

**Edge Cases and Pitfalls**
*   **Timing:** `vpShadeTable()` must be called after any changes to shading or viewing parameters, but *before* calling any rendering routines [S2]. Failure to do so results in stale shading data.
*   **Monotonicity:** If the resulting opacity values are not monotonic, DDA optimizations may fail. A CDF transformation may be required if the shading model introduces non-monotonic opacity behavior [S2].
*   **Precision:** Floating-point precision in the LUT entries affects the smoothness of color transitions [S5, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shading_lut(lut_size, lighting_params, material_params, view_transform, transfer_function):
    """
    Computes the shading lookup table entries.
    
    Args:
        lut_size (int): Number of entries in the LUT (must be power of 2).
        lighting_params (dict): Contains ambient, diffuse, specular colors and intensity.
        material_params (dict): Contains diffuse, specular colors and shininess.
        view_transform (Matrix4x4): Current view transformation matrix.
        transfer_function (function): Maps scalar value to base color and opacity.
        
    Returns:
        list: List of float4 tuples (R, G, B, A) representing the LUT.
    """
    lut = []
    
    for i in range(lut_size):
        # 1. Get scalar value for this index
        # In color index mode, the index itself often represents the scalar value
        scalar_value = i / (lut_size - 1) if lut_size > 1 else 0.0
        
        # 2. Get base color and opacity from transfer function
        base_color, base_opacity = transfer_function(scalar_value)
        
        # 3. Determine normal vector
        # [G] VolPack often assumes a normal or derives it from the transfer function gradient.
        # For a generic implementation, we assume a default normal (e.g., (0,0,1)) 
        # or that the transfer function provides a normal.
        normal = Vector3(0.0, 0.0, 1.0) 
        
        # 4. Transform normal to view space
        # [G] Assuming view_transform is a 4x4 matrix. 
        # We transform the normal as a direction vector (ignore translation).
        view_normal = view_transform.transform_direction(normal)
        view_normal = normalize(view_normal)
        
        # 5. Compute shading
        # [G] Simplified Phong shading model
        light_direction = normalize(Vector3(1.0, 1.0, 1.0)) # Example light direction
        view_direction = normalize(Vector3(0.0, 0.0, 1.0))  # Example view direction
        
        # Diffuse component
        diff = max(dot(view_normal, light_direction), 0.0)
        diffuse_color = material_params['diffuse'] * lighting_params['diffuse'] * diff
        
        # Specular component
        reflect_dir = reflect(-light_direction, view_normal)
        spec = pow(max(dot(view_direction, reflect_dir), 0.0), material_params['shininess'])
        specular_color = material_params['specular'] * lighting_params['specular'] * spec
        
        # Ambient component
        ambient_color = material_params['diffuse'] * lighting_params['ambient']
        
        # Final color
        final_color = base_color * (ambient_color + diffuse_color + specular_color)
        
        # 6. Store in LUT
        lut.append(Float4(final_color.r, final_color.g, final_color.b, base_opacity))
        
    return lut
```

## Subtopics

### Color Index Mode with LUT
Color Index Mode with LUT is a variant where the texture stores color indices rather than pre-shaded RGB values. These indices are transformed to color values using the shading lookup table during rendering [S3]. This mode is chosen over pre-shaded color values because it allows for the interactive manipulation of visual quantities (such as lighting or material changes) without re-rendering the volume [S3]. Pre-shaded values are often favored in practical applications for performance but lack this interactivity [S3].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of entries in the LUT. This is a one-time cost per shading parameter change [S2].
*   **Memory Complexity:** $O(N)$ for storing the LUT. Typically 256 or 4096 entries [S1, S3].
*   **Performance:** The computation is fast enough to be performed interactively when lighting or material parameters change [S1]. The actual rendering remains efficient because the shading is pre-computed into the LUT [S1].
*   **Strengths:** Enables real-time interactive visualization of lighting and material effects [S3].
*   **Limitations:** Requires recomputation of the entire LUT if lighting or view parameters change [S2]. Not suitable for dynamic lighting that changes per-voxel or per-sample during rendering [S1].

## Background
The method originates from VolPack, a volume rendering library developed by Steve Westin and others at Stanford University [S2]. The concept of using color index mode with texture lookup tables for interactive volume rendering was further explored in the context of level-of-detail rendering by Weiler et al. [S1, S3]. The `vpShadeTable` function is a specific API call in VolPack that implements this computation [S2].

## Sources
[S1] Level-of-Detail Volume Rendering via 3D Textures, https://www.semanticscholar.org/paper/05e7d88387ac7484397e329456535b0c8c665a16, fetched 2026-11-03
[S2] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S3] PDF Level-Of-Detail Volume Rendering via 3D Textures, https://publications.sci.utah.edu/publications/weiler00/IEEE-VolVis-2000.pdf, fetched 2026-10-27
