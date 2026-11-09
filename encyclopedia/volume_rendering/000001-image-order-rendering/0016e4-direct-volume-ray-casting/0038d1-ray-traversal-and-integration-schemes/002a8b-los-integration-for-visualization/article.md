# LOS-integration for visualization

Also called: LUT-based Line-of-Sight Integration, column_ne integration, emission integration, ewt_temp integration

## Overview
LOS-integration for visualization computes scalar physical quantities along camera rays by integrating sampled field values, specifically designed for solar corona visualization [S1, unofficial]. It differs from standard volume ray casting by bypassing opacity-based compositing; instead, it directly accumulates emissivity, column density, or emission-weighted temperature using a lookup table (LUT) or neural network field to map input densities and temperatures to physical intensities [S1, unofficial]. This method is chosen when the goal is to visualize integrated physical properties (e.g., total electron column density) rather than photorealistic translucent rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origins `rays_o` $(N, 3)$, ray directions `rays_d` $(N, 3)$, sample start times `t_s` $(M,)$, sample end times `t_e` $(M,)$, ray indices `ray_idx` $(M,)$, and a validity mask `valid` $(N,)$ [S1, unofficial].
*   **Outputs:** A tensor `full` of shape $(N, C)$, where $N$ is the number of rays and $C$ is the number of spectral channels (or 1 for scalar quantities), containing the integrated result. Invalid rays contain `NaN` [S1, unofficial].

**Data Structures:**
*   **Field Queries:** The method queries scalar fields (log electron density `log_ne`, log temperature `log_temp`) either from ground truth (GT) data structures or a learned neural network field `self.ne_field`/`self.temp_field` [S1, unofficial].
*   **Emissivity Coefficient Field:** A field `ccoef_field` that takes stacked inputs $(\log n_e, \log T, \log r)$ and outputs emissivity coefficients per spectral channel [S1, unofficial].

**Parameters:**
*   `quantity`: String, one of `'column_ne'`, `'emission'`, or `'ewt_temp'` [S1, unofficial].
*   `uniform_samples`: Integer, number of uniform samples if uniform sampling is requested. Default 0 [S1, unofficial].
*   `r_min_rsun`, `r_max_rsun`: Floats or `None`, minimum and maximum radius in solar radii to restrict integration. Default `None` [S1, unofficial].
*   `channels`: List, tuple, int, or `None`. Spectral line indices to select. Default `None` (all channels) [S1, unofficial].
*   `rsun_to_cm`: Constant, default $6.957 \times 10^{10}$, converting solar radii to centimeters [S1, unofficial].
*   `wavelengths`: List of wavelengths in Angstroms, used for photon-to-ergs conversion [S1, unofficial].
*   `aabb_scale`: Scaling factor from AABB space to physical units [S1, unofficial].
*   `clamp`: Boolean, whether to clamp `log_ne` and `log_temp` to configured min/max bounds [S1, unofficial].

**Steps:**
1.  **Ray Preparation:** Convert `rays_o` and `rays_d` to the renderer's dtype and device [S1, unofficial].
2.  **Sampling:**
    *   If `uniform_samples > 0`, call `_sample_uniform_shell` to obtain `t_s`, `t_e`, `ray_idx`, and `valid` [S1, unofficial].
    *   Otherwise, call `_sample_points_along_rays` [S1, unofficial].
3.  **Coordinate Conversion:**
    *   Compute midpoints: $t_{mid} = 0.5 \times (t_s + t_e)$ [S1, unofficial].
    *   Compute Cartesian coordinates in AABB space: $\mathbf{x}_{AABB} = \text{rays\_o}[valid][ray\_idx] + t_{mid} \times \text{rays\_d}[valid][ray\_idx]$ [S1, unofficial].
    *   Scale to physical units: $\mathbf{x}_{xyz} = \mathbf{x}_{AABB} \times \text{aabb\_scale}$ [S1, unofficial].
4.  **Radial Masking:**
    *   Compute sample radius: $r_{samp} = \|\mathbf{x}_{xyz}\|$ [S1, unofficial].
    *   Initialize mask `keep_w` to ones [S1, unofficial].
    *   If `r_min_rsun` is not `None`, update `keep_w` by multiplying with $(r_{samp} \ge r_{min\_rsun})$ [S1, unofficial].
    *   If `r_max_rsun` is not `None`, update `keep_w` by multiplying with $(r_{samp} \le r_{max\_rsun})$ [S1, unofficial]. This prevents artifacts from the saturated chromospheric base [S1, unofficial].
