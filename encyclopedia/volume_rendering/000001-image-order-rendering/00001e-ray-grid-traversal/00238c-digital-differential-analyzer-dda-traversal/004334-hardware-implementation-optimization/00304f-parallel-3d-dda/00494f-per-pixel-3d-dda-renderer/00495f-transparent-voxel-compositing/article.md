# Transparent Voxel Compositing

Also called: Front-to-back voxel compositing, Beer–Lambert voxel rendering, closure-based voxel shading

## Overview
Transparent Voxel Compositing is a rendering strategy for alpha-blended, additive, and Beer–Lambert volumetric voxels that relies on strict front-to-back traversal order to eliminate the need for depth sorting or Order-Independent Transparency (OIT) schemes [S1]. It is distinct from general OIT methods because it leverages the spatial coherence of voxel grids and the deterministic traversal of the parent [Per-pixel 3D-DDA Renderer](12466-per-pixel-3d-dda-renderer.md) to composite samples in a single pass [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a voxel grid containing transparent materials; a material palette defining per-voxel or per-instance properties.
*   **Output:** The final pixel color $\mathbf{C}$ and accumulated transmittance.

**Data Structures:**
*   **Material Palette:** A 256-entry lookup table used to drive per-instance, per-voxel, and world-terrain material properties [S1]. It supports mixed opaque-frame and glass models (e.g., static sprites, animated clips, glass walls, water) [S1].
*   **Color Function Callbacks:** Closure-based callbacks replace global state to support constant, jittered, position-dependent, and texture-mapped color functions [S1].

**Algorithm Steps:**
1.  **Traversal:** The renderer performs a 3D-DDA traversal over the voxel grid, visiting voxels strictly in front-to-back order relative to the ray origin [S1].
2.  **Sample Evaluation:** For each visited voxel segment, evaluate the material properties and color.
    *   If using closure-based callbacks, invoke the specific color function (constant, jittered, position-dependent, or texture-mapped) [S1].
    *   Compute the volumetric opacity. The opacity is weighted by the ray's path length through the voxel, causing filled clouds or volumes to appear denser at their core [S1].
3.  **Compositing:** Accumulate color and transmittance using front-to-back alpha blending.
    *   Update transmittance: $T_{new} = T_{old} \times (1 - \alpha)$.
    *   Update color: $\mathbf{C}_{new} = \mathbf{C}_{old} + T_{old} \times \alpha \times \mathbf{c}$, where $\mathbf{c}$ is the sample color and $\alpha$ is the sample opacity.
4.  **Termination:** Stop when the ray exits the voxel grid or transmittance falls below a threshold.

**Formulas:**
The core compositing equation for a single sample is:
$$ \mathbf{C} = \mathbf{C} + T \cdot \alpha \cdot \mathbf{c} $$
$$ T = T \cdot (1 - \alpha) $$

For volumetric density $\sigma$ and path length $\Delta t$, the opacity $\alpha$ is typically derived via Beer–Lambert law:
$$ \alpha = 1 - \exp(-\sigma \cdot \Delta t) $$
This weighting ensures that longer path lengths through dense media result in higher opacity [S1].

**Parameters:**
*   **Material Palette Size:** 256 entries [S1].
*   **Color Function Types:** Constant, jittered, position-dependent, texture-mapped [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Standard floating-point precision is sufficient for the accumulation, but transmittance decay must be handled carefully to avoid premature termination in very transparent scenes.
*   **Boundary Conditions:** The DDA traversal must correctly handle voxel boundaries to ensure no gaps or overlaps in the compositing order [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def transparent_voxel_compositing(ray_origin, ray_direction, voxel_grid, material_palette):
    # Initialize compositing state
    accumulated_color = vec3(0.0, 0.0, 0.0)
    transmittance = 1.0
    
    # Perform front-to-back DDA traversal
    # This part relies on the parent Per-pixel 3D-DDA Renderer logic
    # which yields voxels in strict front-to-back order.
    for voxel in voxel_grid.traverse_front_to_back(ray_origin, ray_direction):
        if transmittance < 1e-4:
            break
            
        # Get material properties from the 256-entry palette
        material = material_palette[voxel.material_id]
        
        # Evaluate color and opacity
        # The color function is a closure-based callback
        color = material.color_function(voxel.position, ray_direction)
        
        # Compute opacity weighted by ray path length through the voxel
        path_length = voxel.segment_length(ray_origin, ray_direction)
        density = material.density
        opacity = 1.0 - exp(-density * path_length)
        
        # Front-to-back alpha blending
        accumulated_color += transmittance * opacity * color
        transmittance *= (1.0 - opacity)
        
    return accumulated_color
```

## Subtopics

### Sparse-Voxel Alpha Blending
A method to compute sample opacity and accumulate color/depth/transmittance for a ray passing through sparse voxels, using trilinear interpolation of geometry and exponential-linear activation for density [S2, unofficial]. Unlike NeRF, which stores every sample for back-propagation, SpaRV differentiates the compositing equation in a custom CUDA kernel where memory grows with the number of rays rather than samples [S2, unofficial].

**Steps:**
1.  **Query:** For a point $\mathbf{p}$ inside a voxel, obtain density by trilinear interpolation $\text{interp}(\mathbf{v}_{\text{geo}}, \mathbf{p})$ followed by exponential-linear activation [S2, unofficial].
2.  **Discretization:** Each ray–voxel intersection segment is discretized into $K$ midpoint samples. The $k$-th sample in the $i$-th intersected voxel is at $\mathbf{x}_{i,k}$ with spacing $\Delta t_i$ [S2, unofficial].
3.  **Color Evaluation:** Evaluate view-dependent color $\mathbf{c}_{i,k}$ at $\mathbf{x}_{i,k}$. The DC component ($\text{SH}_0$) is trilinearly interpolated from vertex-stored coefficients, while higher-order SH coefficients are shared by the voxel. Combine via spherical harmonic evaluation [S2, unofficial].
4.  **Accumulation:** Accumulate color using front-to-back alpha blending [S2, unofficial].

**Formulas:**
Sample opacity:
$$ \alpha_{i,k} = 1 - \exp \left( -\Delta t_i \exp \text{lin}(\text{interp}(\mathbf{v}_{\text{geo}_i}, \mathbf{x}_{i,k})) \right) $$
Accumulated color:
$$ \mathbf{C} = \sum_{i=1}^N \sum_{k=1}^K T_{i,k} \alpha_{i,k} \mathbf{c}_{i,k} $$
where $T_{i,k} = \prod_{(j,l) \prec (i,k)} (1 - \alpha_{j,l})$ and $\prec$ denotes ray-marching sampling order [S2, unofficial].

**Parameters:**
*   $N$: Number of intersected voxels along the ray.
*   $K$: Number of midpoint samples per voxel segment [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels intersected by the ray. The front-to-back traversal ensures that each voxel is visited once.
*   **Memory Complexity:** Constant per pixel, as only the accumulated color and transmittance are stored. The material palette is a fixed 256-entry structure [S1].
*   **Strengths:** Eliminates the overhead of depth sorting and OIT buffers. Supports complex volumetric effects (smoke, fire, glass) with a single pass. Closure-based shading allows flexible material definitions without global state [S1].
*   **Limitations:** Requires strict front-to-back traversal order, which is naturally provided by the DDA but may be complex to implement for non-grid-based transparent objects. Performance depends on the density of the voxel grid and the number of samples per voxel in sparse variants [S2, unofficial].
*   **When to use:** Ideal for real-time rendering of volumetric scenes (clouds, fog, fire) and transparent objects (glass, water) where order-independent transparency is not required due to the spatial coherence of the voxel grid.

## Background
Transparent Voxel Compositing is implemented in the `roxlap` engine, specifically in the CPU renderer which is feature-complete with voxel terrain, animated sprites, and world-voxel lighting [S1]. It builds on the Per-pixel 3D-DDA Renderer to provide a clean-room alternative to column-coherent raycasting [S1]. The method supports runtime voxel editing (carve/fill spans, cubes, rectangles, spheres) and is byte-equality validated against `voxlap` C's `setspans` [S1]. The 256-entry material palette and closure-based color callbacks were introduced to replace the global-state dance of the original engine [S1].

## Sources
[S1] roxlap-cli 0.32.0, https://docs.rs/crate/roxlap-cli/latest, fetched 2026-10-29
[S2] SpaRV: Sparse Radiance Voxels for Scalable Geometrically Consistent LiDAR-Visual Mapping and Rendering, https://doi.org/10.48550/arxiv.2610.25665, fetched 2026-10-26 (unofficial)
