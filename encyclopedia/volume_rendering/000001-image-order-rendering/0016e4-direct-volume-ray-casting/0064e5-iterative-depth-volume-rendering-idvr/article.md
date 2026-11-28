# Iterative Depth Volume Rendering (IDVR)

Also called: IDVR with Skip-Step Optimization, IDVR design

## Overview
Iterative Depth Volume Rendering (IDVR) is an interactive volume rendering technique designed to expose deeper internal structures of a volume (specifically the brain) by removing the tissue in front of a clipping surface while retaining anatomical context [S1]. Unlike standard direct volume ray casting which composites all samples along a ray, IDVR iteratively searches for the closest valid voxel (V) or depth voxel (Vd) along view rays to determine depth and opacity, allowing observation of subcortical regions and white matter that surface-based visualizations miss [S1]. It sets itself apart by using a specific iterative skip-step optimization to manage the performance cost of searching for these closest valid voxels without early termination [S1].

## How it works
IDVR operates on the principle of removing the brain volume in front of a clipping surface while rendering anatomical maps and volume change data completely [S1]. The core mechanism involves an iterative search for the Point of View/Projection (PV), defined as the closest valid voxel (V) or depth voxel (Vd) with the highest lighting/alpha value ($l_a$) along the view ray [S1].

**Inputs and Outputs:**
*   **Input:** Volume data (texture $T_V$), clipping surface definition, view ray parameters, skip-step parameter $i$.
*   **Output:** Final pixel color and depth, with the front-most brain tissue removed to reveal internal structures [S1].

**Data Structures:**
*   **Texture $T_V$:** The volume data texture. The resolution is assumed to be sufficiently large for the skip-step optimization to be effective [S1].
*   **Skip-step parameter $i$:** An integer controlling the accuracy and performance of the iteration [S1].

**Algorithm Steps:**
1.  **Ray Initialization:** For each pixel, define the view ray intersecting the volume bounding box [G].
2.  **Iterative Search:** Iterate over neighboring texels along the ray to seek the PV (Point of View/Projection) [S1].
3.  **Skip-Step Optimization:** To improve performance, skip every $i$-th texel so it is not tested as a candidate for V or Vd [S1].
    *   If $i=1$ (High Accuracy): Test every texel.
    *   If $i=2$ (Balanced): Skip every 2nd texel.
    *   If $i=3$ (Low Accuracy): Skip every 3rd texel.
