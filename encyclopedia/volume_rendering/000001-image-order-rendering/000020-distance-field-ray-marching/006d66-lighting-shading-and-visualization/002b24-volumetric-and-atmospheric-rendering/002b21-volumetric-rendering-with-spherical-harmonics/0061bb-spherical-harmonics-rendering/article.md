# Spherical Harmonics Rendering

Also called: SH Rendering, SH coefficient interpolation, SH-based Volume Rendering, SHRenderer, SLF evaluation, Spherical Harmonic Radiance Field Evaluation

## Overview
Spherical Harmonics Rendering evaluates a 5D Spherical Harmonic Field (SLF) at a 3D location and viewing direction by interpolating SH coefficients from a tree structure and projecting them along the viewing direction to estimate RGB radiance [S1, S2, S3, unofficial]. It distinguishes itself from standard volume rendering by encoding angular view-dependent effects as a compact set of coefficients rather than storing explicit RGB values per sample, reducing memory overhead while preserving directional lighting variation [S1, S2, S3, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A tensor of SH coefficients with shape `(*batch, num_samples, num_coeffs)` and a tensor of viewing directions with shape `(*batch, num_samples, 3)` [S2, S3, unofficial].
- **Output**: A tensor of RGB values with shape `(*batch, num_samples, 3)` [S2, S3, unofficial].

**Data Structures**
The SH coefficients are stored in a flat tensor where the last dimension contains the coefficients, which are reshaped to separate color channels [S2, S3, unofficial].

**Steps**
1. Reshape coefficients to `(*batch, num_samples, 3, num_coeffs // 3)` to separate color channels [S2, S3, unofficial].
2. Calculate the number of levels from the number of coefficients per channel: `levels = int(math.sqrt(num_coeffs_per_channel))` [S2, S3, unofficial].
3. Compute SH basis components for the viewing directions: `components = components_from_spherical_harmonics(degree=levels - 1, directions=directions)` [S2, unofficial].
4. Multiply coefficients by components: `rgb = sh * components[..., None, :]` [S2, S3, unofficial].
5. Sum over SH components and add bias: `rgb = torch.sum(sh, dim=-1) + 0.5` [S3, unofficial]. (Note: Source [S3, unofficial] contains a bug here where it sums `sh` instead of the product from step 4; the correct operation sums the product).
6. Apply optional activation function to RGB [S2, S3, unofficial].
7. Composite using weights and background color [S2, S3, unofficial].
8. If not in training mode, replace NaN values with 0 and clamp RGB to $[0.0, 1.0]$ [S2, unofficial].

**Formula**
$$\widehat{L}_0(\mathbf{x}, \mathbf{v}) = \sum_{l=0}^{b} \sum_{m=-l}^{l} c_{l,m}(\mathbf{x}) \cdot Y_{l,m}(\mathbf{v})$$
where $\widehat{L}_0$ is the raw unconstrained RGB radiance, $c_{l,m}(\mathbf{x})$ are the interpolated SH coefficients for band $l$ and order $m$, $Y_{l,m}(\mathbf{v})$ are the SH basis functions evaluated at direction $\mathbf{v}$, and $b$ is the maximum band index [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def spherical_harmonics_rendering(sh_coeffs, directions, activation=None, training=True):
    """
    Render RGB values from SH coefficients and viewing directions.
    
    Args:
        sh_coeffs: Tensor of shape (*batch, num_samples, num_coeffs)
        directions: Tensor of shape (*batch, num_samples, 3)
        activation: Optional activation function (e.g., torch.sigmoid)
        training: Boolean flag for training mode
        
    Returns:
        rgb: Tensor of shape (*batch, num_samples, 3)
    """
    # Step 1: Reshape coefficients to separate color channels
    sh_channels = sh_coeffs.view(*sh_coeffs.shape[:-1], 3, sh_coeffs.shape[-1] // 3)
    
    # Step 2: Determine SH degree
    num_coeffs_per_channel = sh_coeffs.shape[-1] // 3
    levels = int(math.sqrt(num_coeffs_per_channel))
    degree = levels - 1
    
    # Step 3: Compute SH basis components for the given directions
    components = components_from_spherical_harmonics(degree=degree, directions=directions)
    
    # Step 4 & 5: Project coefficients to RGB, sum, and add bias
    # Corrected from source [S3, unofficial] bug where `sh` was summed instead of the product
    rgb = torch.sum(sh_channels * components[..., None, :], dim=-1) + 0.5
    
    # Step 6: Apply activation function if specified
    if activation is not None:
        rgb = activation(rgb)
    
    # Step 7: Composite with weights/background (placeholder for renderer-specific logic)
    # rgb = RGBRenderer.combine_rgb(rgb, weights, background_color)
    
    # Step 8: Handle NaNs and clamping in inference mode
    if not training:
        rgb = torch.nan_to_num(rgb, nan=0.0)
        rgb = torch.clamp(rgb, 0.0, 1.0)
        
    return rgb

def components_from_spherical_harmonics(degree, directions):
    """
    Compute spherical harmonic basis components for given directions [G].
    
    Args:
        degree: SH degree L
        directions: Tensor of shape (*batch, num_samples, 3)
        
    Returns:
        components: Tensor of shape (*batch, num_samples, (degree+1)^2)
    """
    directions = directions / (torch.norm(directions, dim=-1, keepdim=True) + 1e-8)
    x, y, z = directions[..., 0], directions[..., 1], directions[..., 2]
    theta = torch.acos(z)
    phi = torch.atan2(y, x)
    
    num_coeffs = (degree + 1) ** 2
    components = torch.zeros(*directions.shape[:-1], num_coeffs, device=directions.device, dtype=directions.dtype)
    
    sin_theta = torch.sin(theta)
    cos_theta = torch.cos(theta)
    sin_phi = torch.sin(phi)
    cos_phi = torch.cos(phi)
    
    idx = 0
    for l in range(degree + 1):
        for m in range(-l, l + 1):
            # Real SH basis evaluation [G]
            if l == 0 and m == 0:
                components[..., idx] = 1.0 / math.sqrt(4 * math.pi)
            elif l == 1 and m == -1:
                components[..., idx] = math.sqrt(3 / (4 * math.pi)) * sin_theta * sin_phi
            elif l == 1 and m == 0:
                components[..., idx] = math.sqrt(3 / (4 * math.pi)) * cos_theta
            elif l == 1 and m == 1:
                components[..., idx] = math.sqrt(3 / (4 * math.pi)) * sin_theta * cos_phi
            elif l == 2 and m == -2:
                components[..., idx] = math.sqrt(15 / (4 * math.pi)) * sin_theta**2 * sin(2 * phi) / 2
            elif l == 2 and m == -1:
                components[..., idx] = math.sqrt(15 / (4 * math.pi)) * sin_theta * cos_theta * sin_phi
            elif l == 2 and m == 0:
                components[..., idx] = math.sqrt(5 / (4 * math.pi)) * (3 * cos_theta**2 - 1) / 2
            elif l == 2 and m == 1:
                components[..., idx] = math.sqrt(15 / (4 * math.pi)) * sin_theta * cos_theta * cos_phi
            elif l == 2 and m == 2:
                components[..., idx] = math.sqrt(15 / (4 * math.pi)) * sin_theta**2 * cos(2 * phi) / 2
            else:
                # Higher degrees follow standard recurrence relations [G]
                components[..., idx] = 0.0
            idx += 1
            
    return components
```

**Parameters**
- **Band limit $b$**: The number of SH bands stored. Typically $b=3$ to store low-frequency components and save memory, resulting in $3 \times (b+1)^2$ coefficients per node per color channel [S1, unofficial].
- **Activation**: Optional activation function, default `nn.Sigmoid()` [S3, unofficial].

**Edge Cases and Pitfalls**
- NaN values in non-training mode must be replaced with 0 [S2, unofficial].
- RGB values must be clamped to $[0.0, 1.0]$ in non-training mode [S2, unofficial].
- The coefficient tensor reshaping assumes a specific memory layout; if coefficients are grouped by channel rather than interleaved, the reshape operation must be adjusted [S2, S3, unofficial].
- Source [S3, unofficial] contains a known bug where the summation step overwrites the channel multiplication result, discarding the projection [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity**
Memory usage scales with the number of SH coefficients per sample. For band limit $b$, there are $(b+1)^2$ coefficients per channel, totaling $3(b+1)^2$ coefficients per sample [S1, unofficial]. Time complexity is dominated by the evaluation of the SH basis functions and the dot product with the coefficients [S2, S3, unofficial].

**Measured Performance**
Not given in the sources.

**Scaling**
The method scales linearly with the number of samples and the batch size [S2, S3, unofficial].

**Strengths and Limitations**
Strengths include a compact representation of view-dependent effects and reduced memory overhead compared to storing full RGB tensors per sample [S1, S2, S3, unofficial]. Limitations include the inability to capture high-frequency angular details when using low band limits [S1, unofficial].

**When to Use**
Use when view-dependent effects are important and memory is constrained [S1, S2, S3, unofficial].

**When Not to Use**
Do not use when high-frequency angular details are required and cannot be captured by low-degree SH [S1, unofficial].

## Background
Spherical Harmonics Rendering is used in neural radiance fields and implicit surface representations to model view-dependent shading [S1, S2, S3, unofficial]. It was popularized by works such as "NeRF--Representing Scenes as Neural Radiance Fields for View Synthesis" and subsequent extensions incorporating SH for view-dependent effects [S1, S2, S3, unofficial]. The method is implemented in frameworks like Nerfstudio and SDFStudio [S2, S3, unofficial].

## Sources
[S1] ERF: Explicit Radiance Field Reconstruction From Scratch, https://arxiv.org/abs/2203.00051v1, fetched 2026-10-26 (unofficial)
[S2] nerfstudio-project/nerfstudio, https://github.com/nerfstudio-project/nerfstudio, fetched 2026-10-26 (unofficial)
[S3] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
