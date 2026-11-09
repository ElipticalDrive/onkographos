# Edge-Preserving Splatting with Compression/Stretching

Also called: C1 Continuous Splatting, Edge-Enhanced Splatting

## Overview
Edge-Preserving Splatting is a volume rendering technique that sharpens discontinuities in low-resolution or undersampled data by applying non-linear compression and stretching functions to standard splatting kernels [S1]. Unlike conventional splatting, which produces smooth, blurred edges, this method ensures C1 continuity (continuous first derivative) across splat boundaries while enhancing the visual definition of iso-surfaces [S1]. It distinguishes itself from sibling methods like [EWA Splatting](1069-ewa-splatting.md) by explicitly modeling edge geometry (direction and strength) to drive view-dependent footprint selection, rather than relying solely on geometric projection [S1].

## How it works
The method modifies the footprint function of a splat based on the local edge characteristics of the voxel. It requires precomputed edge information and uses a lookup table of pre-rendered splat footprints to achieve interactive performance.

### Inputs and Outputs
*   **Inputs:**
    *   Volume data $V$.
    *   Iso-contour value $C$ used for edge detection.
    *   View direction $\mathbf{v}$.
    *   Precomputed table of splat footprints indexed by edge position and strength.
*   **Outputs:**
    *   Final 2D image with enhanced edge sharpness.

### Data Structures
1.  **Voxel Attributes:** For each voxel $i$, store:
    *   Value $V_i$.
    *   Gradient $\nabla V_i = (\frac{\partial V}{\partial x}, \frac{\partial V}{\partial y}, \frac{\partial V}{\partial z})$ estimated via central differences [S1].
    *   Edge position $p_i$: The normalized distance from the voxel center to the iso-surface along the gradient direction. Calculated as $p_i = \frac{C - V_i}{|\nabla V_i|}$, normalized to $[0, 1]$ or similar range depending on the mapping function definition [S1]. *Note: The source describes using trilinear interpolation for edge position estimation in practice [S1].*
    *   Normalized edge direction $\mathbf{d}_i = \frac{\nabla V_i}{|\nabla V_i|}$ [S1].
2.  **Footprint Table:** A texture or memory array storing 2D footprint functions $F_{edge\_pos, strength}(x,y)$ precomputed for various combinations of edge position and projected edge strength [S1].

### Parameters
*   **Iso-contour value $C$:** Threshold for edge detection. In the Sinus Cavity dataset, a binary threshold at 128 was used after low-pass filtering [S1]. In the Hipip dataset, thresholds at 0.0017 and 0.5 were used [S1].
*   **Edge Strength:** Derived from the projected length of the edge direction onto the viewing plane.
*   **Kernel Type:** The base kernel $g(x,y)$ is typically a standard splatting kernel (e.g., Gaussian or cubic B-spline). The source does not specify the exact base kernel function, only that the mapping is applied to it [S1].

### Steps

1.  **Preprocessing (Offline/Setup):**
    1.  Compute gradients $\nabla V_i$ for all voxels using central differences [S1].
    2.  Compute edge position $p_i$ and normalized edge direction $\mathbf{d}_i$ for voxels near the iso-surface $C$ [S1].
    3.  Define the compression/stretching mapping function $map(x)$. The source defines a general shape but does not provide the explicit analytical formula for $map(x)$, only its properties:
        *   It maps the spatial domain to modify the footprint.
        *   It has a first derivative $map'(x) = 1$ at the boundaries $x=0$ and $x=1$ to ensure C1 continuity [S1].
        *   It creates asymmetric compression/stretching profiles based on edge strength [S1].
    4.  Precompute splat footprints for various edge positions and strengths using the mapping function and store them in texture memory [S1].

2.  **Rendering (Per Frame):**
    1.  For each voxel $i$ in the volume:
        a.  **Edge Classification:** Determine if the voxel is a "primary" or "secondary" kernel.
            *   If $V_i > C$ (or the specific threshold logic defined), it is one type; if $V_i < C$, it is the other. The source states: "primary and secondary kernels with larger or lower data value are specified for each splat" [S1]. The exact mapping of "primary/secondary" to "larger/lower" is consistent but arbitrary as long as it is defined [S1].
        b.  **View-Dependent Projection:**
            *   Project the normalized edge direction $\mathbf{d}_i$ onto the viewing plane perpendicular to $\mathbf{v}$ [S1].
            *   If $\mathbf{d}_i$ is parallel to $\mathbf{v}$, the edge is imperceptible; treat as a standard splat or skip edge enhancement [S1].
            *   Calculate the projected edge length/strength from the projected direction [S1].
        c.  **Footprint Selection:**
            *   Use the projected edge position and strength to index into the precomputed footprint table [S1].
            *   Select the corresponding 2D footprint $F_{selected}(x,y)$ [S1].
        d.  **Rasterization:**
            *   Transform the voxel to screen space.
            *   Rasterize the selected footprint $F_{selected}$ onto the image plane [S1].
    2.  **Compositing:**
        *   Accumulate the splats using standard alpha blending (back-to-front or front-to-back) [S1, see Parent].
    3.  **Lighting (Optional):**
        *   Apply view-dependent light source shading *only* to voxels labeled as primary or secondary edge splats [S1].

### Formulas

**C1 Continuity Condition:**
To ensure the first derivative is continuous across splat boundaries, the mapping function $rmap(x)$ (which is a shifted version of $map(x)$) must satisfy:
$$ rmap'(x) = 1 \quad \text{at} \quad x=0 \quad \text{and} \quad x=1 $$
The derivative of the modified kernel $h$ with respect to the original kernel $g$ is:
$$ h_x'(rmap(x)) = g_x'(rmap(x)) \cdot rmap'(x) $$
Since $rmap'(x) = 1$ at the boundaries:
$$ h_x'(rmap(x)) = g_x'(rmap(x)) $$
This ensures the slope of the enhanced kernel matches the slope of the original kernel at the join points, preserving C1 continuity [S1].