4.  **Validity Check:** For each tested texel, determine if it is a valid voxel (V) or depth voxel (Vd) and calculate its lighting/alpha value ($l_a$) [S1].
5.  **No Early Termination:** The iteration cannot be early-terminated because there is always a possibility of finding a closer V or Vd with a higher $l_a$ value further along the ray [S1].
6.  **Compositing:** Render the anatomical regions in front of the clipping surface and the surface texture to provide localization, blending their colors with the volume change data [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def idvr_render(ray, volume_texture, clipping_surface, skip_step_i):
    """
    Renders a single ray using Iterative Depth Volume Rendering.
    
    Args:
        ray: Ray object with origin and direction.
        volume_texture: The 3D volume texture T_V.
        clipping_surface: Definition of the clipping plane/surface.
        skip_step_i: Integer skip-step parameter (1, 2, or 3).
        
    Returns:
        color: Final pixel color.
        depth: Depth of the closest valid voxel found.
    """
    # Intersect ray with volume bounding box to get t_near and t_far
    t_start, t_end = ray.intersect_volume_bbox()
    
    # Initialize variables to track the best candidate found so far
    best_voxel = None
    best_la = -1.0  # Lighting/alpha value, initialized to lowest possible
    best_depth = float('inf')
    
    # Step size for iteration. In a real implementation, this would be 
    # derived from the texture resolution and ray length.
    # Here we assume a discrete iteration over texels along the ray.
    # The sources do not specify the exact sampling interval, only the skip logic.
    step_size = calculate_step_size(ray, volume_texture.resolution)
    
    t = t_start
    texel_index = 0
    
    while t < t_end:
        # Calculate sample position
        sample_pos = ray.origin + ray.direction * t
        
        # Check if this texel should be skipped based on skip_step_i
        if texel_index % skip_step_i != 0:
            # Sample volume value at sample_pos
            v_value = volume_texture.sample(sample_pos)
            
            # Determine if this is a valid voxel (V) or depth voxel (Vd)
            # This depends on the specific volume data and clipping surface.
            # The source implies V and Vd are distinct categories of voxels.
            is_valid = is_valid_voxel(v_value, clipping_surface)
            
            if is_valid:
                # Calculate lighting/alpha value (l_a)
                # The exact formula for l_a is not provided in the sources [G].
                # It likely involves transfer function mapping and shading.
                l_a = compute_lighting_alpha(v_value, sample_pos)
                
                # Update best candidate if this one is closer and has higher l_a
                # Note: "Closer" means smaller t (or distance from origin)
                # "Higher l_a" means better visibility/opacity
                if t < best_depth and l_a > best_la:
                    best_depth = t
                    best_la = l_a
                    best_voxel = v_value
        
        # Advance along the ray
        t += step_size
        texel_index += 1
        
    # If a valid voxel was found, render it with anatomical context
    if best_voxel is not None:
        # Render anatomical regions and volume change data
        # Blending colors as described in the source [S1]
        color = render_anatomical_context(best_voxel, best_depth)
        depth = best_depth
    else:
        # Fallback if no valid voxel found (e.g., behind clipping surface or empty)
        color = background_color
        depth = t_end
        
    return color, depth

def is_valid_voxel(v_value, clipping_surface):
    """
    Determines if a voxel is valid (V) or a depth voxel (Vd) relative to the clipping surface.
    The exact criteria are not specified in the sources [G].
    """
    # Placeholder for the specific validity logic defined by the application
    # e.g., checking if the voxel is inside the brain but behind the clipping plane
    return True 

def compute_lighting_alpha(v_value, sample_pos):
    """
    Computes the lighting/alpha value (l_a) for a voxel.
    The exact formula is not provided in the sources [G].
    """
    # Placeholder for transfer function and shading calculation
    return 0.5
```

**Parameters:**
*   **$i$ (Skip-step parameter):** Controls the trade-off between accuracy and performance [S1].
    *   $i=1$: High Accuracy. Tests every texel.
    *   $i=2$: Balanced. Skips every 2nd texel. Nearly no visual artifacts [S1].
    *   $i=3$: Low Accuracy. Skips every 3rd texel. Noticeable artifacts (loose wrong depth values) but reasonable performance [S1].

**Edge Cases and Pitfalls:**
*   **Visual Artifacts:** When $i=3$, loose wrong depth values may appear due to skipped texels potentially being the true closest valid voxel [S1]. However, these artifacts do not disturb overall depth perception in the cutout [S1].
*   **Performance Limitation:** The iteration cannot be early-terminated because a closer valid voxel with higher $l_a$ might exist further along the ray [S1]. This makes the method computationally expensive compared to standard ray casting with early termination [S1].

## Operating characteristics
*   **Time Complexity:** The method requires iterating over all neighboring texels along the ray without early termination, making it $O(N)$ where $N$ is the number of samples along the ray, scaled by the inverse of the skip-step parameter $i$ [S1].
*   **Memory Complexity:** Depends on the volume texture size and the storage for the best candidate per ray [G].
*   **Measured Performance:**
    *   **Nvidia GeForce GTX 480 (Mono view, 1400x1050):**
        *   $i=1$: 55 fps
        *   $i=2$: 11 fps
        *   $i=3$: 18 fps [S1]
    *   **Nvidia Quadro 6000 (Stereo view, 1400x1050):**
        *   $i=1$: 4 fps
        *   $i=2$: 3 fps
        *   $i=3$: 10 fps [S1]
*   **Strengths:** Allows observation of data inside the brain, including subcortical regions and white matter, which surface-based visualization misses [S1]. Provides good spatial perception and anatomical localization through blending of colors [S1].
*   **Limitations:** Performance is limited by the iteration over all neighboring texels, which cannot be early-terminated [S1]. Frame rates are lower than common volume rendering, especially in stereo mode [S1].
*   **When to use:** When interactive visualization of internal brain structures (e.g., in neurodegenerative disease studies) is required, and the user needs to see past a clipping surface while retaining anatomical context [S1].
*   **When not to use:** When high performance is critical and visual artifacts from skipping texels are unacceptable, or when the volume data does not require internal structure visualization behind a clipping plane [S1].

## Background
IDVR was developed for the interactive 3D visualization of structural changes in the brain of a person with corticobasal syndrome (CBS) [S1]. It was introduced to allow scientists to observe data not only on the surface but also inside the brain, providing a better impression of the spatial extent of regional volume loss in the context of individual brain anatomy [S1]. The work was funded by the Helmholtz Portfolio Theme “Supercomputing and Modeling for the Human Brain” and partially by the “Human Brain Project” [S1].

## Sources
[S1] Interactive 3D visualization of structural changes in the brain of a person with corticobasal syndrome., https://archive.org/details/pubmed-PMC4017140, fetched 2026-10-28
