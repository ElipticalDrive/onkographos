# HAVS with Dynamic LOD

Also called: Hardware-Assisted Visibility Sorting with Dynamic Level of Detail

## Overview
An algorithm for dynamic Level of Detail (LOD) that builds on [Hardware Assisted Visibility Sorting (HAVS)](2918-hardware-assisted-visibility-sorting-havs.md), separating boundary and internal faces, sorting internal faces by importance, and dynamically adjusting the number of faces drawn based on frame rate [S1, unofficial]. It solves the problem of maintaining interactive frame rates on large tetrahedral datasets by progressively streaming approximations that increase in quality during idle frames [S1, unofficial]. Unlike [Optimized HAVS (Sorted k-buffer)](4735-optimized-havs-sorted-k-buffer.md), which focuses on sorting optimization, this method focuses on adaptive primitive reduction to balance interactivity and visual fidelity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh dataset; target frame rate (typically 10 fps) [S1, unofficial].
*   **Output:** A rendered image with adaptive LOD; updated count of internal faces $|I|$ for the next frame [S1, unofficial].

**Data Structures:**
1.  **Boundary Face List:** Contains all boundary faces of the mesh.
2.  **Internal Face List:** Contains internal faces, sorted by importance (most to least). Duplicates (neighboring faces with same vertices and scalar values) are removed to avoid zero-length intervals [S1, unofficial].
3.  **k-buffer:** A fixed-size buffer per pixel storing the $k$ fragments closest to the viewpoint, implemented via fragment shaders [S1, unofficial].
4.  **Lookup Tables:**
    *   **Primary LUT:** Pre-integrated volume integral based on maximum edge length of the mesh. Used for full-quality rendering [S1, unofficial].
    *   **Secondary LUT:** Scaled to handle rays spanning the entire bounding box of the mesh. Used during dynamic LOD to handle gaps caused by removed samples [S1, unofficial].

**Steps:**
1.  **Preprocessing:**
    *   Separate boundary faces from internal faces into distinct lists [S1, unofficial].
    *   Remove duplicate internal faces (neighboring faces with identical vertices and scalar values) to prevent zero-length intervals [S1, unofficial].
    *   Reorder the internal face list from most important to least important using a sampling strategy (Field, Area, Topology, or View-aligned) [S1, unofficial].
2.  **Frame Loop:**
    *   **Measure Render Time:** Track the time taken to render the previous frame [S1, unofficial].
    *   **Adjust LOD:** Dynamically adjust the number of internal faces $|I|$ to be sent to the rasterizer based on the measured render time and the target frame rate (e.g., 10 fps) [S1, unofficial].
    *   **Select Faces:** Pass the first $|I|$ faces from the sorted internal list and all boundary faces to the volume renderer [S1, unofficial].
    *   **Object-Space Sort:** Sort the selected faces by their centroids on the CPU to provide a partial object-space order [S1, unofficial].
    *   **Rasterization & Image-Space Sort:**
        *   Rasterize the faces.
        *   For each pixel, the k-buffer keeps a fixed number of fragments $k$.
        *   When a new fragment arrives, compare it with existing k-buffer entries.
        *   Select the two entries closest to the viewpoint for front-to-back compositing [S1, unofficial].
    *   **Compositing:**
        *   Use the appropriate Lookup Table (Secondary LUT for dynamic LOD, Primary LUT for full quality) to find color and opacity based on front fragment scalar, back fragment scalar, and distance [S1, unofficial].
        *   Composite color and opacity into the framebuffer.
        *   Write remaining fragments back to the k-buffer [S1, unofficial].
    *   **Quality Switch:** If the user is not interacting (idle frames), replace the Secondary LUT with the Primary LUT and use the entire internal face list for full-quality rendering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def havs_dynamic_lod(mesh, target_fps=10):
    # Preprocessing
    boundary_faces = extract_boundary_faces(mesh)
    internal_faces = extract_internal_faces(mesh)
    
    # Remove duplicates to avoid zero-length intervals
    internal_faces = remove_duplicate_faces(internal_faces)
    
    # Sort by importance (most to least)
    internal_faces = sort_by_importance(internal_faces, strategy='field')
    
    # Initialize LUTs
    primary_lut = create_preintegrated_lut(max_edge_length(mesh))
    secondary_lut = create_scaled_lut(bounding_box(mesh))
    
    current_lut = secondary_lut
    num_internal_faces = len(internal_faces)
    k_buffer = initialize_k_buffer(k=2)
    
    while rendering:
        start_time = get_current_time()
        
        # Select faces
        selected_faces = boundary_faces + internal_faces[:num_internal_faces]
        
        # Object-space sort
        sorted_faces = sort_by_centroid(selected_faces)
        
        # Rasterize and k-buffer sort
        for face in sorted_faces:
            fragments = rasterize(face)
            for frag in fragments:
                k_buffer.update(frag)
                
        # Composite
        for pixel in framebuffer:
            if k_buffer.has_entries(pixel):
                front_frag, back_frag = k_buffer.get_closest(pixel)
                color, opacity = composite(front_frag, back_frag, current_lut)
                framebuffer.write(pixel, color, opacity)
                
        end_time = get_current_time()
        render_time = end_time - start_time
        
        # Adjust LOD based on frame rate
        current_fps = 1.0 / render_time
        if current_fps < target_fps:
            # Reduce LOD if too slow
            num_internal_faces = max(0, num_internal_faces - step_size)
            current_lut = secondary_lut
        else:
            # Increase LOD if fast enough
            num_internal_faces = min(len(internal_faces), num_internal_faces + step_size)
            # Use primary LUT only if full quality is desired (e.g., idle)
            if is_idle():
                current_lut = primary_lut
            else:
                current_lut = secondary_lut
                
        yield framebuffer
