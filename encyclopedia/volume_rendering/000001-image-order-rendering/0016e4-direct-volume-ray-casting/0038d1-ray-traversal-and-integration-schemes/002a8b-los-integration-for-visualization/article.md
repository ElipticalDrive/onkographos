# LOS-integration for visualization

Also called: LUT-based Line-of-Sight Integration, column_ne integration, emission integration, ewt_temp integration

## Overview
LOS-integration for visualization computes scalar physical quantities (electron column density, integrated emissivity, or emission-weighted temperature) along camera rays by integrating sampled neural network or ground truth field outputs. It differs from standard ray marching by explicitly handling unit conversions to physical consistency (cm, ergs, photons) and supporting spectral channel selection via a lookup table (LUT) for emissivity coefficients. It builds on [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md).

## How it works
**Inputs:**
- `rays_o`: Ray origins, shape `(N, 3)`.
- `rays_d`: Ray directions, shape `(N, 3)`.
- `t_s`: Start times, shape `(M,)`.
- `t_e`: End times, shape `(M,)`.
- `ray_idx`: Ray indices for samples, shape `(M,)`.
- `valid`: Boolean mask for valid rays, shape `(N,)`.
- `quantity`: String, one of `'column_ne'`, `'emission'`, `'ewt_temp'`.
- `uniform_samples`: Integer, number of uniform samples (default 0).
- `r_min_rsun`: Float or None, minimum radius in solar radii (default None).
- `r_max_rsun`: Float or None, maximum radius in solar radii (default None).
- `channels`: List, tuple, int, or None, spectral line indices to select (default None).
- `use_gt`: Boolean, whether to use ground truth fields.
- `clamp`: Boolean, whether to clamp log fields.

**Outputs:**
- `full`: Integrated values, shape `(N, C)` where `C` is the number of intensity channels. Contains NaN for invalid rays.

**Data Structures:**
- Fields are queried via interpolation (e.g., `self.ne_field(x_xyz)`).
- Emissivity coefficients are retrieved from a field `self.ccoef_field` indexed by stacked inputs `(log_ne, log_temp, log_r)`.

**Parameters:**
- `aabb_scale`: Scaling factor from AABB space to physical units.
- `rsun_to_cm`: Conversion constant, default `6.957e10`.
- `wavelengths`: List of wavelengths in Angstroms for photon-to-ergs conversion.

**Steps:**
1.  **Ray Preparation:** Convert `rays_o` and `rays_d` to the renderer's dtype and device.
2.  **Sampling:**
    - If `uniform_samples > 0`, call `_sample_uniform_shell` to obtain `t_s`, `t_e`, `ray_idx`, and `valid`.
    - Otherwise, call `_sample_points_along_rays`.
3.  **Coordinate Conversion:**
    - Compute midpoints: $t_{mid} = 0.5 \times (t_s + t_e)$.
    - Compute Cartesian coordinates in AABB space: $x_{AABB} = rays_o[valid][ray\_idx] + t_{mid} \times rays_d[valid][ray\_idx]$.
    - Scale to physical units: $x_{xyz} = x_{AABB} \times aabb\_scale$.
4.  **Radial Masking:**
    - Compute sample radius: $r_{samp} = \|x_{xyz}\|_2$.
    - Initialize mask: $keep\_w = \mathbf{1}$.
    - If `r_min_rsun` is not None, $keep\_w \leftarrow keep\_w \times (r_{samp} \ge r_{min\_rsun})$.
    - If `r_max_rsun` is not None, $keep\_w \leftarrow keep\_w \times (r_{samp} \le r_{max\_rsun})$.
    - This restricts integration to the physically-valid corona shell ($r \ge 1 R_{sun}$) to avoid saturated filled disk artifacts from chromospheric base padding [S1, unofficial].
5.  **Field Retrieval:**
    - If `use_gt` and `self.ne_field` exists, query $log\_ne = self.ne\_field(x_{xyz})$ and optionally $log\_temp = self.temp\_field(x_{xyz})$.
    - Else, call `self._predict_learned_fields(x_AABB)` to get $log\_ne$ and $log\_temp$.
    - If $log\_temp$ is None but `self.temp_field` exists, query it directly.
    - If `clamp` is True, clamp $log\_ne$ and $log\_temp$ using configured min/max bounds.
6.  **Segment Length:**
    - Compute physical segment length: $seg\_len = (t_e - t_s) \times aabb\_scale \times rsun\_to\_cm$.
7.  **Integration Helper:**
    - Define `integ(vals)`:
        - Initialize `out` tensor of size `valid.sum()`.
        - Perform `out.index_add_(0, ray_idx, vals * seg_len * keep_w)`.
8.  **Quantity-Specific Integration:**
    - **If `quantity == 'column_ne'`:**
        - $per\_ray = integ(10^{log\_ne})$.
    - **If `quantity == 'emission'` or `quantity == 'ewt_temp'`:**
        - Compute $r = \|x_{xyz}\|$ and $log\_r = \log_{10}(r)$. Clamp $log\_r$ if `clamp` is True.
        - Stack inputs: $ntr = \text{stack}(log\_ne, log\_temp, log\_r)$.
        - Query emissivity coefficients: $ccoef\_all = self.ccoef\_field(ntr)$.
        - Transpose $ccoef\_all$ to shape $(M, C)$.
        - If `channels` is specified, select those channels.
        - Sum over channels to get emissivity $\epsilon$.
        - **For `emission`:** $per\_ray = integ(\epsilon)$.
        - **For `ewt_temp`:**
            - $num = integ(10^{log\_temp} \times \epsilon)$.
            - $den = integ(\epsilon)$.
            - $per\_ray = num / \max(den, 10^{-30})$.