**Edge Position Estimation:**
$$ p = \frac{C - V_i}{|\nabla V_i|} $$
Where $C$ is the contour value and $\nabla V_i$ is the gradient [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing Phase
def preprocess_volume(volume_data, contour_value):
    """
    Computes edge attributes for each voxel.
    """
    edge_attributes = {}
    # Compute gradients using central differences
    gradients = compute_central_differences(volume_data)
    
    for voxel in volume_data:
        grad = gradients[voxel]
        val = volume_data[voxel]
        
        # Skip voxels far from the contour
        if abs(val - contour_value) > threshold:
            continue
            
        # Calculate edge position (normalized distance to iso-surface)
        # Note: Source mentions trilinear interpolation for position estimation
        # Simplified analytical form based on source description:
        dist = (contour_value - val) / norm(grad)
        
        # Normalized edge direction
        dir_norm = grad / norm(grad)
        
        # Classify as primary or secondary based on value relative to contour
        # Source: "primary and secondary kernels with larger or lower data value"
        is_primary = val > contour_value 
        
        edge_attributes[voxel] = {
            'position': dist,
            'direction': dir_norm,
            'is_primary': is_primary
        }
        
    return edge_attributes

def render_frame(volume_data, edge_attributes, view_direction, footprint_table):
    """
    Renders the volume using edge-preserving splatting.
    """
    image = create_empty_image()
    
    # Sort voxels back-to-front for correct alpha blending
    sorted_voxels = sort_voxels_by_depth(volume_data, view_direction)
    
    for voxel in sorted_voxels:
        attrs = edge_attributes.get(voxel)
        
        # If no edge info, use standard splat (not detailed in source, assumed standard)
        if attrs is None:
            render_standard_splat(voxel, image)
            continue
            
        # View-dependent edge processing
        edge_dir = attrs['direction']
        
        # Check if edge is parallel to view (imperceptible)
        if dot(edge_dir, view_direction) > 0.99: # Threshold for parallel
            render_standard_splat(voxel, image)
            continue
            
        # Project edge direction onto viewing plane
        # Projected direction is the component of edge_dir perpendicular to view_direction
        proj_edge_dir = edge_dir - dot(edge_dir, view_direction) * view_direction
        proj_edge_dir = normalize(proj_edge_dir)
        
        # Determine edge strength from projected length/direction
        # Source: "use this projected length to select a strength"
        strength = compute_edge_strength(proj_edge_dir, view_direction)
        
        # Select footprint from precomputed table
        # Index by projected edge position and strength
        footprint = footprint_table.lookup(attrs['position'], strength)
        
        # Rasterize the footprint
        rasterize_footprint(voxel, footprint, image)
        
        # Apply lighting only to primary/secondary edge splats
        if attrs['is_primary']:
            apply_lighting(voxel, view_direction, image)
            
    return image
```

### Edge Cases and Pitfalls
*   **Parallel Edge Direction:** If the viewing direction is parallel to the edge direction, the edge is imperceptible. The method skips special treatment for these voxels [S1].
*   **Undersampled Structures:** Small structures protruding into cavities are often undersampled due to high-frequency components in digitized data. This makes edge preservation challenging and may require adaptive multivariate representations or mixing datasets of different resolutions (e.g., 8-Tesla and 1.5-Tesla MRI) [S1].
*   **Edge Detection Accuracy:** The method requires fairly accurate edge detection. Iso-contouring produces sharp discontinuities, allowing good estimates of edge direction and contour distance. For experimental data (CT/MRI), advanced edge detection or segmentation is indispensable [S1].
*   **Asymmetry:** The compression/stretching function is not symmetric. The definition of "primary" vs "secondary" (based on data value being larger or lower than the threshold) must be consistent throughout the application [S1].

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ where $N$ is the number of voxels, assuming constant-time footprint lookup and rasterization. The preprocessing step is also $O(N)$ for gradient and edge attribute computation.
*   **Memory Complexity:** Requires storage for the volume data, edge attributes (gradient, position, direction per voxel), and the precomputed footprint table. The footprint table size depends on the discretization of edge position and strength [S1].
*   **Performance:** Enables interactive rendering by precomputing splats and loading them into texture memory [S1].
*   **Strengths:** Produces much sharper decreasing regions and nice sharp edges, especially visible in low-resolution or undersampled data, compared to conventional splatting [S1]. Enhances details on sidewalls and coarse resolution cross-sections [S1].
*   **Limitations:** Requires accurate edge detection and additional per-voxel data (gradient, direction). Does not handle high-frequency undersampled structures well without additional techniques like adaptive representations [S1].

## Background
*   **Origin:** The method was introduced in the paper "Edge Preservation in Volume Rendering Using Splatting" [S1].
*   **History:** It was applied to the Sinus Cavity dataset (derived from the Visible Human dataset) and the Hipip (HIgh Potential Iron Protein) dataset [S1]. The Sinus Cavity application involved virtual navigation to enhance sidewall details [S1]. The Hipip dataset was used to enhance details in protein structures [S1].
*   **Development:** The method builds on standard splatting principles but adds a layer of edge-aware footprint selection. It differs from previous work by requiring more information than simple binary classification, specifically edge direction and strength for view-dependent processing [S1].

## Sources
[S1] Edge Preservation in Volume Rendering Using Splatting, https://web.eecs.utk.edu/~huangj/papers/edgesplat.pdf, fetched 2026-10-28
