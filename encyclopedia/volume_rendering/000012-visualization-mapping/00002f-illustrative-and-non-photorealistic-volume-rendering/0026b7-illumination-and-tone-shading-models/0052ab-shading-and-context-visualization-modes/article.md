# Shading and Context Visualization Modes

Also called: Focus and Context Shading

## Overview
Shading and Context Visualization Modes is a volume rendering technique that separates data into "focus" (primary interest) and "context" (surrounding structure) clusters to enhance perceptual clarity [S1, unofficial]. It solves the problem of visual clutter in complex multiclassified volumes by rendering the focus with high-fidelity shading while rendering the context as simplified, semi-transparent proxies [S1, unofficial]. It is distinct from its siblings, such as [Lit Sphere Shading](785-lit-sphere-shading.md) or [Volume Illustration Tone Shading](805-volume-illustration-tone-shading.md), by explicitly decoupling the rendering pipeline into two distinct semantic layers with different shading models and opacity modulations, rather than applying a single stylized lighting model to the entire volume [S1, unofficial].

## How it works
The method operates on a pre-segmented volume where voxels are classified into focus clusters and context clusters. The context is often represented as a hierarchical structure (ghosting depth) [S1, unofficial].

**Inputs:**
- Volume data $V$ with classification labels.
- Focus cluster bounding box and center.
- Context cluster hierarchy (depth levels).
- Shading type parameter $\Sigma \in \{\text{Phong}, \text{E+A}, \text{E+A+S}\}$ [S1, unofficial].
- Context visualization mode $\Omega \in \{\text{All Voxels}, \text{Boundary Voxels}, \text{Bounding Boxes}\}$ [S1, unofficial].

**Outputs:**
- Final color $c$ for each fragment.

**Data Structures:**
- **Context Cluster Hierarchy:** A tree where nodes represent clusters. Each node has a `depth` (ghosting depth) and `opacity` factor.
- **Gradient Modulation Transfer Function (GMTF):** A function mapping local gradient magnitude to a width/opacity factor for boundary context [S1, unofficial].

**Steps:**

1. **Camera Setup:**
   - Compute view reference point $P_{cam}$ as the center of the focus clusters [S1, unofficial].
   - Compute projection window as the bounding box of the union of focus and all context clusters [S1, unofficial].

2. **Context Opacity Pre-computation (Parser Stage):**
   - If $\Omega = \text{Bounding Boxes}$:
     - Set Context Transfer Function (CTF) opacity to 0 for all context voxels to prevent them from being rendered during standard texture slicing [S1, unofficial].
   - If $\Omega = \text{Boundary Voxels}$:
     - Compute GMTF factor $g$ for each context voxel based on its gradient magnitude [S1, unofficial].
     - Modulate context surface width/opacity using $g$. Deeper hierarchy levels (higher ghosting depth) result in thinner, more transparent surfaces [S1, unofficial].
   - If $\Omega = \text{All Voxels}$:
     - Render context voxels with standard opacity derived from their cluster hierarchy depth [S1, unofficial].

3. **Fragment Shader Execution (Per Sample):**
   - Determine if the current sample belongs to Focus ($F$) or Context ($C$).
   - **If Focus:**
     - Compute shading using the global parameter $\Sigma$ [S1, unofficial].
     - If $\Sigma = \text{Phong}$: Use Phong illumination model with normal $\nabla f_n$ [S1, unofficial].
     - If $\Sigma = \text{E+A}$: Compute emission and absorption [S1, unofficial].
     - If $\Sigma = \text{E+A+S}$: Combine emission, absorption, and shading [S1, unofficial].
     - Color $c_F$ is derived from the focus transfer function (typically RGB) [S1, unofficial].
   - **If Context:**
     - Convert context cluster RGB values to grayscale to enhance contrast with the colored focus [S1, unofficial].
     - Compute shading using the same global parameter $\Sigma$ [S1, unofficial].
     - Apply opacity modulation based on ghosting depth and/or GMTF [S1, unofficial].
     - Color $c_C$ is the grayscale representation of the context cluster [S1, unofficial].

