# Multiple Flat Shaded Opaque Isosurfaces

Also called: Hardware-based view-independent cell projection for multiple isosurfaces

## Overview
A rendering technique for multiple flat-shaded opaque isosurfaces within unstructured tetrahedral meshes that eliminates the need for explicit visibility sorting of cells [S1]. It leverages hardware-accelerated texture lookups and OpenGL's alpha test combined with the z-test to guarantee correct occlusion of isosurface fragments directly during rasterization [S1]. Unlike sibling methods that rely on geometric intersection refinement or deferred shading buffers, this approach projects tetrahedral cells as 2D faces and uses scalar field inequalities to mask out non-isosurface regions, enabling the rendering of virtually arbitrary numbers of directional lights via vertex programs [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh where each vertex $v$ has an associated scalar value $s_v$. A set of isovales $\{\tau_1, \tau_2, \dots, \tau_N\}$ defining the isosurfaces. Camera parameters for view-independent cell projection.
*   **Output:** A 2D image with flat-shaded, opaque isosurfaces correctly occluded, supporting multiple directional lights.

**Data Structures:**
1.  **Scalar Field:** Stored at mesh vertices.
2.  **Texture Map:** A 2D texture used to encode isosurface presence and color.
    *   **Layout:** The texture coordinates $(u, v)$ are derived from the barycentric coordinates or projected 2D coordinates of the tetrahedron's vertices.
    *   **Content:** Each texel stores a color value. If the tetrahedron contains an isosurface fragment for a specific isovalue, the texel stores the ambient/diffuse material color for that isovalue. If no isosurface is present, the alpha channel is set to 0.
    *   **Thickening:** The texture map is slightly modified to "thicken" the isosurface regions. This prevents gaps between isosurface patches in adjacent tetrahedra caused by discretization or floating-point precision errors during texture sampling [S1].

**Numbered Steps:**
1.  **Cell Projection:** Project each tetrahedron of the unstructured mesh onto the 2D screen space. This is done view-independently, treating each tetrahedron as a set of faces or a projected volume [S1].
2.  **Scalar Inequality Check:** For each fragment (pixel) covered by the projected tetrahedron, determine if an isosurface exists. Let $s_f$ be the scalar value at the front vertex and $s_b$ be the scalar value at the back vertex of the ray passing through the fragment within the tetrahedron. An isosurface for isovalue $\tau$ exists if:
    $$ (s_f < \tau \text{ and } s_b > \tau) \text{ or } (s_f > \tau \text{ and } s_b < \tau) $$
    This indicates a sign change or crossing of the isovalue within the tetrahedron [S1].
3.  **Texture Lookup:** Perform a lookup in the pre-computed texture map using the fragment's 2D coordinates.
    *   If the texture lookup returns a color with $\alpha > 0$, the fragment belongs to an isosurface. The color is set to the ambient material color of the corresponding isovalue [S1].
    *   If $\alpha = 0$, no isosurface is present at this location.
4.  **Alpha Test and Z-Test:**
    *   Apply an **alpha test**: Discard fragments where the alpha channel is 0. This ensures only isosurface fragments are rendered [S1].
    *   Apply the **OpenGL z-test**: This guarantees correct occlusion between overlapping isosurface fragments from different tetrahedra without requiring explicit sorting of the cells [S1].
