# Brain Peeling

Also called: Brain Peeling Cutting Tool

## Overview
Brain peeling is a view-independent volume visualization technique designed to reveal interior structures located in the vicinity of the brain surface by removing brain tissue step-by-step in the negative normal direction [S1]. Unlike its sibling [Opacity Peeling](8428-opacity-peeling.md), which relies on view-dependent occlusion culling, brain peeling uses a precomputed distance field to discard voxels based on their proximity to the surface, requiring fewer parameters and offering a more reliable exploration of surface-adjacent features like vessels and pathological lesions [S1].

## How it works
**Inputs:**
1.  A binary brain mask volume $M$ (1 for brain tissue, 0 for background) [S1].
2.  A scalar volume dataset $V$ (e.g., MRI intensity values) [S1].
3.  A user-defined distance threshold $d_{thresh}$ representing the depth of the peel [S1].

**Outputs:**
A rendered image where brain tissue within distance $d_{thresh}$ of the surface is removed, revealing underlying structures [S1].

**Data Structures:**
*   **Distance Field Volume ($D$):** A 3D grid of the same dimensions as $M$ and $V$. Each voxel $D(x,y,z)$ stores the Euclidean distance to the nearest non-brain voxel (surface).
*   **Brain Mask ($M$):** Binary volume indicating brain presence.

**Algorithm Steps:**

