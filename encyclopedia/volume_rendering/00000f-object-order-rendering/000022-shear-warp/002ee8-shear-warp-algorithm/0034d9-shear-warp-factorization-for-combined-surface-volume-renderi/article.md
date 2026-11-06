# Shear-warp Factorization for Combined Surface/Volume Rendering

Also called: Fast Surface and Volume Rendering Based on Shear-Warp Factorization, Fast combination of volume-rendered and surface-rendered images, Shear-warp based surgical simulator rendering

## Overview
This method enables real-time visualization in surgical simulators by combining volume-rendered 3D medical images (organs) with surface-rendered medical instruments (e.g., needles) using shear-warp factorization, Z-buffer depth comparison, early ray termination, and instrument-region masking to reduce computation [S1, S2]. It sets itself apart from pure volume rendering by leveraging the speed of surface rendering for simple, frequently moving objects (instruments) while retaining the segmentation-free realism of volume rendering for complex, fuzzy data (organs), using a shared intermediate sheared space to efficiently composite the two [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D medical volume data (organs), instrument geometry (surface primitives), viewer position (view matrix), and instrument position/orientation.
*   **Outputs:** A combined 2D image with correct perspective, where instruments are correctly occluded by or blended with organs.

**Data Structures:**
*   **Intermediate Plane Z-Buffer:** Two Z-buffers are maintained on the sheared image space $(u, v)$:
    *   $Z_{B,v}(u, v)$: Stores the terminated depth for volume rendering.
    *   $Z_{B,m}(u, v)$: Stores the depth for surface rendering of instruments.
*   **Mask Region:** A 2D boolean or integer mask on the sheared image space $(u, v)$ indicating pixels that require update due to instrument motion.

**Numbered Steps:**

1.  **Initialization:**
    *   Get instrument information (geometry, current position).
    *   Initialize the view-matrix corresponding to the viewer's position.
    *   Model the instrument image and its Z-buffer ($Z_{B,m}$) on the intermediate (sheared) plane according to the view-matrix and instrument information.
    *   Produce the volume-rendered intermediate image and its Z-buffer ($Z_{B,v}$) according to the view-matrix.
    *   Calculate the accumulated opacity up to the surface of the instrument.
    *   Combine the surface-rendered instrument image and the volume-rendered intermediate image using accumulated opacity by comparing $Z_{B,v}$ with $Z_{B,m}$.
    *   Warp the combined image according to the view-matrix to restore perspective.

2.  **Update Loop (While Motion):**
    *   **Case A: Instrument Motion (Rotation/Translation):**
        *   Model the instrument image and Z-buffer ($Z_{B,m}$) on the intermediate plane.
        *   Determine the mask region representing the instrument's movement (before and after motion).
        *   Produce the volume-rendered intermediate image and Z-buffer ($Z_{B,v}$) **only** on the mask region.
        *   Get accumulated opacity up to the surface of the instrument **only** on the mask region.
        *   Combine the surface-rendered instrument image and the volume-rendered intermediate image **only** on the mask region.
        *   Warp the combined image.
        *   Clear the mask.
    *   **Case B: Viewer Position Change (Rotation/Translation):**
        *   Modify the view-matrix.
        *   Model the instrument image and Z-buffer ($Z_{B,m}$) on the intermediate plane.
        *   Produce the volume-rendered intermediate image and Z-buffer ($Z_{B,v}$) for the entire image.
        *   Get accumulated opacity up to the surface of the instrument.
        *   Combine the images using accumulated opacity by comparing Z-buffers.
        *   Warp the combined image.

**Formulas:**

*   **Early Ray Termination Depth ($Z_{B,v}$):**
    The ray traversal for volume rendering is terminated when the accumulated opacity exceeds a threshold (0.95) or when the ray meets the instrument surface. The terminated depth stored in the Z-buffer is:
    $$ Z_{B,v}(u, v) = d(x_N) \quad \text{for the smallest } N \text{ such that } 1 - \prod_{i=1}^{N} (1 - \alpha(x_i)) \ge 0.95 $$
    Or, if the instrument is encountered:
    $$ Z_{B,v}(u, v) = d(x_N) \quad \text{for the smallest } N \text{ if } d(x_N) \ge Z_{B,m}(u, v) $$
    Where:
    *   $Z_{B,v}(u, v)$ is the terminated depth (Z-buffer) for volume rendering on sheared space $(u, v)$.
    *   $Z_{B,m}(u, v)$ is the Z-buffer for surface rendering of instruments.
    *   $d(x_N)$ is the distance from voxel $x_N$ to the pixel on sheared space $(u, v)$.
    *   $\alpha(x_i)$ is the opacity at voxel $x_i$.
    *   $x_i$ are voxels on a ray projected to $(u, v)$.

*   **Image Combination ($C$):**
    The surface-rendered instrument image is combined with the volume-rendered organ image using alpha blending, applied only when the instrument is inside the organ (overlapping case):
    $$ C(u, v) = \alpha_{acc}(u, v) \cdot f_m(u, v) + (1 - \alpha_{acc}(u, v)) \cdot I_v(u, v) $$
    Where:
    *   $C(u, v)$ is the combined image pixel.
    *   $f_m(u, v)$ is the pixel of the 2D instrument image.
    *   $I_v(u, v)$ is the pixel of the volume-rendered organ image from ray traversal up to $Z_{B,v}(u, v)$.
    *   $\alpha_{acc}(u, v)$ is the accumulated opacity of the organs.

**Parameters:**
*   **Early Ray Termination Threshold:** 0.95. The ray traversal is terminated when accumulated opacity exceeds this value [S1, S2].
*   **Instrument Opacity:** Assumed to be 1.0 (opaque). This ensures that once the instrument surface is hit, no further volume contribution is possible [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def combine_surface_volume(volume_data, instrument, view_matrix):
    """
    Combines surface-rendered instrument and volume-rendered organs
    using shear-warp factorization.
    """
    # 1. Initialization
    # Model instrument on intermediate (sheared) plane
    instrument_image, Z_Bm = model_instrument_on_sheared_plane(instrument, view_matrix)
    
    # Volume render on intermediate plane with early termination
    # Z_Bv stores the depth where ray terminates
    volume_image, Z_Bv, alpha_acc = volume_render_with_termination(
        volume_data, view_matrix, threshold=0.95, instrument_z_buffer=Z_Bm
    )
    
    # Combine images
    combined_image = combine_images(volume_image, instrument_image, alpha_acc, Z_Bv, Z_Bm)
    
    # Warp to final perspective
    final_image = warp_image(combined_image, view_matrix)
    
    return final_image

def update_instrument_motion(volume_data, instrument, view_matrix, previous_instrument, mask):
    """
    Updates rendering when only the instrument moves.
    Uses masking to reduce computation.
    """
    # Model new instrument position on sheared plane
    new_instrument_image, Z_Bm = model_instrument_on_sheared_plane(instrument, view_matrix)
    
    # Determine mask region (area of change)
    # mask is a 2D array indicating pixels to update
    update_mask = calculate_motion_mask(previous_instrument, instrument, view_matrix)
    
    # Volume render ONLY on mask region
    volume_image, Z_Bv, alpha_acc = volume_render_with_termination(
        volume_data, view_matrix, threshold=0.95, 
        instrument_z_buffer=Z_Bm, mask=update_mask
    )
    
    # Combine ONLY on mask region
    combined_image = combine_images_on_mask(
        volume_image, new_instrument_image, alpha_acc, Z_Bv, Z_Bm, update_mask
    )
    
    # Warp and clear mask
    final_image = warp_image(combined_image, view_matrix)
    clear_mask(update_mask)
    
    return final_image

def update_viewer_motion(volume_data, instrument, view_matrix):
    """
    Updates rendering when viewer position changes.
    Requires full recomputation.
    """
    # Update view matrix
    new_view_matrix = modify_view_matrix(view_matrix)
    
    # Full re-render
    instrument_image, Z_Bm = model_instrument_on_sheared_plane(instrument, new_view_matrix)
    volume_image, Z_Bv, alpha_acc = volume_render_with_termination(
        volume_data, new_view_matrix, threshold=0.95, instrument_z_buffer=Z_Bm
    )
    combined_image = combine_images(volume_image, instrument_image, alpha_acc, Z_Bv, Z_Bm)
    final_image = warp_image(combined_image, new_view_matrix)
    
    return final_image

def combine_images(volume_image, instrument_image, alpha_acc, Z_Bv, Z_Bm):
    """
    Combines volume and instrument images based on depth comparison.
    Alpha blending is applied only if instrument is inside organ (overlapping).
    """
    combined = empty_image_like(volume_image)
    
    for u, v in pixels:
        # Case 1: Instrument in front of organ (no contact)
        # Instrument is visible, volume is occluded.
        if Z_Bm[u, v] < Z_Bv[u, v]:
            combined[u, v] = instrument_image[u, v]
        
        # Case 2: Instrument inside organ (overlapping)
        # Apply alpha blending
        elif Z_Bm[u, v] >= Z_Bv[u, v]:
            # C(u,v) = alpha_acc * f_m + (1 - alpha_acc) * I_v
            combined[u, v] = (
                alpha_acc[u, v] * instrument_image[u, v] + 
                (1 - alpha_acc[u, v]) * volume_image[u, v]
            )
        
        # Case 3: Instrument behind organ
        # Volume is visible, instrument is occluded.
        else:
            combined[u, v] = volume_image[u, v]
            
    return combined

def volume_render_with_termination(volume_data, view_matrix, threshold, instrument_z_buffer, mask=None):
    """
    Performs volume rendering on sheared plane.
    Terminates ray if accumulated opacity >= threshold OR if ray hits instrument surface.
    """
    volume_image = empty_image_like(view_matrix.output_size)
    Z_Bv = empty_image_like(view_matrix.output_size)
    alpha_acc = empty_image_like(view_matrix.output_size)
    
    for u, v in pixels:
        if mask is not None and not mask[u, v]:
            continue
            
        accumulated_opacity = 0.0
        terminated_depth = infinity
        color = (0, 0, 0)
        
        # Traverse ray in sheared space
        for x_i in ray_traversal(volume_data, u, v, view_matrix):
            alpha_i = get_opacity(x_i)
            d_i = get_distance(x_i, u, v)
            
            # Check for instrument intersection
            if d_i >= instrument_z_buffer[u, v]:
                terminated_depth = d_i
                break
            
            # Update accumulated opacity and color
            weight = alpha_i * (1 - accumulated_opacity)
            color += get_color(x_i) * weight
            accumulated_opacity += weight
            
            # Early termination check
            if accumulated_opacity >= threshold:
                terminated_depth = d_i
                break
        
        # If ray didn't hit instrument or threshold, terminated_depth might be infinity
        # Store the depth of the last processed voxel or the termination point
        if terminated_depth == infinity:
             # Ray passed through without termination? 
             # Source implies Z_Bv is stored for the termination event.
             # If no termination, we might store the last voxel depth or handle as background.
             # For implementation safety, we store the depth of the last voxel if no termination occurred.
             terminated_depth = d_last # [G]
             
        Z_Bv[u, v] = terminated_depth
        alpha_acc[u, v] = accumulated_opacity
        volume_image[u, v] = color
        
    return volume_image, Z_Bv, alpha_acc
```

**Subtopics:**
*   **Parallel Perspective Shear-Warp Rendering:** Uses parallel projection instead of perspective, simplifying the warp step but losing depth cues. Choose when perspective is not critical. [Link](802-parallel-perspective-shear-warp-rendering.md)
*   **CPU-based Shear-Warp:** Optimizes the shear-warp algorithm for CPU architectures, often using SIMD instructions. Choose when GPU resources are unavailable. [Link](1161-cpu-based-shear-warp.md)
*   **Permutation Proof for Projection and Warp:** Provides a mathematical proof for the correctness of the shear-warp factorization. Useful for theoretical understanding. [Link](1702-permutation-proof-for-projection-and-warp.md)
*   **Warped Isosurface Extraction (WISE):** Extracts isosurfaces in the warped space to combine with volume rendering. Choose when explicit surface extraction is needed. [Link](1906-warped-isosurface-extraction-wise.md)
*   **Fast volume rotation using binary shear-warp factorization:** Uses binary data for faster rotation. Choose for binary segmentation masks. [Link](2006-fast-volume-rotation-using-binary-shear-warp-factorization.md)
*   **Spatially Partitioned Shear-Warp Method:** Partitions the volume to improve cache locality. Choose for large volumes. [Link](3481-spatially-partitioned-shear-warp-method.md)
*   **GPU-based Shear-Warp:** Implements shear-warp on GPU for parallel execution. Choose for high-performance real-time rendering. [Link](7425-gpu-based-shear-warp.md)

## Operating characteristics
*   **Time Complexity:** Volume rendering is $O(N)$ where $N$ is the number of voxels along the ray. Surface rendering is $O(M)$ where $M$ is the number of surface primitives. The combination step is $O(P)$ where $P$ is the number of pixels. Masking reduces the effective $N$ and $P$ for instrument motion updates.
*   **Memory Complexity:** Requires storage for the volume data, instrument geometry, two Z-buffers ($Z_{B,v}$ and $Z_{B,m}$), and the mask.
*   **Measured Performance:** Applied to a spine biopsy simulator and image-guided therapy surgical simulators for real-time visualization [S1, S2].
*   **Strengths:** Reduces computation by early ray termination and masking. Combines the realism of volume rendering with the speed of surface rendering.
*   **Limitations:** Requires segmentation for surface rendering (though only for instruments, not organs). Instrument opacity is assumed to be 1.0.
*   **When to Use:** Real-time surgical simulators where instruments move frequently and organs are complex.
*   **When Not to Use:** When instruments are transparent or have complex internal structures requiring volume rendering.

## Background
The method was developed for surgical simulators to address the need for real-time visualization of both medical instruments and organs [S1, S2]. It was presented at the SPIE International Symposium on Medical Imaging in San Diego, CA, February 2000, and published in *International Journal of CARS* in 2003 [S1]. The work builds on the shear-warp factorization technique, which decomposes the viewing transformation into a 3D shear, a projection, and a 2D warp to facilitate efficient resampling and compositing [S2].

## Sources
[S1] Fast Surface and Volume Rendering Based on Shear-Warp Factorization for a Surgical Simulator, https://doi.org/10.1002/igs.10050, fetched 2026-10-26
[S2] Fast surface and volume rendering based on shear-warp factorization for a surgical simulator., https://www.scienceopen.com/document?vid=077fb90e-0925-455c-85c6-4927cdfc1199, fetched 2026-10-30