4. **Compositing:**
   - Apply per-plane opacity modulation to proxy polygons (if used) in front of the focus bounding box to increase transparency of foreground context [S1, unofficial].
   - Composite $c_F$ and $c_C$ using standard volume rendering integration (e.g., front-to-back alpha blending) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shading_and_context_visualization_mode(sample, focus_bbox, context_hierarchy, shading_type, context_mode):
    """
    sample: dict with 'value', 'gradient', 'cluster_id', 'depth'
    focus_bbox: bounding box of focus clusters
    context_hierarchy: dict mapping cluster_id to {'depth': int, 'color': (r,g,b)}
    shading_type: str ('Phong', 'E+A', 'E+A+S')
    context_mode: str ('All Voxels', 'Boundary Voxels', 'Bounding Boxes')
    """
    cluster_id = sample['cluster_id']
    is_focus = cluster_id in focus_clusters # Assumed global check
    
    if is_focus:
        # Focus Rendering
        normal = normalize(sample['gradient'])
        if shading_type == 'Phong':
            color = phong_shading(normal, sample['value'])
        elif shading_type == 'E+A':
            color = emission_absorption(sample['value'])
        elif shading_type == 'E+A+S':
            color = emission_absorption_shading(normal, sample['value'])
        
        # Focus color is typically RGB from transfer function
        final_color = apply_focus_transfer_function(color)
        alpha = 1.0 # Focus is opaque/highly visible
        
    else:
        # Context Rendering
        cluster_info = context_hierarchy[cluster_id]
        depth = cluster_info['depth']
        base_color = cluster_info['color']
        
        # Convert to grayscale for contrast
        gray_color = rgb_to_grayscale(base_color)
        
        # Compute shading for context (same model as focus)
        normal = normalize(sample['gradient'])
        if shading_type == 'Phong':
            shade = phong_shading(normal, sample['value'])
        elif shading_type == 'E+A':
            shade = emission_absorption(sample['value'])
        elif shading_type == 'E+A+S':
            shade = emission_absorption_shading(normal, sample['value'])
            
        context_color = gray_color * shade
        
        # Opacity Modulation
        if context_mode == 'Bounding Boxes':
            alpha = 0.0 # Prevent rendering during texture slicing
        elif context_mode == 'Boundary Voxels':
            # GMTF modulation based on gradient
            gmtf_factor = compute_gmtf(sample['gradient'])
            # Depth-based transparency: deeper = thinner/more transparent
            depth_factor = 1.0 / (1.0 + depth) 
            alpha = gmtf_factor * depth_factor
        else: # All Voxels
            # Standard depth-based transparency
            alpha = 1.0 / (1.0 + depth)
            
        final_color = context_color
        alpha = clamp(alpha, 0.0, 1.0)

    return final_color, alpha
```

**Edge Cases and Pitfalls:**
- **Boundary Definition:** If the focus cluster boundary is ill-defined, the context bounding box may overlap incorrectly with focus data, causing visual artifacts [S1, unofficial].
- **Gradient Noise:** In low-resolution volumes, gradient estimation for GMTF may be noisy, leading to erratic context opacity [S1, unofficial].
- **Performance:** Computing shading for both focus and context layers doubles the shading computation cost compared to single-layer rendering [S1, unofficial].

## Subtopics

### 3-D Contextual Shading (3x3x3)
A method for binary voxel models that uses a divide-and-conquer strategy to estimate surface normals from a 3x3x3 neighborhood [S2, unofficial]. It represents the neighborhood with three orthogonal 3x3 planes, computes 2D normal vectors for the 256 possible patterns in each plane into a lookup table, and combines them at runtime [S2, unofficial]. To handle ambiguous cases, multiple normal vectors are encoded and selected based on the viewing direction quadrant [S2, unofficial]. It also adds diagonal vectors from occupied corner voxels to account for vertex-adjacent influences neglected by the orthogonal planes [S2, unofficial]. It achieves a mean error of 0.0337 and standard deviation of 0.0518 compared to polygon models, performing slightly better than gray-level methods near corners [S2, unofficial].

### [3-D Contextual Shading (5x5x5)](20379-3-d-contextual-shading-5x5x5.md)
An extension of the 3x3x3 method using a 5x5x5 neighborhood. Due to the impractical size of a full lookup table for 5x5 planes (25 entries each), it employs systematic elimination of interior and redundant entries that have little effect on the normal vector. This reduces the effective combinations to fewer than 1,000, keeping the lookup table size manageable while capturing smoother surface inclinations than the 3x3x3 method.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, with a constant factor roughly doubled due to dual-layer shading [S1, unofficial]. The 3x3x3/5x5x5 normal estimation is $O(1)$ per voxel using lookup tables [S2, unofficial].
- **Memory Complexity:** $O(N)$ for volume data. The 3x3x3 method requires a small lookup table (256 entries per plane) [S2, unofficial]. The 5x5x5 method requires a reduced lookup table (<1,000 entries) [S2, unofficial].
- **Performance:** Real-time performance is achievable for the 3x3x3 method on standard hardware [S2, unofficial].
- **Strengths:** Enhances perception of complex structures by separating focus and context; reduces visual clutter; real-time capable [S1, unofficial][S2, unofficial].
- **Limitations:** Requires pre-segmentation of focus and context; context shading may obscure fine details if opacity is not carefully tuned [S1, unofficial].

## Background
The concept of focus and context rendering was introduced to address the limitations of traditional volume rendering in displaying complex, multiclassified data [S1, unofficial]. The specific 3-D contextual shading methods (3x3x3 and 5x5x5) were developed by Chen et al. in the mid-1990s to overcome the limitations of existing methods like Hohne’s gray-scale shading, which was sensitive to distance value distributions [S2, unofficial]. The 3x3x3 method was designed for binary voxel models, while the 5x5x5 method extended this to larger neighborhoods with optimized pattern reduction [S2, unofficial].

## Sources
[S1] Structural focus+context rendering of multiclassified volume data, http://hdl.handle.net/2117/86811, fetched 2026-10-26 (unofficial)
[S2] Simple visualization techniques for die casting part and die design. Final report, July 1, 1995--September 30, 1997, https://digital.library.unt.edu/ark:/67531/metadc690873/?q=voxelization triangle mesh, fetched 2026-10-28 (unofficial)
