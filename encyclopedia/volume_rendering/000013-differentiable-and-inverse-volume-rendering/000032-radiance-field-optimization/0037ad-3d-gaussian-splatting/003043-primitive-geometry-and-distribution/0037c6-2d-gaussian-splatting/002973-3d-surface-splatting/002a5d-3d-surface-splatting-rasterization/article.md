# 3D Surface Splatting Rasterization

Also called: 3DSS rasterization, Depth-Interval Grouping, Interval-based surface separation, Multi-layer interval grouping, Per-pixel rasterization with depth-interval grouping

## Overview
This method is the rasterization core of 3D Surface Splatting (3DSS), designed to render an unstructured set of oriented surfel primitives into a 2D image while correctly handling self-occlusion at concavities. It solves the problem where single-layer rasterization approaches fail to distinguish between geometrically distinct surface sheets that project onto the same pixel, resulting in holes or translucent artifacts in high-curvature regions like ears or under chins. It sets itself apart from classical A-buffer approaches by restricting summation to surfels within the same depth-interval layer rather than gathering across all depths, and from single-layer ternary depth tests by detecting gaps in the depth-interval chain to support an arbitrary number of surface layers per pixel.

## How it works
The algorithm processes surfels that have already been shaded via split-sum Image-Based Lighting (IBL) against a co-optimized HDR environment, yielding a pre-shaded linear radiance $\mathbf{c}_i$ [S1, unofficial]. The rasterization proceeds in three main phases: tiling and sorting, per-pixel interval grouping, and front-to-back compositing.

### Inputs and Outputs
**Inputs:**
- A set of surfels $\mathcal{S}$, each with position $\mathbf{p}_i$, orientation, size, and pre-shaded color $\mathbf{c}_i$.
- Camera parameters (view/projection matrices) to compute screen-space positions and depth intervals.
- Kernel cutoff radius $r_{\text{cut}}$.
- Half-thickness parameter $\varepsilon_z^{(i)}$ for each surfel, defining the depth interval extent.

**Outputs:**
- A composited HDR image.
- Gradients flowing back to surfel parameters and the environment map.

### Data Structures
- **Screen-space tiles:** Surfels are binned into tiles based on their screen-space bounding boxes.
- **Sorted primitive sequence:** Within each tile, surfels are radix-sorted by their depth-interval start value $z_{\text{start}}^{(i)}$.
- **Per-pixel accumulators:** For each pixel, the algorithm maintains:
  - $\mathbf{C}_g$: Accumulated color for the current surface group $g$.
  - $W_g$: Accumulated weight for the current surface group $g$.
  - $z_{\text{end}}^{\max}$: The farthest depth extent encountered so far in the current group.

### Numbered Steps

1.  **Tiling and Sorting:** Bin surfels into screen-space tiles. Radix-sort the surfels within each tile by $z_{\text{start}}^{(i)}$ [S1, unofficial].
2.  **Per-Pixel Initialization:** For each pixel, initialize group accumulators $\mathbf{C}_g \leftarrow \mathbf{0}$, $W_g \leftarrow 0$, and $z_{\text{end}}^{\max} \leftarrow 0$ [S1, unofficial].
3.  **Iterate Surfels:** For each surfel $i$ in the sorted sequence:
    a.  **Intersection:** Compute the ray-surfel intersection to obtain local coordinates $(u, v)$ [S1, unofficial].
    b.  **Weight Evaluation:** Calculate the MIP-filtered squared distance $\rho_{\text{mip}}^2$ using the precomputed inverse covariance $\Sigma_{\text{mip}}^{-1}$ [S1, unofficial].
    c.  **Culling:** If $\rho_{\text{mip}}^2 \geq r_{\text{cut}}^2$, the surfel does not cover this pixel; skip to the next surfel [S1, unofficial].
    d.  **Weight Calculation:** Compute the scalar weight $w = n_f \cdot \exp(-\frac{1}{2} \rho_{\text{mip}}^2)$ [S1, unofficial].
    e.  **Depth Interval:** Define the depth interval $[z_{\text{start}}^{(i)}, z_{\text{end}}^{(i)}]$ where $z_{\text{start}}^{(i)} = z^{(i)} - \varepsilon_z^{(i)}$ and $z_{\text{end}}^{(i)} = z^{(i)} + \varepsilon_z^{(i)}$ [S1, unofficial].
    f.  **Gap Detection:** If $W_g > 0$ AND $z_{\text{start}}^{(i)} > z_{\text{end}}^{\max}$, a gap is detected between the current surfel and the accumulated group.
        i.  Finalize the current layer by calling `FLUSHLAYER($\mathbf{C}_g, W_g$)` [S1, unofficial].
        ii. Reset accumulators: $\mathbf{C}_g \leftarrow \mathbf{0}$, $W_g \leftarrow 0$, $z_{\text{end}}^{\max} \leftarrow 0$ [S1, unofficial].
    g.  **Update Extent:** Update the group's farthest extent: $z_{\text{end}}^{\max} \leftarrow \max(z_{\text{end}}^{\max}, z_{\text{end}}^{(i)})$ [S1, unofficial].
    h.  **Accumulate:** Add the surfel's contribution to the current group:
        $\mathbf{C}_g \leftarrow \mathbf{C}_g + w \cdot \mathbf{c}_i$
        $W_g \leftarrow W_g + w$ [S1, unofficial].
