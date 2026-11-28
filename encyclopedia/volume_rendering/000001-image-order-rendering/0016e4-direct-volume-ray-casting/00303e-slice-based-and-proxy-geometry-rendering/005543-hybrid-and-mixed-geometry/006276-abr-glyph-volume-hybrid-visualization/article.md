# ABR Glyph-Volume Hybrid Visualization

Also called: Artifact-Based Rendering (ABR) for Brain Microstructure, Magic Lens Glyph-Volume Rendering

## Overview
Artifact-Based Rendering (ABR) is a hybrid visualization technique for Diffusion-Tensor MRI (DT-MRI) data that combines volume rendering for anatomical context with sparsely sampled, texture-encoded glyphs and oriented ellipsoids to represent crossing fibers and microstructural parameters [S1]. It solves the problem of occlusion in 3D representations of complex white matter by using a "magic lens" design: volume rendering and sparse glyphs provide global context, while high-resolution glyph data is presented in an interactively defined small volume [S1]. It differs from prior slice-based approaches by encoding fiber orientation via glyph texture variations instead of color, freeing the color channel to encode a derived variable (similarity between primary and secondary directions) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D Diffusion-Tensor MRI volume data; user-defined interaction region (magic lens); parameters for axonal density and axonal radius.
*   **Output:** Composite image containing volume-rendered anatomical context, sparse oriented glyphs for crossing fibers, and oriented ellipsoids for leftover diffusion.

**Data Structures:**
*   **Voxel Data:** Each voxel contains diffusion tensor information, from which primary and secondary fiber orientations are derived [S1].
*   **Glyph Profile:** A 2D texture or geometry profile shared by all glyphs. The profile has a linear texture whose density varies along its length [S1].
*   **Ellipsoid Geometry:** An oriented ellipsoid geometry per voxel, scaled and rotated to depict leftover diffusion [S1].

**Steps:**

1.  **Filter Voxels:** Identify voxels containing crossing fibers. Only these voxels are processed for glyph rendering [S1].
2.  **Compute Orientations:** For each selected voxel, compute the primary and secondary fiber orientations from the diffusion tensor data [S1].
3.  **Compute Derived Variable:** Calculate the similarity between the primary and secondary directions [S1].
4.  **Configure Glyphs:**
    *   Scale the glyph size based on the **axonal radius parameter** [S1].
    *   Adjust the linear texture density along the glyph length based on the **axonal density variable** [S1].
    *   Orient the glyph to align with the primary and secondary fiber orientations [S1].
    *   Map the derived similarity variable to color, using a red color range to highlight nearly perpendicular crossings [S1].
5.  **Configure Ellipsoids:** For each selected voxel, generate an oriented ellipsoid that depicts the "leftover" diffusion after accounting for the two primary fiber orientations [S1].
6.  **Render Context:** Render the full volume using standard volume rendering techniques to provide anatomical context [S1].
7.  **Render Magic Lens:**
    *   Define an interactively specified small volume (the magic lens) [S1].
    *   Within this region, render the configured glyphs and ellipsoids at high resolution [S1].
    *   Outside this region, render only the sparse glyphs for context [S1].
