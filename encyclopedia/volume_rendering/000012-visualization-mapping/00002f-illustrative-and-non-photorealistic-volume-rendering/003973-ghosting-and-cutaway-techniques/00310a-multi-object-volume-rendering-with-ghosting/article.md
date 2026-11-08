# Multi-object volume rendering with ghosting

Also called: Direct multi-object volume visualization, Ghosting, Multi-Pass Rendering Implementation, VolumeShop multi-object rendering

## Overview
Multi-object volume rendering with ghosting is a technique for visualizing inter-penetrating volumetric objects by decomposing a scalar field into three distinct logical objects: **Selection** (the primary focus), **Ghost** (the original position of the selection), and **Background** (the occluding environment) [S1]. It solves the problem of spatial ambiguity in complex, overlapping structures by allowing independent control of appearance and spatial transformation for each object [S1]. Unlike standard volume rendering, it distinguishes itself by using a three-pass rendering strategy with depth-buffer-based occlusion culling to efficiently handle intersections and apply "ghosting" (selective opacity reduction) or "cutaways" (complete suppression) to the background, preserving edge cues and spatial context [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   $f_V$: Scalar-valued volumetric function defined for every point $p$ in space (e.g., CT values) [S1].
*   $f_S$: Selection function defined for every point $p$ in space, with values in $[0, 1]$, where $0$ means 'not selected' and $1$ means 'fully selected' [S1].
*   $T$: Transformation applied to the selection object (move, rotate, scale) [S1].
*   $c_S, \alpha_S$: Color and opacity transfer functions for the selection, based on $f_V(T(p))$ [S1].
*   $c_G, \alpha_G$: Color and opacity transfer functions for the ghost, based on $f_V(p)$ [S1].
*   $c_B, \alpha_B$: Color and opacity transfer functions for the background, based on $f_V(p)$ [S1]
*   Ghosting factor: User-defined value controlling the degree of ghosting, ranging from no ghosting (opacity not reduced) to full cutaway (occluding structures suppressed) [S1].
*   Optional: Depth map of opaque geometry intersections [S1].

**Outputs:**
*   Final color and opacity image with three logical objects composited, respecting intersections and ghosting/cutaway rules [S1].

### Data Structures
*   **Bricked Memory Layout:** Data volumes and selection volumes are stored in a bricked memory layout using reference counting. Volumes are subdivided into small cubes accessed via an index data structure. Redundant bricks are stored only once [S1].
*   **Copy-on-Write:** Used for handling modifications to shared bricks, particularly for the sparse selection volume, to avoid corrupting other references [S1].
*   **Meta Information:** Min-max octrees, bounding boxes, and transformations are stored for both volumes and updated on modification to allow quick extraction of tight bounding volumes [S1].
*   **Depth Textures:** Two depth maps are generated for the ghost and selection bounding volumes, and optionally one for opaque geometry [S1].

### Mathematical Formulation
The method defines three fuzzy selection sets ($SS, SG, SB$) and three volume sets ($VS, VG, VB$) based on membership functions $\mu$ [S1]:

$$
\begin{aligned}
\mu_{SS}(p) &= f_S(T(p)) \\
\mu_{SG}(p) &= f_S(p) \\
\mu_{SB}(p) &= 1 - f_S(p)
\end{aligned}
$$

$$
\begin{aligned}
\mu_{VS}(p) &= \alpha_S(f_V(T(p))) \\
\mu_{VG}(p) &= \alpha_G(f_V(p)) \\
\mu_{VB}(p) &= \alpha_B(f_V(p))
\end{aligned}
$$

The final object sets ($S, G, B$) are the intersections of the corresponding selection and volume sets [S1]:

$$
\begin{aligned}
S &= SS \cap VS \\
G &= SG \cap VG \\
B &= SB \cap VB
\end{aligned}
$$

The final opacity $\alpha(p)$ at point $p$ is defined as the grade of membership in the union of all object sets [S1]:

$$
\alpha(p) = \mu_{S \cup G \cup B}(p)
$$

### Rendering Steps
The implementation uses three volume rendering passes with increasing complexity to avoid the computational overhead of single-pass multi-object compositing and dynamic branching [S1].

**Preprocessing:**
1.  Extract bounding volumes for background, ghost, and selection by traversing hierarchical data structures [S1].
2.  Render the bounding volumes of the ghost and selection into separate depth textures with depth test set to `LESS` [S1].
3.  Filter these depth maps with a large kernel to enable smooth cutaways [S1].
4.  (Optional) If opaque geometry intersections are present, generate an additional depth map before the background pass and blend geometry color contributions into the frame buffer [S1].

**Pass 1: Background**
1.  Set depth test to `LESS` and render the bounds of the background object into the depth buffer [S1].
2.  Disable depth buffer writes.
3.  Set depth test to `GREATER`. This skips empty space up to the first intersection with the background bounding volume (early-z culling) [S1].
4.  Render view-aligned slices in back-to-front order.
5.  In the fragment program, shade the background object [S1].

**Pass 2: Ghost**
1.  Clear the depth buffer.
2.  Render the bounding volume of the ghost object with depth test set to `GREATER` [S1].
3.  Disable depth buffer writes.
4.  Set depth test to `LESS`.
5.  In the fragment program:
    *   Perform shading for both background and ghost objects [S1].
    *   Discard fragments whose depth value is greater than or equal to the corresponding value in the **selection depth map** (effectively cutting out regions containing the selection from the background/ghost view) [S1].
    *   If cutaways are enabled, modulate the background opacity by the user-defined **ghosting factor** for fragments whose depth value is greater than or equal to the corresponding value in the **ghost depth map** [S1].

**Pass 3: Selection**
1.  Render the selection bounds into the cleared depth buffer with depth test set to `GREATER` [S1].
2.  Disable depth buffer writes.
3.  Set depth test to `LESS`.
4.  Pass two sets of texture coordinates to the fragment program:
    *   Unmodified coordinates for background and ghost data access.
    *   Transformed coordinates (using $T$) for selection data access [S1].
5.  In the fragment program:
    *   Perform shading for background, selection, and ghost objects [S1].
    *   Handle intersections (background/selection and ghost/selection) using intersection transfer functions [S1].
    *   For cutaways, modulate the background's opacity if the fragment's depth value is greater than or equal to the corresponding values in the ghost or selection depth maps [S1].

**Visual Conventions:**
*   Display auxiliary geometry (boxes, arrows, fans) as 3D objects with correct intersection handling or as overlays [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_multi_object_ghosting(volume_data, selection_data, transformation_T, 
                                 transfer_functions, ghosting_factor, depth_maps):
    """
    volume_data: Scalar field f_V
    selection_data: Selection field f_S
    transformation_T: Transformation matrix/function T
    transfer_functions: Dict with keys 'S', 'G', 'B' containing (color_func, opacity_func)
    ghosting_factor: Float [0, 1]
    depth_maps: Dict with keys 'ghost', 'selection', 'geometry' (optional)
    """
    
    # Preprocessing: Extract bounding volumes
    bounds = extract_bounding_volumes(volume_data, selection_data)
    
    # Generate depth maps for occlusion culling
    # Render ghost bounds with depth test LESS
    depth_map_ghost = render_depth(bounds['ghost'], depth_test='LESS')
    # Filter depth map for smooth cutaways
    depth_map_ghost_smooth = filter_kernel(depth_map_ghost, kernel_size='large')
    
    # Render selection bounds with depth test LESS
    depth_map_selection = render_depth(bounds['selection'], depth_test='LESS')
    depth_map_selection_smooth = filter_kernel(depth_map_selection, kernel_size='large')
    
    # Optional: Handle opaque geometry
    if 'geometry' in depth_maps:
        depth_map_geometry = depth_maps['geometry']
        # Blend geometry color into frame buffer (assumed handled by renderer state)
    else:
        depth_map_geometry = None

    # Pass 1: Background
    # Set depth test LESS, render background bounds to depth buffer
    render_depth(bounds['background'], depth_test='LESS')
    
    # Disable depth writes, set depth test GREATER
    set_depth_write(False)
    set_depth_test('GREATER')
    
    # Render background slices back-to-front
    render_slices(volume_data, bounds['background'], 
                  color_func=transfer_functions['B'].color, 
                  opacity_func=transfer_functions['B'].opacity,
                  texture_coords='unmodified')

    # Pass 2: Ghost
    clear_depth_buffer()
    
    # Render ghost bounds with depth test GREATER
    render_depth(bounds['ghost'], depth_test='GREATER')
    
    # Disable depth writes, set depth test LESS
    set_depth_write(False)
    set_depth_test('LESS')
    
    # Render ghost slices
    # Fragment program logic for Ghost Pass:
    # 1. Shade background and ghost
    # 2. Discard if depth >= depth_map_selection
    # 3. If cutaways: modulate background opacity by ghosting_factor if depth >= depth_map_ghost_smooth
    
    render_slices(volume_data, bounds['ghost'], 
                  color_func=transfer_functions['G'].color, 
                  opacity_func=transfer_functions['G'].opacity,
                  texture_coords='unmodified',
                  discard_condition=lambda depth: depth >= depth_map_selection[depth],
                  modulate_background_opacity=lambda depth: 
                      ghosting_factor if depth >= depth_map_ghost_smooth[depth] else 1.0)

    # Pass 3: Selection
    clear_depth_buffer()
    
    # Render selection bounds with depth test GREATER
    render_depth(bounds['selection'], depth_test='GREATER')
    
    # Disable depth writes, set depth test LESS
    set_depth_write(False)
    set_depth_test('LESS')
    
    # Render selection slices
    # Fragment program logic for Selection Pass:
    # 1. Shade background, selection, and ghost
    # 2. Handle intersections using intersection transfer functions
    # 3. If cutaways: modulate background opacity if depth >= depth_map_ghost_smooth or depth_map_selection_smooth
    
    render_slices(volume_data, bounds['selection'], 
                  color_func=transfer_functions['S'].color, 
                  opacity_func=transfer_functions['S'].opacity,
                  texture_coords='transformed', # Uses T
                  additional_texture_coords='unmodified', # For background/ghost access
                  intersection_handling=compute_intersection_transfer,
                  modulate_background_opacity=lambda depth: 
                      apply_ghosting(ghosting_factor, depth, depth_map_ghost_smooth, depth_map_selection_smooth))

    # Render visual conventions (boxes, arrows, fans)
    render_visual_conventions(bounds, transformation_T)

def compute_intersection_transfer(p, selection_val, ghost_val, background_val):
    """
    Handles intersections between S, G, and B using specific transfer functions.
    Note: B and G never intersect by definition (mu_SB and mu_SG are mutually exclusive).
    Intersections occur between S and G, or S and B.
    """
    # Implementation depends on specific intersection transfer functions defined by user
    pass

def apply_ghosting(factor, depth, ghost_depth_map, selection_depth_map):
    """
    Modulates background opacity for cutaways.
    """
    if depth >= ghost_depth_map[depth] or depth >= selection_depth_map[depth]:
        return factor
    return 1.0
```

### Edge Cases and Pitfalls
*   **Shared Bricks:** Modifications to shared bricks in the bricked memory layout require copy-on-write to avoid corrupting other references to the same data [S1].
*   **Performance Scaling:** Performance depends heavily on the size of the selection. If no selection is defined, frame rates match conventional slice-based rendering due to early-z culling [S1].
*   **Non-Intersecting Selections:** If the selection does not intersect the background or ghost (checked via bounding box test), a simplified fragment program can be executed in the selection pass [S1].
*   **Ghosting vs. Cutaway:** Ghosting preserves edge cues by reducing opacity, while cutaways remove occluding regions completely. The ghosting factor smoothly interpolates between these states [S1].

## Operating characteristics
*   **Time Complexity:** The method uses three rendering passes. Complexity is dominated by the selection size. If the selection is small, performance is close to conventional slice-based rendering due to early-z culling [S1].
*   **Memory Complexity:** Uses a bricked memory layout with reference counting and copy-on-write, which is efficient for sparse selection volumes [S1]. Requires storage for two additional depth textures (ghost and selection) and their filtered versions [S1].
*   **Measured Performance:** On a standard UNC CT head dataset ($256^3$), rendering $3 \cdot 256 \approx 444$ slices took approximately 2 seconds on the hardware used in the source study [S1].
*   **Strengths:** Efficient handling of inter-penetrating objects; preserves spatial context through ghosting; allows smooth control between ghosting and cutaway; leverages hardware early-z culling for performance [S1].
*   **Limitations:** Performance degrades with large selection sizes; requires careful management of depth maps for smooth cutaways; complex fragment program logic in the selection pass [S1].
*   **When to use:** When visualizing inter-penetrating volumetric objects where spatial context and edge cues of occluding structures are important [S1].
*   **When not to use:** When simple, fast visualization of non-intersecting or fully opaque objects is sufficient, or when hardware does not support efficient depth buffering and early-z culling [S1].

## Background
*   **Origin:** The method was introduced in the VolumeShop system, an interactive system for direct volume illustration [S1].
*   **History:** Developed as part of the VolumeShop project, which aimed to provide illustrative enhancement methods and selective illustration techniques for volumetric data [S1].
*   **Usage:** Used in medical visualization and scientific data exploration where understanding the spatial relationship between overlapping structures is critical [S1].

## Sources
[S1] VolumeShop: An Interactive System for Direct Volume Illustration, https://doi.org/10.1109/vis.2005.135, fetched 2026-10-26