5.  **Field Retrieval:**
    *   If `use_gt` and `self.ne_field` is not `None`, query `log_ne = self.ne_field(x_xyz)` and optionally `log_temp` [S1, unofficial].
    *   Else, call `self._predict_learned_fields(x_AABB)` to get `_, log_ne, log_temp` [S1, unofficial].
    *   If `log_temp` is `None` but `self.temp_field` exists, query it directly [S1, unofficial].
    *   If `clamp` is `True`, clamp `log_ne` and `log_temp` to configured bounds [S1, unofficial].
6.  **Integration Setup:**
    *   Compute physical segment length: $\text{seg\_len} = (t_e - t_s) \times \text{aabb\_scale} \times \text{rsun\_to\_cm}$ [S1, unofficial].
    *   Define helper `integ(vals)`: Initialize output tensor of size `valid.sum()`. Accumulate values using `out.index_add_(0, ray_idx, vals * seg_len * keep_w)` [S1, unofficial].
7.  **Quantity-Specific Integration:**
    *   **If `quantity == 'column_ne'`:**
        *   Compute per-ray column density: $\text{per\_ray} = \text{integ}(10^{\text{log\_ne}})$ [S1, unofficial].
    *   **If `quantity == 'emission'` or `'ewt_temp'`:**
        *   Compute radius $r = \|\mathbf{x}_{xyz}\|$ and $\log r = \log_{10}(r)$ [S1, unofficial].
        *   Clamp $\log r$ if `clamp` is `True` [S1, unofficial].
        *   Stack inputs: $\mathbf{ntr} = [\log n_e, \log T, \log r]^T$ (transposed to shape $(M, 3)$) [S1, unofficial].
        *   Query emissivity coefficients: $\mathbf{ccoef\_all} = \text{self.ccoef\_field}(\mathbf{ntr})$ [S1, unofficial].
        *   Select specified channels if provided [S1, unofficial].
        *   Sum over channels to get emissivity $\epsilon$ [S1, unofficial].
        *   **For `'emission'`:** $\text{per\_ray} = \text{integ}(\epsilon)$ [S1, unofficial].
        *   **For `'ewt_temp'`:**
            *   Numerator: $\text{num} = \text{integ}(10^{\text{log\_temp}} \times \epsilon)$ [S1, unofficial].
            *   Denominator: $\text{den} = \text{integ}(\epsilon)$ [S1, unofficial].
            *   Result: $\text{per\_ray} = \text{num} / \max(\text{den}, 10^{-30})$ [S1, unofficial].