9.  **Finalization:**
    - Initialize `full` tensor of shape `(rays_o.shape[0], C)` with NaN.
    - Set `full[valid] = per_ray`.
    - Return `full`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def los_integration(rays_o, rays_d, t_s, t_e, ray_idx, valid, quantity, 
                    uniform_samples=0, r_min_rsun=None, r_max_rsun=None, 
                    channels=None, use_gt=False, clamp=False, 
                    aabb_scale=1.0, rsun_to_cm=6.957e10, wavelengths=None):
    # 1. Ray Preparation
    # (Assume rays_o, rays_d are already on correct device/dtype)
    
    # 2. Sampling
    if uniform_samples > 0:
        t_s, t_e, ray_idx, valid = sample_uniform_shell(rays_o, rays_d, uniform_samples)
    else:
        t_s, t_e, ray_idx, valid = sample_points_along_rays(rays_o, rays_d)
        
    # 3. Coordinate Conversion
    t_mid = 0.5 * (t_s + t_e)
    # Select valid rays for coordinate computation
    valid_rays_o = rays_o[valid]
    valid_rays_d = rays_d[valid]
    x_AABB = valid_rays_o[ray_idx] + t_mid[:, None] * valid_rays_d[ray_idx]
    x_xyz = x_AABB * aabb_scale
    
    # 4. Radial Masking
    r_samp = torch.linalg.vector_norm(x_xyz, dim=-1)
    keep_w = torch.ones_like(r_samp)
    if r_min_rsun is not None:
        keep_w = keep_w * (r_samp >= r_min_rsun)
    if r_max_rsun is not None:
        keep_w = keep_w * (r_samp <= r_max_rsun)
        
    # 5. Field Retrieval
    if use_gt and ne_field is not None:
        log_ne = ne_field(x_xyz)
        if temp_field is not None:
            log_temp = temp_field(x_xyz)
        else:
            log_temp = None
    else:
        _, log_ne, log_temp = predict_learned_fields(x_AABB)
        if log_temp is None and temp_field is not None:
            log_temp = temp_field(x_xyz)
            
    if clamp:
        log_ne = torch.clamp(log_ne, min=log_ne_min, max=log_ne_max)
        if log_temp is not None:
            log_temp = torch.clamp(log_temp, min=log_temp_min, max=log_temp_max)
            
    # 6. Segment Length
    seg_len = (t_e - t_s) * aabb_scale * rsun_to_cm
    
    # 7. Integration Helper
    def integ(vals):
        out = torch.zeros(valid.sum(), device=vals.device, dtype=vals.dtype)
        out.index_add_(0, ray_idx, vals * seg_len * keep_w)
        return out
        
    # 8. Quantity-Specific Integration
    if quantity == 'column_ne':
        per_ray = integ(torch.pow(10.0, log_ne))
    elif quantity in ('emission', 'ewt_temp'):
        r = torch.linalg.vector_norm(x_xyz, dim=-1)
        log_r = torch.log10(r)
        if clamp:
            log_r = torch.clamp(log_r, min=log_r_min, max=log_r_max)
            
        ntr = torch.stack([log_ne, log_temp, log_r], dim=-1)
        ccoef_all = ccoef_field(ntr)
        # Transpose to (M, C)
        ccoef_all = ccoef_all.transpose(0, 1) # Assuming ccoef_field returns (C, M) or similar layout requiring transpose
        
        if channels is not None:
            ccoef_all = ccoef_all[channels]
            
        eps = torch.sum(ccoef_all, dim=0)
        
        if quantity == 'emission':
            per_ray = integ(eps)
        elif quantity == 'ewt_temp':
            num = integ(torch.pow(10.0, log_temp) * eps)
            den = integ(eps)
            per_ray = num / torch.clamp_min(den, 1e-30)
    else:
        raise ValueError(f"Unknown quantity: {quantity}")
        
    # 9. Finalization
    full = torch.full((rays_o.shape[0], per_ray.shape[1] if quantity in ('emission', 'ewt_temp') else 1), float('nan'), device=rays_o.device, dtype=rays_o.dtype)
    full[valid] = per_ray
    return full
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples $M$ and rays $N$, dominated by field queries and integration.
- **Memory Complexity:** Linear with respect to $M$ and $N$.
- **Strengths:** Physically consistent unit conversion; supports spectral channel selection; handles radial cutoffs to prevent artifacts.
- **Limitations:** Requires accurate field data (neural network or GT); sensitive to radial boundaries.
- **When to use:** When physical quantities (column density, emissivity, temperature) are needed for visualization rather than just intensity.
- **When not to use:** When only visual appearance (color/intensity) is required without physical interpretation.

## Background
Originated in the CoroNeRF project [S1, unofficial]. It is used for visualizing solar corona properties such as electron density and temperature from neural radiance fields.

## Sources
[S1] CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