```

**Parameters:**
*   **Target Frame Rate:** 10 frames per second [S1, unofficial].
*   **k:** Fixed number of fragments per pixel in the k-buffer (typically 2 for front-to-back compositing) [S1, unofficial].

**Pitfalls:**
*   **Gaps in Lookup:** Removing samples introduces intervals larger than the maximum edge length used to build the pre-integrated lookup table, causing gaps. This is solved by using a secondary LUT scaled to the entire bounding box during dynamic LOD [S1, unofficial].
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.

## Subtopics

### [HAVS-based Progressive Rendering](7395-havs-based-progressive-rendering.md)
A level-of-detail framework for interactive exploration of large tetrahedral datasets. It incrementally streams portions of the mesh, displaying approximations that increase in quality until full quality is achieved. It builds on HAVS with Dynamic LOD by separating boundary and internal faces and sorting them by importance.

## Operating characteristics
*   **Time Complexity:** Depends on the number of faces $|I|$ and the k-buffer size $k$. Rasterization is the bottleneck [S1, unofficial].
*   **Memory Complexity:** GPU memory stores the mesh faces and the k-buffer. The set of all possible mesh faces does not change with viewpoint, allowing them to remain in GPU memory for efficiency [S1, unofficial].
*   **Measured Performance:** Tested on a PC with a 3.2 GHz Pentium 4, 2 GB RAM, and ATI Radeon X800 Pro (256 MB RAM) using Spx2, Torso, and Fighter datasets [S1, unofficial].
*   **Strengths:** Fewer rasterizations than cell-based methods (2*n* triangles for n tetrahedra vs 3.4*n* for projected tetrahedra). Avoids empty space accumulation errors when dropping primitives. Works with perspective or parallel projections without additional computation [S1, unofficial].
*   **Limitations:** Approximate rendering may not match exact rendering visually, though subjective impression is that it is accurate enough for interaction [S1, unofficial].
*   **When to use:** Interactive exploration of large tetrahedral datasets where frame rate is critical.
*   **When not to use:** When full visual accuracy is required without approximation.

## Background
Originated in Steve Callahan's MS thesis, "The k-Buffer and Its Applications to Volume Rendering" (2006), which proposed HAVS and its dynamic LOD variant for efficient volume rendering on hardware [S1, unofficial]. It is used in visualization systems requiring interactive exploration of large volumetric datasets.

## Sources
[S1] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