8.  **Composite:** Combine the volume-rendered context, the sparse context glyphs, and the high-resolution magic lens content into the final image [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_abr_glyph_volume(volume_data, camera, magic_lens_region, axonal_density_var, axonal_radius_param):
    """
    Renders ABR Glyph-Volume Hybrid Visualization.
    
    Args:
        volume_data: 3D array of diffusion tensors.
        camera: Camera position and orientation.
        magic_lens_region: Bounding box or sphere defining the high-res interaction volume.
        axonal_density_var: Scalar controlling glyph texture density.
        axonal_radius_param: Scalar controlling glyph size.
    """
    # 1. Render Volume Context
    # Standard volume rendering for anatomical context
    context_image = volume_render(volume_data, camera)
    
    # 2. Identify Crossing Fiber Voxels
    crossing_voxels = []
    for voxel in volume_data:
        if has_crossing_fibers(voxel):
            crossing_voxels.append(voxel)
            
    # 3. Prepare Glyphs and Ellipsoids
    glyphs = []
    ellipsoids = []
    
    for voxel in crossing_voxels:
        primary_dir, secondary_dir = compute_primary_secondary_orientations(voxel)
        similarity = compute_similarity(primary_dir, secondary_dir)
        
        # Configure Glyph
        glyph = create_glyph(
            profile=get_shared_glyph_profile(),
            size=axonal_radius_param,
            texture_density=axonal_density_var,
            orientation=primary_dir, # Simplified; actual implementation may use both dirs
            color=map_similarity_to_color(similarity) # Red for perpendicular
        )
        
        # Configure Ellipsoid for leftover diffusion
        ellipsoid = create_oriented_ellipsoid(
            voxel.tensor, 
            primary_dir, 
            secondary_dir
        )
        
        glyphs.append(glyph)
        ellipsoids.append(ellipsoid)
        
    # 4. Render Magic Lens (High-Res)
    # Render glyphs and ellipsoids inside the magic lens region
    lens_content = render_empty_image()
    for glyph in glyphs:
        if is_inside(glyph.position, magic_lens_region):
            render_glyph(lens_content, glyph)
    for ellipsoid in ellipsoids:
        if is_inside(ellipsoid.position, magic_lens_region):
            render_ellipsoid(lens_content, ellipsoid)
            
    # 5. Render Sparse Context Glyphs (Outside Magic Lens)
    # Render only glyphs outside the magic lens to provide context
    context_glyphs = render_empty_image()
    for glyph in glyphs:
        if not is_inside(glyph.position, magic_lens_region):
            render_glyph(context_glyphs, glyph)
            
    # 6. Composite
    # Combine volume context, sparse context glyphs, and high-res lens content
    final_image = alpha_blend(context_image, context_glyphs)
    final_image = alpha_blend(final_image, lens_content)
    
    return final_image

def has_crossing_fibers(voxel):
    # [G] Implementation detail: Check eigenvalues or angular separation of eigenvectors
    # to determine if fibers cross. Not specified in sources.
    return True # Placeholder for actual DT-MRI crossing detection logic

def compute_primary_secondary_orientations(voxel):
    # [G] Extract dominant eigenvectors from the diffusion tensor
    # Not specified in sources.
    return (Vector(1,0,0), Vector(0,1,0)) # Placeholder

def compute_similarity(dir1, dir2):
    # [G] Compute angular similarity or dot product
    # Not specified in sources.
    return 0.5 # Placeholder

def map_similarity_to_color(similarity):
    # Red color range for nearly perpendicular crossings
    # [G] Specific color mapping function not detailed in sources
    return Color(1, 0, 0) # Placeholder for red

def create_glyph(profile, size, texture_density, orientation, color):
    # [G] Create geometry/texture for the glyph
    # Not specified in sources.
    return {"profile": profile, "size": size, "texture_density": texture_density, "orientation": orientation, "color": color}

def create_oriented_ellipsoid(tensor, primary, secondary):
    # [G] Create ellipsoid geometry representing leftover diffusion
    # Not specified in sources.
    return {"position": Vector(0,0,0), "orientation": primary} # Placeholder

def render_glyph(image, glyph):
    # [G] Rasterize glyph onto image
    pass

def render_ellipsoid(image, ellipsoid):
    # [G] Rasterize ellipsoid onto image
    pass

def is_inside(position, region):
    # [G] Check if position is within magic lens region
    return True # Placeholder
```

**Parameters:**
*   **axonal density variable:** Controls the density of the linear texture along the length of each glyph [S1].
*   **axonal radius parameter:** Controls the size of each glyph [S1].
*   **similarity between primary and secondary directions:** Derived variable encoded via color mapping to highlight nearly perpendicular crossings (red color range) [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion:** Severe occlusion degrades 3D representations of complex white matter. Mitigated by using the magic lens design to isolate high-resolution data in an interactively defined small volume while glyphs provide surrounding context [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:**
    *   Provides a natural, organic visual language [S1].
    *   Allows visualization of structure not possible in 2D slice-based visualizations, particularly in VR [S1].
    *   Frees the color channel by encoding fiber orientation in texture, allowing color to encode a new derived variable (direction similarity) [S1].
*   **Limitations:**
    *   Occlusion is a major factor in 3D representation design [S1].
    *   Relies on user interaction (magic lens) to manage complexity [S1].
*   **When to use:**
    *   Visualizing brain microstructure (Diffusion-Tensor MRI) in VR, specifically regions with high fiber crossings like the centrum semiovale [S1].
*   **When not to use:**
    *   Not given in the sources.

## Background
*   **Origin:** Artifact-Based Rendering (ABR) was introduced in the paper "Artifact-Based Rendering: Harnessing Natural and Traditional Visual Media for More Expressive and Engaging 3D Visualizations" [S1].
*   **History and Development:** Prior to ABR, DT-MRI data had only been visualized using slice-based approaches [S1]. A straightforward translation of slice-based visualizations to 3D failed due to occlusion [S1]. ABR was developed to address this by combining volume rendering with sparsely sampled glyphs and oriented ellipsoids [S1]. The technique was applied to the centrum semiovale region, a region with high fiber crossings [S1]. The collaborating domain scientist confirmed expected brain structure and noted that the 3D VR visualization revealed structure not visible in 2D slice-based visualizations [S1].
*   **Usage:** Applied to brain microstructure imaging (Diffusion-Tensor MRI) in VR [S1].

## Sources
[S1] Artifact-Based Rendering: Harnessing Natural and Traditional Visual Media for More Expressive and Engaging 3D Visualizations, https://doi.org/10.1109/tvcg.2019.2934260, fetched 2026-10-26