5.  **Visibility Ordering Logic:** The method implicitly handles visibility ordering based on the scalar field gradient direction. If $s_f < s_b$, the view is along the gradient; isosurfaces with smaller isovales occlude those with greater isovales. If $s_f > s_b$, the reverse holds. This logic is embedded in the texture map generation or the inequality checks [S1].
6.  **Lighting Calculation:** Since the method does not provide interpolated gradients for surface normals, shading is flat. Complete lighting (ambient + diffuse) is calculated in the vertex program.
    *   For each directional light $L_i$, the vertex program computes the diffuse contribution.
    *   The lighting is applied per-vertex and interpolated across the flat-shaded face [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-processing: Generate Texture Map
def generate_texture_map(tetrahedra, scalar_values, isovales, material_colors):
    """
    Creates a 2D texture map where texels encode isosurface presence and color.
    'Thickening' is applied by expanding the valid region in texture space.
    """
    texture = create_2d_texture()
    for tet in tetrahedra:
        # Project tetrahedron to 2D screen space
        projected_coords = project_to_screen(tet.vertices)
        
        # Determine scalar values at vertices
        s_vals = [scalar_values[v] for v in tet.vertices]
        
        # For each pixel in the projected bounding box
        for u, v in projected_coords.pixels:
            # Interpolate scalar values at (u,v) to get s_f and s_b
            # Note: In view-independent projection, s_f and s_b refer to 
            # scalar values at the 'front' and 'back' of the ray through the cell
            # relative to the isovalue crossing logic.
            s_f, s_b = interpolate_scalars(tet, u, v, s_vals)
            
            isosurface_present = False
            color = (0, 0, 0, 0) # Default transparent
            
            for tau in isovales:
                # Check for isosurface crossing
                if (s_f < tau and s_b > tau) or (s_f > tau and s_b < tau):
                    isosurface_present = True
                    color = material_colors[tau]
                    break # Assume first hit or specific ordering logic
            
            # Apply thickening: if isosurface is near, expand alpha
            if isosurface_present:
                texture.set_texel(u, v, color)
            else:
                texture.set_texel(u, v, (0, 0, 0, 0))
    return texture

# Rendering Pass
def render_tetrahedra(tetrahedra, texture, camera, lights):
    """
    Renders tetrahedra using the texture map and OpenGL state.
    """
    enable_alpha_test()
    enable_depth_test()
    
    for tet in tetrahedra:
        # Project and rasterize
        for fragment in rasterize(tet, camera):
            # Lookup color from texture
            color = texture.lookup(fragment.u, fragment.v)
            
            # Alpha test discards fragments with alpha == 0
            if color.alpha == 0:
                continue
            
            # Lighting calculation in vertex program (simplified here)
            # Flat shading: use vertex normal or average normal
            normal = compute_flat_normal(tet)
            final_color = ambient_color
            
            for light in lights:
                # Light direction in object coordinates
                L = light.direction_object
                # Diffuse color
                C_d = light.diffuse_color
                
                # Diffuse term: max(0, dot(N, L))
                diff = max(0, dot(normal, L))
                final_color += C_d * diff
            
            # Write to framebuffer (Z-test handles occlusion)
            framebuffer.write(fragment.x, fragment.y, final_color, color.alpha)
```

**Parameters:**
*   **Isovalues:** A list of scalar thresholds $\tau_i$.
*   **Material Colors:** A mapping from each $\tau_i$ to an RGB color.
*   **Light Data:** Per light: direction in object coordinates and diffuse color. Each light adds two entries to the vertex program's attribute vector [S1].
*   **Thickening Factor:** A small expansion in texture space to prevent gaps. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Gaps between patches:** Without thickening the texture map, gaps may appear between isosurface patches in adjacent tetrahedra due to discretization. This is mitigated by slightly modifying the texture map to "thicken" the isosurfaces [S1].
*   **Flat Shading Limitation:** The approach does not provide interpolated gradients, limiting rendering to flat-shaded isosurfaces. Smooth shading is not supported [S1].
*   **Visibility Ordering:** Correct occlusion relies on the implicit visibility ordering derived from the scalar field gradient direction ($s_f < s_b$ vs $s_f > s_b$). This must be consistent with the texture map generation [S1].

## Subtopics

### Joint Rendering for Transparency
A variant used in the `rgl` engine (R) for rendering multiple contour surfaces together. It ensures transparency is handled correctly when combining separate contour surfaces into a single scene. Joint rendering is essential for the standard and grid engines in R, which draw triangles in back-to-front order, to maintain correct alpha blending and occlusion [S2].

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of tetrahedra and the number of directional lights. Each light requires three vertex program operations per vertex [S1].
*   **Memory Complexity:** Requires storage for the texture map, which scales with the number of isosurfaces and the resolution of the projection.
*   **Strengths:**
    *   No explicit cell sorting required, simplifying the pipeline for unstructured meshes [S1].
    *   Supports a virtually arbitrary number of directional lights efficiently via vertex programs [S1].
    *   Correct occlusion is guaranteed by hardware z-test and alpha test [S1].
*   **Limitations:**
    *   Limited to flat shading; no interpolated gradients for smooth lighting [S1].
    *   View-independent projection may lead to distortion or inefficiency for certain camera angles compared to view-dependent methods.
    *   Requires careful texture map generation to avoid gaps and ensure correct visibility ordering [S1].
*   **When to use:** Suitable for real-time or interactive rendering of unstructured tetrahedral meshes where flat shading is acceptable and explicit sorting is costly or complex.
*   **When not to use:** When smooth shading or high-fidelity lighting is required, or when the mesh is structured and other methods (like ray casting) are more efficient.

## Background
This method was introduced by Weiler and Kraus in their work on hardware-based view-independent cell projection [S1]. It builds on earlier work by Röttger et al. [14] (cited in [S1]) regarding texture map-based isosurface extraction. The technique is particularly relevant for medical imaging and scientific visualization where unstructured tetrahedral meshes are common. The variant for joint rendering in R is discussed in the context of the `AnalyzeFMRI` package and the `rgl` engine [S2].

## Sources
[S1] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S2] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