8.  **Output Assembly:**
    *   Initialize `full` tensor of shape $(N, C)$ with `NaN` [S1, unofficial].
    *   Set `full[valid] = per_ray` [S1, unofficial].
    *   Return `full` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def los_integration(rays_o, rays_d, t_s, t_e, ray_idx, valid, cfg):
    # 1. Ray Preparation
    rays_o = rays_o.to(cfg.dtype, cfg.device)
    rays_d = rays_d.to(cfg.dtype, cfg.device)
    
    # 2. Sampling (Assuming samples already provided for this snippet)
    # If uniform_samples > 0: t_s, t_e, ray_idx, valid = _sample_uniform_shell(...)
    # Else: t_s, t_e, ray_idx, valid = _sample_points_along_rays(...)
    
    # 3. Coordinate Conversion
    t_mid = 0.5 * (t_s + t_e)
    # Gather valid rays
    valid_rays_o = rays_o[valid]
    valid_rays_d = rays_d[valid]
    # Map ray_idx to valid indices for gathering
    # Note: ray_idx usually maps sample index to original ray index.
    # We need to gather from valid_rays_o using the subset of ray_idx corresponding to valid rays.
    # Assuming ray_idx is already aligned or we filter it:
    valid_ray_indices = ray_idx[valid] # Simplified assumption based on source logic
    
    x_AABB = valid_rays_o[valid_ray_indices] + t_mid[..., None] * valid_rays_d[valid_ray_indices]
    x_xyz = x_AABB * cfg.aabb_scale
    
    # 4. Radial Masking
    r_samp = torch.linalg.vector_norm(x_xyz, dim=-1)
    keep_w = torch.ones_like(r_samp)
    
    if cfg.r_min_rsun is not None:
        keep_w = keep_w * (r_samp >= cfg.r_min_rsun)
    if cfg.r_max_rsun is not None:
        keep_w = keep_w * (r_samp <= cfg.r_max_rsun)
        
    # 5. Field Retrieval
    if cfg.use_gt and cfg.ne_field is not None:
        log_ne = cfg.ne_field(x_xyz)
        log_temp = cfg.temp_field(x_xyz) if cfg.temp_field is not None else None
    else:
        _, log_ne, log_temp = cfg._predict_learned_fields(x_AABB)
        if log_temp is None and cfg.temp_field is not None:
            log_temp = cfg.temp_field(x_xyz)
            
    if cfg.clamp:
        log_ne = torch.clamp(log_ne, min=cfg.min_log_ne, max=cfg.max_log_ne)
        if log_temp is not None:
            log_temp = torch.clamp(log_temp, min=cfg.min_log_temp, max=cfg.max_log_temp)
            
    # 6. Integration Setup
    seg_len = (t_e - t_s) * cfg.aabb_scale * cfg.rsun_to_cm
    
    def integ(vals):
        # vals shape: (M,)
        # out shape: (valid.sum(),)
        out = torch.zeros(valid.sum(), device=vals.device, dtype=vals.dtype)
        # Map global ray_idx to local valid indices? 
        # Source uses index_add with ray_idx. Assuming ray_idx is compatible with valid output size.
        # In practice, ray_idx often refers to original N rays. 
        # If valid is a mask, we usually gather valid indices.
        # The source code snippet: out.index_add_(0, ray_idx, vals * seg_len * keep_w)
        # This implies ray_idx is already filtered or mapped to the output buffer size.
        # We assume the helper handles the indexing correctly for the valid subset.
        out.index_add_(0, ray_idx, vals * seg_len * keep_w)
        return out

    # 7. Quantity-Specific Integration
    quantity = cfg.quantity
    if quantity == 'column_ne':
        per_ray = integ(torch.pow(10.0, log_ne))
        
    elif quantity in ['emission', 'ewt_temp']:
        r = torch.linalg.vector_norm(x_xyz, dim=-1)
        log_r = torch.log10(r)
        if cfg.clamp:
            log_r = torch.clamp(log_r, min=cfg.min_log_r, max=cfg.max_log_r)
            
        ntr = torch.stack((log_ne, log_temp, log_r), dim=-1) # Shape (M, 3)
        # Transpose to (M, 3) is already correct for row-wise input if ntr is (M,3)
        # Source says: Stack... then transpose. If stack dim=-1, it is (M,3). 
        # If stack dim=0, it is (3, M). Source: "Stack... ntr_in = torch.stack(..., axis=0) then transpose"
        # So ntr_in is (3, M) then transposed to (M, 3).
        ntr_in = torch.stack((log_ne, log_temp, log_r), axis=0) # (3, M)
        ntr_in = torch.transpose(ntr_in, 0, 1) # (M, 3)
        
        ccoef_all = cfg.ccoef_field(ntr_in) # (M, C)
        
        if cfg.channels is not None:
            ccoef_all = ccoef_all[:, cfg.channels]
            
        eps = torch.sum(ccoef_all, dim=-1) # (M,)
        
        # Unit conversion for emission
        # sr2arcsec = (180.0/np.pi)**2 * 3600.0**2
        # ph2ergs = 6.62607015e-27 * 3e10 / (wavelengths * 1e-8)
        # epsI = ccoef / ph2ergs / sr2arcsec
        # dI = epsI * seg_len_phys
        # For simplicity in pseudocode, we assume eps is already in correct intensity units or handled by field
        # The source notes mention converting ccoef to intensity. 
        # We will assume the field output or subsequent step handles the physical constant conversion
        # or that 'emission' here refers to the raw integrated coefficient if not explicitly converted.
        # However, note [S1, unofficial] explicitly mentions conversion. 
        # Let's assume eps is the emissivity in physical units required for integration.
        
        if quantity == 'emission':
            per_ray = integ(eps)
        else: # ewt_temp
            num = integ(torch.pow(10.0, log_temp) * eps)
            den = integ(eps)
            per_ray = num / torch.clamp_min(den, 1e-30)
            
    # 8. Output Assembly
    full = torch.full((rays_o.shape[0], per_ray.shape[-1] if quantity != 'column_ne' else 1), float('nan'), device=rays_o.device, dtype=rays_o.dtype)
    full[valid] = per_ray
    return full
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples $M$ and rays $N$, dominated by field queries and the `index_add` accumulation [S1, unofficial].
*   **Memory Complexity:** Linear with respect to $N$ and $M$ for storing ray data, intermediate fields, and the output tensor [S1, unofficial].
*   **Strengths:** Efficiently computes integrated physical quantities without the overhead of opacity compositing; supports multiple physical interpretations (density, emission, temperature) via a single framework [S1, unofficial].
*   **Limitations:** Requires accurate field data (GT or learned); sensitive to radial boundaries due to chromospheric artifacts if not masked [S1, unofficial].
*   **When to use:** When visualizing integrated solar corona properties like electron column density or total emission intensity [S1, unofficial].
*   **When not to use:** When photorealistic translucent rendering with depth and occlusion is required [S1, unofficial].

## Background
LOS-integration for visualization is implemented in the CoroNeRF project, which uses neural radiance fields for solar corona reconstruction [S1, unofficial]. It was developed to provide interpretable physical metrics from learned fields, allowing comparison with ground truth solar data [S1, unofficial].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