4.  **Finalize Last Layer:** After processing all surfels for a pixel, if $W_g > 0$, call `FLUSHLAYER($\mathbf{C}_g, W_g$)` to finalize the last accumulated group [S1, unofficial].
5.  **Shepard Normalization:** For each finalized surface group, normalize the accumulated color by its weight to obtain the layer color: $\mathbf{C}_{\text{layer}} = \mathbf{C}_g / W_g$ [S1, unofficial].
6.  **Coverage Opacity:** Compute the coverage opacity $\alpha_{\text{layer}}$ for each layer from the accumulated weight $W_g$ using a smooth, monotonically increasing mapping [S1, unofficial]. This replaces threshold-based clamping used in classical A-buffers.
7.  **Front-to-Back Compositing:** Composite the resulting layers front-to-back using the computed coverages and opacities to produce the final pixel color [S1, unofficial].
8.  **Post-Processing:** Blend the composited HDR image with the background, then apply tone-mapping and gamma correction [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterize_pixel(sorted_surfels, camera_ray, r_cut, n_f):
    """
    Rasterizes a single pixel using depth-interval grouping.
    
    Args:
        sorted_surfels: List of surfels sorted by z_start.
        camera_ray: Ray origin and direction for the pixel.
        r_cut: Kernel cutoff radius squared.
        n_f: Front-facing normal factor (constant for the surfel).
        
    Returns:
        final_color: The composited HDR color for the pixel.
    """
    # Per-pixel accumulators
    C_g = Vector3(0, 0, 0)
    W_g = 0.0
    z_end_max = 0.0
    
    # List to store finalized layers (color, opacity)
    layers = []
    
    for surfel in sorted_surfels:
        # 1. Ray-Surfel Intersection
        # Returns local coordinates (u, v) if hit, else None
        uv = intersect_ray_surfel(camera_ray, surfel)
        if uv is None:
            continue
            
        # 2. Evaluate MIP-filtered weight
        # Precomputed inverse covariance Sigma_mip_inv is part of surfel data
        rho_mip_sq = evaluate_mip_weight(uv, surfel.Sigma_mip_inv)
        
        # 3. Culling
        if rho_mip_sq >= r_cut:
            continue
            
        # 4. Calculate scalar weight
        w = n_f * exp(-0.5 * rho_mip_sq)
        
        # 5. Define depth interval
        z_center = surfel.z_center
        epsilon_z = surfel.epsilon_z
        z_start = z_center - epsilon_z
        z_end = z_center + epsilon_z
        
        # 6. Gap Detection
        if W_g > 0 and z_start > z_end_max:
            # Finalize current group
            if W_g > 0:
                layer_color = C_g / W_g
                # Coverage opacity mapping (exact function not specified in source, 
                # assumed to be a smooth monotonic function of W_g)
                layer_opacity = map_weight_to_opacity(W_g)
                layers.append((layer_color, layer_opacity))
            
            # Reset accumulators
            C_g = Vector3(0, 0, 0)
            W_g = 0.0
            z_end_max = 0.0
            
        # 7. Update extent
        if z_end > z_end_max:
            z_end_max = z_end
            
        # 8. Accumulate
        C_g += w * surfel.color
        W_g += w
        
    # Finalize last group
    if W_g > 0:
        layer_color = C_g / W_g
        layer_opacity = map_weight_to_opacity(W_g)
        layers.append((layer_color, layer_opacity))
        
    # 9. Front-to-Back Compositing
    final_color = Vector3(0, 0, 0)
    alpha_acc = 0.0
    
    # Layers are sorted by depth because surfels were sorted by z_start
    # and gaps ensure correct layer separation.
    for layer_color, layer_opacity in layers:
        # Standard front-to-back blending
        # C_out = C_in + C_layer * alpha_layer * (1 - alpha_acc)
        # alpha_out = alpha_acc + alpha_layer * (1 - alpha_acc)
        
        contribution = layer_color * layer_opacity * (1.0 - alpha_acc)
        final_color += contribution
        alpha_acc += layer_opacity * (1.0 - alpha_acc)
        
    return final_color

def map_weight_to_opacity(W_g):
    """
    Maps accumulated weight to coverage opacity.
    The source describes this as a "smooth, monotonically increasing mapping"
    based on the assumption that surfel distribution approximates a partition of unity.
    The exact formula is not provided in the source text. [G]
    A common approximation for Gaussian splatting coverage is 1 - exp(-W_g).
    """
    return 1.0 - exp(-W_g)
```

### Parameters
- $\varepsilon_z^{(i)}$: Half-thickness of surfel $i$'s depth interval. This parameter defines the start and end of the depth interval $[z^{(i)} - \varepsilon_z^{(i)}, z^{(i)} + \varepsilon_z^{(i)}]$ used for gap detection. Typical values depend on the surfel size and camera distance, but the source does not specify a fixed value.
- $r_{\text{cut}}$: Kernel cutoff radius. Surfels with $\rho_{\text{mip}}^2 \geq r_{\text{cut}}^2$ are skipped.
- $n_f$: Front-facing normal factor. Used in the weight calculation $w = n_f \cdot \exp(-\frac{1}{2} \rho_{\text{mip}}^2)$.

### Edge Cases and Pitfalls
- **Self-occlusion at concavities:** Without depth-interval grouping, single-layer approaches produce self-occlusion errors at high-curvature concavities (e.g., inside ears or under chins) where surfels from geometrically distinct surface sheets project onto the same pixel. This manifests as holes or translucent patches revealing the background [S1, unofficial].
- **Silhouette artifacts:** Single-layer approaches can produce a "crusty" shell extending beyond the true silhouette, with aliased contours instead of gradual opacity falloff. The multi-layer approach with smooth coverage falloff anti-aliases silhouettes without post-process edge detection [S1, unofficial].
- **Precision:** The gap detection relies on comparing $z_{\text{start}}^{(i)}$ with $z_{\text{end}}^{\max}$. Floating-point precision issues could theoretically cause incorrect gap detection if surfels are extremely close in depth, though the source does not specify mitigation strategies.

## Subtopics

### [Rasterize-then-splat (RtS)](6690-rasterize-then-splat-rts.md)
Rasterize-then-splat (RtS) is a method for differentiable rendering of 3D surfaces that supports both explicit and implicit representations, providing derivatives at occlusion boundaries while remaining fast and simple to implement. It differs from alternatives like Soft Rasterizer by avoiding costly closest-point queries or mesh processing, and from Nvdiffrast by requiring no specific GPU hardware or custom derivative code, making its differentiable cost independent of mesh size.

## Operating characteristics
- **Time Complexity:** The sorting step is $O(N \log N)$ for $N$ surfels per tile. The per-pixel processing is linear in the number of surfels covering the pixel, with a constant number of operations per surfel (intersection, weight evaluation, accumulation).
- **Memory Complexity:** Requires storage for the sorted surfel list per tile and per-pixel accumulators. The number of layers per pixel is typically small, so memory overhead is low.
- **Measured Performance:** The source does not provide specific performance benchmarks (FPS, memory usage) for the rasterization step alone. It notes that the method is "fast and simple to implement" in the context of RtS comparison, but 3DSS itself is a complex optimization framework.
- **Strengths:** Correctly handles self-occlusion at concavities without explicit mesh processing. Anti-aliases silhouettes naturally through smooth coverage falloff. Differentiable gradients flow back through all operations.
- **Limitations:** Performance depends on the number of surfels per pixel and the number of layers. The source does not specify how many layers are typical or how this scales with scene complexity.
- **When to use:** When rendering unstructured surface representations (surfels) where self-occlusion at concavities is a concern, and differentiability is required.
- **When not to use:** For simple opaque meshes where a single layer suffices, or when hardware-specific optimizations (like Nvdiffrast) are available and preferred.

## Background
3D Surface Splatting (3DSS) was introduced in the paper "3DSS: 3D Surface Splatting for Inverse Rendering" [S1, unofficial]. The method is part of a framework for physically-based inverse rendering that jointly recovers geometry, materials, and illumination from multi-view images. The rasterization algorithm described here is a key component of this framework, enabling the differentiable rendering of surfels. The approach builds on earlier work in splatting and A-buffer techniques but introduces depth-interval grouping to handle multiple surface layers.

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
