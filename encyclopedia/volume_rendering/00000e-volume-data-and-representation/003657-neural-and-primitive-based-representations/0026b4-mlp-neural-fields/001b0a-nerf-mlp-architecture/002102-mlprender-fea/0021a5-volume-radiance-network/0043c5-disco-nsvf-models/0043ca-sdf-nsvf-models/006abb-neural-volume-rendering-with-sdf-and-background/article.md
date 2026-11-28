# Neural Volume Rendering with SDF and Background

Also called: RISE-SDF Forward Pass

## Overview
This method performs neural volume rendering by decoupling geometry and appearance into separate Signed Distance Function (SDF) and radiance networks, as defined in the parent [SDF NSVF Models](11064-sdf-nsvf-models.md). It solves the problem of rendering high-fidelity novel views from sparse inputs by leveraging the geometric priors of SDFs for robust opacity estimation and surface normal computation, while explicitly compositing a learned or constant background to handle transparent or incomplete foreground objects. It differs from standard NeRF by using the SDF gradient for normals and a specific alpha derivation function, and from basic SDF rendering by including full volumetric integration and background compositing.

## How it works
The method executes a forward pass that queries geometry and texture networks, computes opacity based on SDF properties, accumulates volumetric fields, and composites the result with a background.

### Inputs and Outputs
**Inputs:**
- `rays`: A dictionary or object containing ray origins, directions, and potentially other metadata (e.g., `t_dirs` for direction, `dists` for interval lengths, `midpoints` for sample positions).
- `positions`: 3D coordinates of sample points along the rays.
- `config`: Configuration object containing flags like `learned_background` and parameters like `ray_chunk`.

**Outputs:**
- A dictionary containing:
  - `comp_rgb_full`: Final composited RGB color.
  - `opacity`: Accumulated opacity of the foreground.
  - `depth`: Accumulated depth.
  - `comp_normal`: Accumulated and normalized surface normal.
  - `comp_rgb`: Foreground-only RGB.
  - `comp_rgb_bg`: Background RGB (if learned).
  - `rays_valid`: Boolean mask indicating rays with non-zero opacity.
  - Auxiliary fields (during training): `sdf_samples`, `sdf_grad_samples`, `weights`, `points`, `intervals`, `ray_indices`.

### Data Structures
The implementation relies on the `sdf_radiance_field` structure from the parent [SDF NSVF Models](11064-sdf-nsvf-models.md). Key internal components include:
- `geometry`: An MLP that takes `positions` and returns `sdf` (scalar), `sdf_grad` (vector), and `feature` (vector).
- `texture`: An MLP that takes `feature`, `t_dirs` (ray direction), and `normal` to return `rgb`.
- `get_alpha`: A function mapping `sdf`, `normal`, `t_dirs`, and `dists` to opacity $\alpha$.
- `render_weight_from_alpha`: Computes ray weights $w_i$ from $\alpha_i$.
- `accumulate_along_rays`: Accumulates values weighted by $w_i$.

### Steps
1. **Query Geometry:** For each sample point along the ray, query the geometry network to obtain the signed distance, its gradient, and intermediate features.
   $$ sdf, sdf\_grad, feature = \text{geometry}(positions, \text{with\_grad}=True, \text{with\_feature}=True) $$
2. **Compute Normals:** Normalize the SDF gradient to obtain the surface normal.
   $$ normal = \frac{sdf\_grad}{\|sdf\_grad\|_2} $$
3. **Compute Opacity:** Calculate the opacity $\alpha$ for each sample using the SDF value, the computed normal, the ray direction, and the distance interval.
   $$ \alpha = \text{get\_alpha}(sdf, normal, t\_dirs, dists) $$
4. **Compute RGB:** Estimate the color using the intermediate features, ray direction, and surface normal.
   $$ rgb = \text{texture}(feature, t\_dirs, normal) $$
5. **Compute Ray Weights:** Calculate the weight $w_i$ for each sample based on its opacity and the transparency of preceding samples.
   $$ w_i = \alpha_i \prod_{j<i} (1-\alpha_j) $$
6. **Accumulate Fields:** Accumulate opacity, depth, RGB, and normal along the rays using the computed weights.
   $$ opacity = \sum w_i $$
   $$ depth = \sum w_i \cdot midpoints_i $$
   $$ comp\_rgb = \sum w_i \cdot rgb_i $$
   $$ comp\_normal = \sum w_i \cdot normal_i $$
7. **Normalize Composite Normal:** Normalize the accumulated normal vector.
   $$ comp\_normal = \frac{comp\_normal}{\|comp\_normal\|_2} $$
8. **Handle Background:**
   - If `config.learned_background` is true, query the background network for `comp_rgb_bg`.
   - Otherwise, use a constant background color expanded to match the RGB shape.
9. **Composite Final RGB:** Combine the foreground and background.
   $$ comp\_rgb\_full = comp\_rgb + comp\_rgb\_bg \cdot (1.0 - opacity) $$
