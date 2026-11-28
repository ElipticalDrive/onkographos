# Alpha Texturing (Billboarding)

Also called: Alpha Masking

## Overview
Alpha Texturing (Billboarding) renders complex, high-frequency geometry (such as vegetation) as simple 2D primitives (rectangles) textured with RGBA maps, where the alpha channel defines visibility [S1, book]. It replaces explicit 3D polygonal modeling with 2D proxy geometry and static texture masks [G].

## How it works
**Inputs:** A set of 2D RGBA texture maps; simple geometric primitives (typically quads or rectangles) representing the proxy geometry [S1, book].
**Outputs:** A rasterized image where transparent regions of the texture allow background visibility [G].

**Data Structures:**
*   **RGBA Texture:** A 2D array of pixels, each containing four components: Red, Green, Blue, and Alpha ($A$) [G].
*   **Primitive:** A polygon defined by vertices in screen or object space [G].

**Steps:**
1.  **Primitive Selection:** Define a simple geometric primitive (e.g., a rectangle) that approximates the extent of the complex object [S1, book].
2.  **Texture Application:** Bind an RGBA texture map to the primitive [S1, book].
3.  **Alpha Definition:** Use the alpha channel to define visibility: set $A = 1$ for visible parts (e.g., leaves, branches) and $A = 0$ for transparent parts (e.g., gaps, open space) [S1, book].
4.  **Rasterization:** Render the primitive. The graphics hardware interpolates texture coordinates across the primitive and samples the RGBA values [G].
5.  **Compositing:** Fragments are composited into the frame buffer using hardware-supported alpha blending [G].

**Formulas:**
The final color $C_{final}$ for a fragment is determined by the texture color $C_{tex}$ and alpha $A_{tex}$:
$$ C_{final} = C_{tex} \cdot A_{tex} + C_{bg} \cdot (1 - A_{tex}) $$
where $C_{bg}$ is the background color [G].

**Parameters:**
*   **Alpha Values:** $A = 1$ represents solid/visible parts; $A = 0$ represents transparent/gaps [S1, book].

**Edge Cases:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_billboard(primitive_vertices, texture_map, background_color):
    # Step 1: Define simple geometric primitive (e.g., rectangle)
    # Step 2: Apply RGBA texture map to primitive
    # Step 3: Use alpha channel to define visibility (A=1 for visible, A=0 for transparent)
    for fragment in rasterize(primitive_vertices):
        # Step 4: Graphics hardware interpolates texture coordinates and samples RGBA values
        tex_color, tex_alpha = texture_map.sample(fragment.uv)
        # Step 5: Composite fragment using alpha blending
        final_color = tex_color * tex_alpha + background_color * (1 - tex_alpha)
        return final_color
```

## Subtopics

### [Volumetric Tree Reconstruction via Opacity Estimation and View-Dependent Texturing](17465-volumetric-tree-reconstruction-via-opacity-estimation-and-vi.md)
Reconstructs trees from photographs using view-dependent billboards to preserve high-frequency color details, solving the blurring artifacts of static alpha texturing.

## Operating characteristics
Time and memory complexity, measured performance, and limitations are Not given in the sources.

## Background
Alpha texturing for billboarding was introduced as a common trick in computer graphics to render complex geometry like trees efficiently [S1, book]. It leverages the RGBA texture map's alpha channel to simulate transparency without explicit geometric complexity [S1, book].

## Sources
[S1] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