1.  **Compute Distance Field:**
    Calculate $D$ from the brain mask $M$. This can be done using a simple diffusion filter or a standard distance transformation algorithm [S1].
    $$ D(x,y,z) = \min_{(x',y',z') \notin \text{Brain}} \| (x,y,z) - (x',y',z') \| $$
    Where the minimum is taken over all voxels not classified as brain tissue [S1].

2.  **Shader Evaluation (Per Voxel):**
    For each voxel $(x,y,z)$ in the volume during ray casting or texture-based rendering:
    a. Retrieve the distance value $d = D(x,y,z)$ [S1].
    b. Retrieve the mask value $m = M(x,y,z)$ [S1].
    c. Determine if the voxel should be discarded. A voxel is discarded if it is part of the brain mask ($m > 0$) AND its distance to the surface is less than the threshold $d_{thresh}$ [S1].
    $$ \text{discard} = (m > 0) \land (d < d_{thresh}) $$
    [G] Note: The source states "intensity of the mask voxel is lower than the threshold which encodes the distance". In binary masks, "lower" usually implies the background value (0) is lower than the threshold if the threshold is positive, but the context "peel away... surface" implies keeping voxels *far* from the surface. The standard interpretation of "peeling" is removing the outer layer. Thus, we discard voxels *close* to the surface (small $d$). The source phrasing "intensity... lower than threshold" likely refers to the distance value $D$ being lower than $d_{thresh}$ for brain voxels. We follow the logical intent: discard brain voxels with $D < d_{thresh}$.

3.  **Alpha Blending for Smoothness:**
    To avoid hard edges at the peel boundary, blend alpha values in a small range in the normal direction [S1].
    [G] The source does not provide the exact blending function. A standard implementation uses a smoothstep or linear interpolation over a small transition width $\epsilon$ around $d_{thresh}$.
    $$ \alpha_{final} = \alpha_{original} \times \text{smoothstep}(d_{thresh} - \epsilon, d_{thresh}, d) $$
    If $d < d_{thresh} - \epsilon$, $\alpha_{final} = 0$. If $d > d_{thresh}$, $\alpha_{final} = \alpha_{original}$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_distance_field(mask_volume):
    """
    Computes the Euclidean distance transform for the brain mask.
    Returns a volume D where D[i] is the distance to the nearest non-brain voxel.
    """
    # Implementation depends on the chosen distance transformation algorithm
    # (e.g., chamfer distance, Euclidean distance transform via diffusion)
    # [S1] mentions "simple diffusion filter or distance transformation"
    return distance_transform(mask_volume)

def render_brain_peeling(volume_data, distance_field, threshold, epsilon=1.0):
    """
    Renders the volume with brain peeling applied.
    """
    # Initialize output image
    image = zeros_like(volume_data.shape[:2])
    
    for each ray in viewport:
        accumulated_color = 0
        accumulated_alpha = 0
        
        for each voxel along ray:
            # Get precomputed distance to surface
            d = distance_field[voxel_index]
            
            # Check if voxel is brain tissue (assuming mask is implicit or passed)
            # For this example, we assume we have access to the mask or 
            # that the distance field is only valid for brain tissue.
            # If d is infinite or NaN, it's background.
            if d is infinite:
                continue
                
            # Determine visibility based on threshold
            # Discard if distance is less than threshold
            if d < threshold:
                continue
                
            # Smooth blending at the boundary
            # [G] Standard smoothstep implementation for alpha modulation
            if d < threshold - epsilon:
                alpha_mod = 0.0
            elif d > threshold:
                alpha_mod = 1.0
            else:
                # Linear interpolation between 0 and 1 over the epsilon range
                alpha_mod = (d - (threshold - epsilon)) / epsilon
            
            # Get original opacity from volume data (e.g., via transfer function)
            original_alpha = transfer_function(volume_data[voxel_index])
            
            # Apply modulation
            final_alpha = original_alpha * alpha_mod
            
            # Standard volume compositing (front-to-back)
            color = volume_data[voxel_index]
            accumulated_color += color * final_alpha * (1 - accumulated_alpha)
            accumulated_alpha += final_alpha * (1 - accumulated_alpha)
            
            if accumulated_alpha > 0.99:
                break
                
        image[ray_index] = accumulated_color
        
    return image
```

**Parameters:**
*   $d_{thresh}$: The distance threshold for peeling. Typical values depend on the resolution and desired depth of exploration. Not given in the sources.
*   $\epsilon$: The width of the blending region for smooth visualization. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The accuracy of the peel depends on the precision of the distance field computation.
*   **Boundaries:** The technique assumes a robust skull stripping method has been applied to generate the initial brain mask [S1].
*   **Degenerate Input:** If the brain mask is incomplete or noisy, the distance field may produce incorrect peel depths, potentially removing healthy tissue or failing to reveal lesions [S1].

## Subtopics

### Automatic Axis-Aligned Cutting Tool
A complementary technique that divides the volume into eight octants and hides those between the viewpoint and the Region of Interest (ROI) [S1]. It is view-dependent and used for coarse exploration, whereas brain peeling is view-independent and used for fine surface exploration [S1].

### Opacity Peeling
The parent technique that uses view-dependent occlusion to reveal hidden structures. Brain peeling is an alternative that is view-independent and requires fewer parameters [S1].

## Operating characteristics
*   **Time Complexity:** The distance field computation is a one-time pre-processing cost. Rendering time is comparable to standard volume rendering with minimal overhead, as the distance field is accessed via texture lookup [S1].
*   **Memory Complexity:** Requires storing an additional distance field volume of the same size as the input data [S1].
*   **Measured Performance:** On a Mac Pro 2.66 GHz with 4 GB RAM and ATI Radeon X1900, rendering at 512x512 viewport with 205x225x158 data achieves 7 fps [S1]. The method incurs minimal performance loss compared to manual clipping, primarily due to the additional texture lookup for the distance field [S1].
*   **Strengths:** View-independent, intuitive exploration of surface-adjacent structures, fewer parameters than opacity peeling, reliable for medical experts [S1].
*   **Limitations:** Requires a high-quality brain mask (skull stripping) as a prerequisite [S1]. Not suitable for exploring deep interior structures far from the surface.

## Background
Brain peeling was introduced by Rezk-Salama and Kolb in 2008 as part of a framework for high-quality multimodal volume visualization of intracerebral pathological tissue [S1]. It was inspired by opacity peeling [RSK06] and the work of Tappenbeck et al. [TPD06] [S1]. The technique is specifically designed for neuro-visualization, allowing medical experts to intuitively explore structures like vessels and tumors located near the brain surface, which are difficult to examine with standard cutting planes [S1].

## Sources
[S1] High-Quality Multimodal Volume Visualization of Intracerebral Pathological Tissue, https://doi.org/10.2312/vcbm/vcbm08/167-176, fetched 2026-10-26