10. **Construct Output:** Create the output dictionary. Set `rays_valid = opacity > 0`. If in training mode, include auxiliary data (`sdf_samples`, `sdf_grad_samples`, `weights`, `points`, `intervals`, `ray_indices`) for loss computation.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward(self, rays, training=False):
    # Chunked batching for inference to manage memory
    if not training and self.config.ray_chunk is not None:
        return self.chunk_batch(self.forward_, self.config.ray_chunk, True, rays)
    return self.forward_(rays, training)

def forward_(self, rays, training=False):
    # 1. Query Geometry
    # positions are derived from rays (origins, directions, intervals)
    sdf, sdf_grad, feature = self.geometry(rays.positions, with_grad=True, with_feature=True)
    
    # 2. Compute Normals
    normal = F.normalize(sdf_grad, p=2, dim=-1)
    
    # 3. Compute Opacity
    alpha = self.get_alpha(sdf, normal, rays.t_dirs, rays.dists)
    
    # 4. Compute RGB
    rgb = self.texture(feature, rays.t_dirs, normal)
    
    # 5. Compute Ray Weights
    weights, _ = render_weight_from_alpha(alpha, ray_indices=rays.ray_indices, n_rays=rays.n_rays)
    
    # 6. Accumulate Fields
    opacity = accumulate_along_rays(weights, values=None, ray_indices=rays.ray_indices, n_rays=rays.n_rays)
    depth = accumulate_along_rays(weights, values=rays.midpoints, ray_indices=rays.ray_indices, n_rays=rays.n_rays)
    comp_rgb = accumulate_along_rays(weights, values=rgb, ray_indices=rays.ray_indices, n_rays=rays.n_rays)
    comp_normal = accumulate_along_rays(weights, values=normal, ray_indices=rays.ray_indices, n_rays=rays.n_rays)
    
    # 7. Normalize Composite Normal
    comp_normal = F.normalize(comp_normal, p=2, dim=-1)
    
    # 8. Handle Background
    if self.config.learned_background:
        out_bg = self.forward_bg_(rays)
        comp_rgb_bg = out_bg['comp_rgb']
    else:
        # Constant background color (e.g., white or black)
        comp_rgb_bg = torch.ones_like(comp_rgb) * self.config.background_color
        
    # 9. Composite Final RGB
    comp_rgb_full = comp_rgb + comp_rgb_bg * (1.0 - opacity)
    
    # 10. Construct Output
    rays_valid = opacity > 0
    out = {
        'comp_rgb_full': comp_rgb_full,
        'opacity': opacity,
        'depth': depth,
        'comp_normal': comp_normal,
        'comp_rgb': comp_rgb,
        'comp_rgb_bg': comp_rgb_bg,
        'rays_valid': rays_valid
    }
    
    if training:
        out['sdf_samples'] = sdf
        out['sdf_grad_samples'] = sdf_grad
        out['weights'] = weights
        out['points'] = rays.positions
        out['intervals'] = rays.dists
        out['ray_indices'] = rays.ray_indices
        
    return out
```

### Parameters and Edge Cases
- **`learned_background`**: Boolean flag. If false, a constant color is used.
- **`ray_chunk`**: Integer specifying the number of rays to process in parallel during inference to prevent OOM errors.
- **Precision**: SDF gradients can be noisy near the surface; normalization helps stabilize normals.
- **Degenerate Input**: If `sdf_grad` is zero (e.g., inside a flat region or far from surface), normalization may produce NaNs. A small epsilon should be added to the denominator in normalization [G].
- **Background**: If the foreground is fully opaque, the background term vanishes. If the foreground is transparent, the background contributes proportionally to $(1.0 - opacity)$.

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of samples per ray and the number of rays. Chunked batching reduces peak memory usage at the cost of increased overhead.
- **Memory Complexity**: Proportional to the number of samples stored for gradients and features during training. Inference can be memory-efficient via chunking.
- **Strengths**: Robust geometry representation via SDF; explicit normal estimation; handles transparent objects and background compositing.
- **Limitations**: Requires careful tuning of the `get_alpha` function; SDF queries can be computationally expensive if the geometry network is deep; background learning adds complexity.
- **When to use**: When high-quality novel view synthesis is required with explicit geometry and normal outputs, and when background handling is critical (e.g., transparent objects, incomplete scans).
- **When not to use**: For simple opaque scenes where background is uniform or irrelevant, or when computational resources are extremely limited and a simpler NeRF variant suffices.

## Background
This method is part of the RISE-SDF framework, which extends Neural Surface Fields (NSVF) by integrating SDF-based geometry with neural radiance fields. It builds on the [SDF NSVF Models](11064-sdf-nsvf-models.md) architecture, which separates geometry and appearance. The approach addresses limitations of standard NeRF in representing sharp surfaces and estimating normals by leveraging the signed distance function.

## Sources
[S1] dehezhang2/RISE-SDF, https://github.com/dehezhang2/RISE-SDF, fetched 2026-10-27 (unofficial)
