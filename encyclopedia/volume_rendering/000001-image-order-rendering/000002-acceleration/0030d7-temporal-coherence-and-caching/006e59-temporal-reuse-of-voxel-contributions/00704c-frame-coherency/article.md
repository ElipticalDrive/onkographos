# Frame Coherency

Also called: Animation frame reuse, Frame Coherence

## Overview
Frame coherency is an optimization for animation sequences that exploits the small difference between successive images to expedite the generation of the next image [S1, unofficial]. In the context of non-photorealistic rendering and silhouette-based techniques, it specifically refers to schemes that vary the opacity of rendered elements based on the dot product of the surface normal and the view vector to achieve "fade in" and "fade out" effects, thereby stabilizing silhouettes and reducing visual distraction [S2]. It builds on [Temporal Reuse of Voxel Contributions](21611-temporal-reuse-of-voxel-contributions.md) by addressing temporal stability in image-space rendering rather than just voxel integration caching.

## How it works
Frame coherency operates on the principle that silhouettes and edges exhibit varying stability under small view changes. The method stabilizes these features by modulating their opacity based on their geometric relationship to the camera.

**Inputs and Outputs:**
- Input: Surface normal vector $\mathbf{n}$, view vector $\mathbf{v}$ (pointing from surface to camera), and base opacity $\alpha_{base}$.
- Output: Modulated opacity $\alpha_{final}$ for the rendered quad or edge.

**Data Structures:**
- No specific complex data structures are required beyond standard vertex attributes (normal) and view parameters.

**Steps:**
1. Calculate the dot product between the surface normal $\mathbf{n}$ and the view vector $\mathbf{v}$.
   $$ d = \mathbf{n} \cdot \mathbf{v} $$
2. Map the dot product $d$ to an opacity factor. The opacity is varied to "fade in" when the surface is facing the viewer (high stability) and "fade out" when the surface is edge-on or moving rapidly across the view (low stability) [S2].
   $$ \alpha_{final} = f(d) \cdot \alpha_{base} $$
   Where $f(d)$ is a function that increases opacity as $d$ approaches 1 (front-facing) and decreases it as $d$ approaches 0 or -1 (side/back-facing). Specific implementations may use linear or smoothstep mappings [S2].

**Parameters:**
- The exact mapping function $f(d)$ is not strictly defined in the general description but is implementation-dependent to achieve the desired "fade" effect [S2].

**Edge Cases and Pitfalls:**
- Unstable silhouettes that appear and disappear quickly can still cause distraction if the fade rate is too slow [S2].
- The method assumes small changes in view between frames; large camera movements may break the coherence assumption.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_frame_coherent_opacity(surface_normal, view_vector, base_opacity):
    """
    Computes opacity for frame coherency based on normal-view alignment.
    
    Args:
        surface_normal: Normalized 3D vector (nx, ny, nz)
        view_vector: Normalized 3D vector pointing from surface to camera (vx, vy, vz)
        base_opacity: Float in [0, 1]
        
    Returns:
        Float in [0, 1] representing the final opacity
    """
    # Calculate dot product of normal and view vector
    dot_product = surface_normal.x * view_vector.x + \
                  surface_normal.y * view_vector.y + \
                  surface_normal.z * view_vector.z
    
    # Map dot product to opacity factor
    # Fade in when facing camera (dot ~ 1), fade out when edge-on (dot ~ 0)
    # A simple linear mapping from [0, 1] to [0, 1] is used as a baseline
    # Note: Specific mapping functions may vary by implementation [S2]
    opacity_factor = max(0.0, min(1.0, dot_product))
    
    # Apply factor to base opacity
    final_opacity = opacity_factor * base_opacity
    
    return final_opacity
```

## Subtopics

### Edge Classification for Opacity
Brosz et al. [BSS04] define a classification scheme for edges and corresponding opacities to further refine stability [S2].

### Suggestive Contour Stability
De-Carlo et al. [DFR04] apply stability to suggestive contours, using motion measures to discard contours that move too fast and using opacity to fade stable remaining contours [S2].

## Operating characteristics
- **Time Complexity**: $O(1)$ per fragment/quad for the dot product and opacity calculation.
- **Memory Complexity**: $O(1)$ additional memory per fragment.
- **Strengths**: Reduces visual distraction caused by unstable silhouettes; improves perceptual quality of animated NPR renders.
- **Limitations**: Does not accelerate ray casting or voxel traversal directly; primarily an aesthetic/stability optimization for image-space rendering.
- **When to use**: When rendering animated sequences with silhouette-based or edge-based NPR techniques where view changes are small between frames.
- **When not to use**: For large camera movements or when photorealistic rendering is required without stylized edge effects.

## Background
The concept of frame coherency in volume rendering generally refers to reusing work from previous frames due to small differences between successive images [S1, unofficial]. In the specific context of non-photorealistic rendering, Lengyel et al. [LPFH01] provided a scheme for using the dot product of the surface normal with the view vector to vary opacity for improved frame coherence [S2]. This approach addresses the "stability" of silhouettes, which refers to the likelihood of a silhouette being visible following small changes in view [S2].

## Sources
[S1] Volume Rendering, https://www.cs.unh.edu/~cs880/visualization/Celebi-Volume%20Rendering.pdf, fetched 2026-10-26 (unofficial)
[S2] State of the Art Non-Photorealistic Rendering (NPR) Techniques, https://doi.org/10.2312/localchapterevents/tpcg/tpcg06/089-098, fetched 2026-10-26
