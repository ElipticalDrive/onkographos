# LOS-integration for visualization

Also called: LUT-based Line-of-Sight Integration, column_ne integration, emission integration, ewt_temp integration

## Overview
LOS-integration for visualization computes physical scalar quantities (electron column density, integrated emissivity, or emission-weighted temperature) along camera rays by integrating interpolated field values. It differs from standard direct volume rendering by outputting integrated physical metrics rather than a composited RGB image, reusing the ray sampling and field query infrastructure of the forward pass [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: Ray origins `rays_o` $(N, 3)$, ray directions `rays_d` $(N, 3)$, sample start times `t_s` $(M,)$, sample end times `t_e` $(M,)$, and ray indices `ray_idx` $(M,)$ mapping samples to rays. Optional parameters: `uniform_samples` (int), `r_min_rsun` (float), `r_max_rsun` (float), `channels` (list/tuple), `quantity` (str: 'column_ne', 'emission', 'ewt_temp').
*   **Outputs**: A tensor `full` of shape $(N, C)$, where $C$ is the number of spectral channels (or 1 for scalar quantities). Invalid rays contain NaN.

**Data Structures**
*   **Field Queries**: The method queries a neural network or ground truth field for $\log_{10} n_e$ (electron density) and $\log_{10} T$ (temperature).
*   **Emissivity Coefficient Field**: A lookup or network `ccoef_field` that takes inputs $(\log_{10} n_e, \log_{10} T, \log_{10} r)$ and outputs emissivity coefficients `ccoef` in units of $\text{erg s}^{-1} \text{cm}^{-3} \text{sr}^{-1}$ [S1, unofficial].

**Parameters**
*   `rsun_to_cm`: Conversion factor, default $6.957 \times 10^{10}$ cm [S1, unofficial].
*   `aabb_scale`: Scaling factor from AABB space to physical units [S1, unofficial].
*   `wavelengths`: List of wavelengths in Angstroms for photon-to-erg conversion [S1, unofficial].
*   `clamp`: Boolean to clamp log-field values to configured min/max bounds [S1, unofficial].

**Steps**
1.  **Ray Discretization**: If `uniform_samples > 0`, generate uniform samples along rays to get `t_s`, `t_e`, `ray_idx`, and a validity mask `valid`. Otherwise, use pre-computed points [S1, unofficial].
2.  **Coordinate Conversion**: Compute sample midpoints $t_{mid} = 0.5(t_s + t_e)$. Convert to Cartesian coordinates in AABB space: $\mathbf{x}_{AABB} = \mathbf{o}_{valid}[\text{ray\_idx}] + t_{mid} \mathbf{d}_{valid}[\text{ray\_idx}]$. Convert to physical units: $\mathbf{x}_{xyz} = \mathbf{x}_{AABB} \times \text{aabb\_scale}$ [S1, unofficial].
3.  **Radial Masking**: Compute sample radius $r_s = \|\mathbf{x}_{xyz}\|$. Initialize a keep mask `keep_w` to 1. Apply radial constraints: if `r_min_rsun` is set, `keep_w *= (r_s >= r_min_rsun)`. If `r_max_rsun` is set, `keep_w *= (r_s <= r_max_rsun)`. This prevents artifacts from the saturated chromospheric base below $1 R_{sun}$ [S1, unofficial].
4.  **Field Retrieval**: Query the field for $\log n_e$ and $\log T$ at $\mathbf{x}_{xyz}$. Clamp values if `clamp` is True [S1, unofficial].
5.  **Integration Helper**: Define segment length in cm: $\Delta l = (t_e - t_s) \times \text{aabb\_scale} \times \text{rsun\_to\_cm}$. Define an integration function that accumulates values per ray using `index_add` [S1, unofficial].
6.  **Quantity Calculation**:
    *   **column_ne**: Compute electron density $n_e = 10^{\log n_e}$. Integrate: $I = \sum n_e \cdot \Delta l \cdot \text{keep\_w}$ [S1, unofficial].
    *   **emission**: Compute $r = \|\mathbf{x}_{xyz}\|$, $\log r = \log_{10} r$. Stack inputs $(\log n_e, \log T, \log r)$ into the emissivity coefficient field to get `ccoef`. Convert to intensity per arcsec$^2$:
        $$ \epsilon_I = \frac{\text{ccoef}}{P_{ergs}} \frac{1}{S_{arcsec}} $$
        where $P_{ergs} = \frac{h c}{\lambda \cdot 10^{-8}}$ (photon energy in ergs) and $S_{arcsec} = (\frac{180}{\pi} \cdot 3600)^2$ (steradians to arcsec$^2$). Integrate differential intensity $dI = \epsilon_I \cdot \Delta l \cdot \text{keep\_w}$ [S1, unofficial].
    *   **ewt_temp**: Compute numerator $N = \sum T \cdot \epsilon \cdot \Delta l \cdot \text{keep\_w}$ and denominator $D = \sum \epsilon \cdot \Delta l \cdot \text{keep\_w}$. Result: $T_{ewt} = N / \max(D, 10^{-30})$ [S1, unofficial].
7.  **Output Assembly**: Initialize output tensor of shape $(N, C)$ with NaN. Assign integrated values to valid rays [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def los_integration(rays_o, rays_d, t_s, t_e, ray_idx, valid, cfg):
    # 1. Coordinate Conversion
    t_mid = 0.5 * (t_s + t_e)
    # Select valid rays
    o_valid = rays_o[valid][ray_idx]
    d_valid = rays_d[valid][ray_idx]
    
    # AABB to Physical
    x_aabb = o_valid + t_mid[:, None] * d_valid
    x_xyz = x_aabb * cfg['aabb_scale']
    
    # 2. Radial Masking
    r_samp = torch.linalg.norm(x_xyz, dim=-1)
    keep_w = torch.ones_like(r_samp)
    if cfg.get('r_min_rsun') is not None:
        keep_w = keep_w * (r_samp >= cfg['r_min_rsun'])
    if cfg.get('r_max_rsun') is not None:
        keep_w = keep_w * (r_samp <= cfg['r_max_rsun'])
        
    # 3. Field Retrieval
    # Assume get_log_ne and get_log_temp are defined elsewhere
    log_ne = get_log_ne(x_xyz)
    log_temp = get_log_temp(x_xyz)
    if cfg.get('clamp'):
        log_ne = torch.clamp(log_ne, min=cfg['log_ne_min'], max=cfg['log_ne_max'])
        log_temp = torch.clamp(log_temp, min=cfg['log_temp_min'], max=cfg['log_temp_max'])
        
    # 4. Integration Setup
    seg_len_phys = (t_e - t_s) * cfg['aabb_scale'] * cfg.get('rsun_to_cm', 6.957e10)
    n_valid = valid.sum()
    out = torch.zeros((n_valid, cfg['num_channels']))
    
    # 5. Compute Quantity
    quantity = cfg['quantity']
    
    if quantity == 'column_ne':
        ne = torch.pow(10.0, log_ne)
        # Integrate: sum(ne * seg_len * keep_w) per ray
        out.index_add_(0, ray_idx, ne * seg_len_phys[:, None] * keep_w[:, None])
        
    elif quantity in ['emission', 'ewt_temp']:
        r = torch.linalg.norm(x_xyz, dim=-1)
        log_r = torch.log10(r)
        if cfg.get('clamp'):
            log_r = torch.clamp(log_r, min=cfg['log_r_min'], max=cfg['log_r_max'])
            
        # Query emissivity coefficient
        ntr_in = torch.stack([log_ne, log_temp, log_r], dim=1) # Shape (M, 3)
        ccoef_all = ccoef_field(ntr_in) # Shape (M, C)
        
        # Select channels if specified
        if cfg.get('channels') is not None:
            ccoef = ccoef_all[:, cfg['channels']]
        else:
            ccoef = ccoef_all
            
        # Unit Conversion
        # wavelengths in Angstroms
        wavelengths = torch.tensor(cfg['wavelengths'], device=cfg['device'])
        h = 6.62607015e-27
        c_light = 3e10
        ph2ergs = (h * c_light) / (wavelengths * 1e-8) # Shape (C,)
        sr2arcsec = (180.0 / np.pi)**2 * 3600.0**2
        
        # Convert ccoef (erg/s/cm^3/sr) to intensity per arcsec^2
        # epsI = ccoef / ph2ergs / sr2arcsec
        epsI = ccoef / ph2ergs[None, :] / sr2arcsec
        
        # Differential Intensity
        dI = epsI * seg_len_phys[:, None] * keep_w[:, None]
        
        if quantity == 'emission':
            out.index_add_(0, ray_idx, dI)
        elif quantity == 'ewt_temp':
            # Numerator: T * eps * dl
            T_lin = torch.pow(10.0, log_temp)[:, None]
            num = T_lin * dI
            out.index_add_(0, ray_idx, num)
            # Denominator: eps * dl
            den = dI
            # We need to store num and den separately or handle division later
            # For simplicity, assuming separate accumulation buffers or post-processing
            # Here we assume a helper that returns both or we modify the loop
            pass 
            
    # 6. Output Assembly
    full = torch.full((rays_o.shape[0], cfg['num_channels']), float('nan'), device=cfg['device'])
    full[valid] = out
    return full
```

**Edge Cases and Pitfalls**
*   **Chromospheric Artifacts**: Integrating below $1 R_{sun}$ causes saturated filled disk artifacts due to border-padding of the dense chromospheric base. Mitigated by `r_min_rsun` masking [S1, unofficial].
*   **Division by Zero**: In `ewt_temp`, the denominator (integrated emissivity) can be zero. Mitigated by `clamp_min(1e-30)` [S1, unofficial].

## Operating characteristics
*   **Complexity**: Linear in the number of samples $M$ and rays $N$. The dominant cost is the field query (neural network inference or interpolation) and the `index_add` accumulation.
*   **Performance**: Efficient on GPU due to vectorized tensor operations and `index_add`.
*   **Strengths**: Physically consistent integration; reuses existing ray sampling infrastructure; supports multiple physical quantities.
*   **Limitations**: Requires accurate field data (neural network or GT); sensitive to radial boundaries; not a direct image renderer.

## Background
Originated in the CoroNeRF project for visualizing solar corona physical fields [S1, unofficial]. It adapts standard line-of-sight integration techniques to output specific physical metrics (density, temperature, emission) rather than synthetic images.

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
