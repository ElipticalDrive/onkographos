# Content-Aware Texturing for Gaussian Splatting

Also called: Content-Aware Texturing, Ours

## Overview
This method augments 2D Gaussian primitives with spatially varying texture maps to capture high-frequency appearance details that fixed spherical harmonics cannot represent [S1]. It distinguishes itself from sibling methods by dynamically allocating texture resolution based on local image content and error, avoiding the convergence issues associated with cloning-based densification [S1]. Unlike [GSTex](12924-gstex.md) which uses a fixed texel budget on pretrained primitives, this approach optimizes primitives and textures from scratch, and unlike [BBSplat](14740-color-and-appearance-modeling.md) which uses a fixed texel-to-primitive ratio, it adapts texel size to scene frequency [S1].

## How it works
**Inputs:** A set of 2D Gaussian primitives $\{G_i\}$, each defined by mean $\boldsymbol{\mu}_i$, scale $\boldsymbol{\sigma}_i$, rotation $\mathbf{R}_i$, opacity $\alpha_i$, and spherical harmonic coefficients for color $\mathbf{c}_{SH}$. Input views $\Pi$ with ground truth images.
**Outputs:** Optimized primitives with dynamic texture maps $T_i$ and parameters.

**Data Structures:**
*   **Primitive Parameters:** 59 floats per primitive [S1].
    *   2DGS base: 3 (pos) + 2 (scale) + 4 (rot) + 1 (opacity) + 48 (SH color) = 58 params [S1].
    *   Additional: 1 float for texel size ratio $t2pr_i$ [S1].
*   **Texture Map $T_i$:** Stored in a custom "jagged" tensor structure to handle varying resolutions [S1].
    *   Format: RGB offsets. 3 floats per texel [S1].
    *   Values constrained to $[-1, 1]$ via sigmoid activation on unactivated features $c'_{Ti}$: $c_{Ti} = 2\sigma(c'_{Ti}) - 1$ [S1].
*   **Coordinate Systems:**
    *   $p_w$: Intersection point in world space.
    *   $p_{w0} = p_w - \boldsymbol{\mu}_i$: World space centered on primitive.
    *   $p_l = \mathbf{R}_i^{-1} p_{w0}$: Local axis-aligned space ($p_l \in \mathbb{R}^2$) [S1].
    *   $p_c = \mathbf{S}_i^{-1} p_l$: Local normalized/canonical space, where $\mathbf{S}_i = \text{diag}(\boldsymbol{\sigma}_i)$ [S1].

**Rendering:**
The color of a ray $\mathbf{r}$ is computed by alpha blending [S1]:
$$ C(\mathbf{r}) = \sum_i w_i(p_i) c_i(\mathbf{d}) = \sum_i T_i o_i G_i(p_i) c_i(\mathbf{d}) $$
where $T_i = \prod_{j<i} (1 - o_j G_j(p_j))$ is the transmittance, $G_i(p_i) = \exp(-\frac{1}{2} p_i^\top p_i)$ is the Gaussian density in canonical space, and $c_i(\mathbf{d})$ is the primitive color [S1].

The primitive color combines spherical harmonics and texture offset [S1]:
$$ c_i = \text{SH}(\mathbf{d}) + \text{bilerp}(T_i, u) $$
where $u$ is the UV coordinate in texture space [S1]. To prevent texture stretching during optimization, $u$ is defined in world space units rather than canonical space [S1]:
$$ u = \frac{p_l}{k_i} + T_{\text{offset}} $$
Here, $k_i$ is the texel size in world space units (same units as scale $\boldsymbol{\sigma}$), and $T_{\text{offset}}$ centers the texture map [S1]. The texel size $k_i$ is derived from the minimum pixel size $k_{\min}$ and the primitive's texel-to-pixel ratio $t2pr_i$ [S1]:
$$ k_i = k_{\min} \cdot t2pr_i $$
$t2pr_i$ takes values that are powers of two, with a lower bound of 1 [S1].

**Training Loss:**
The total loss $L$ is the sum of RGB, texture sparsity, and opacity regularization terms [S1]:
$$ L = L_{\text{RGB}} + L_{\text{texture}} + L_{\text{opacity}} $$
1.  **RGB Loss:** Weighted L1 and SSIM [S1]:
    $$ L_{\text{RGB}} = (1 - \lambda_{\text{SSIM}}) L_1 + \lambda_{\text{SSIM}} L_{\text{SSIM}} $$
    Typical $\lambda_{\text{SSIM}} = 0.2$ [S1].
2.  **Texture Sparsity Loss:** Pushes texel values toward zero to allow downsampling [S1]:
    $$ L_{\text{texture}} = \lambda_{\text{texture}} \sum_i |c_{Ti}| $$
3.  **Opacity Regularization:** Incentivizes low-contribution primitives to disappear [S1]:
    $$ L_{\text{opacity}} = \lambda_{\text{opacity}} \frac{1}{N} \sum_i o_i $$

**Primitive Management and Densification:**
Cloning is avoided because superimposing multiple textured primitives causes convergence difficulties [S1]. Instead, splitting is used for geometric error, and texture resolution is adjusted for appearance error [S1].

1.  **Error Calculation:**
    For each primitive $i$ and view $\pi$, compute per-pixel error $E^\pi_i$ weighted by primitive contribution $w^\pi_i(r)$ [S1]:
    $$ E^\pi_i = \sum_{r \in P_i} E^\pi(r) w^\pi_i(r) $$
    where $P_i$ is the set of pixels covered by primitive $i$ [S1]. The total error $E_i$ is the weighted sum across all views $\Pi$ [S1]:
    $$ E_i = \frac{\sum_{\pi \in \Pi} E^\pi_i w^\pi_i}{\sum_{\pi \in \Pi} w^\pi_i}, \quad w^\pi_i = \sum_{r \in P_i} w^\pi_i(r) $$
    This differs from [RPK24] by using a weighted sum instead of maximum error [S1].

2.  **Upscaling (Appearance Error):**
    Select the top 10% of primitives with highest $E_i$ [S1].
    *   Condition: Only upscale if $t2pr_i \ge 2$ [S1].
    *   Action: Halve $t2pr_i$ (increasing texture resolution by factor of 4) [S1].

3.  **Downscaling (Low Frequency):**
    Compute appearance error $E_d$ by comparing original texture $T_{\text{orig}}$ to a low-pass filtered version $T_{\text{lowpass}}$, weighted by Gaussian falloff [S1]:
    $$ E_d = \frac{\sum_p G(p)(T_{\text{orig}}(p) - T_{\text{lowpass}}(p))}{\sum_p G(p)} $$
    If $E_d < \tau_{ds}$, double $t2pr_i$ (reducing texel count by 4) [S1].

4.  **Splitting (Geometric Error):**
    If a primitive in the top 10% error list has texture resolution exceeding threshold $\tau_{tr}$, it is split [S1].
    *   Action: Replace primitive with two new primitives.
    *   Displacement: Each displaced by $\pm 1$ standard deviation along the axis.
    *   Properties: New primitives have half the scale in the corresponding axis, half the texture resolution, and $G(1)$ times the opacity [S1].
    *   Texture Init: New texture maps are created by sampling the original point at the respective locations [S1].

**Pitfalls and Edge Cases:**
*   **Texture Stretching:** Naive mapping using canonical space causes stretching and local minima [S1]. The world-space mapping $u$ prevents this [S1].
*   **Primitive Growth:** If a primitive grows outside its allocated texture grid (due to lack of allocation or hitting the 256 limit), zero-padding is used. Since texels are offsets, this reduces to the DC color [S1].
*   **Memory Limit:** Hard limit of 256 texture resolution to avoid excessive memory usage [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, primitives, t2pr_threshold=2.0):
    """
    Renders a single ray using content-aware textured 2D Gaussians.
    """
    # 1. Sort primitives by depth along ray
    sorted_prims = sort_primitives_by_depth(primitives, ray_origin, ray_direction)
    
    accumulated_transmittance = 1.0
    accumulated_color = [0.0, 0.0, 0.0]
    
    for prim in sorted_prims:
        # 2. Compute intersection in camera space
        # Note: Implementation uses Eq 1 directly in camera space, not R2->R2 transform
        intersection = compute_intersection_2d_gaussian(prim, ray_origin, ray_direction)
        
        if intersection is None:
            continue
            
        p_w = intersection.point
        p_l = transform_to_local_space(p_w, prim.mu, prim.R)
        
        # 3. Compute Gaussian weight
        p_c = transform_to_canonical_space(p_l, prim.sigma)
        gaussian_weight = exp(-0.5 * dot(p_c, p_c))
        
        # 4. Compute color
        # SH color
        sh_color = evaluate_spherical_harmonics(prim.sh_coeffs, ray_direction)
        
        # Texture offset
        # k_i is texel size in world space units
        k_i = prim.k_min * prim.t2pr
        u = (p_l / k_i) + prim.t_offset
        texel_offset = bilerp(prim.texture_map, u)
        
        c_i = sh_color + texel_offset
        
        # 5. Alpha blend
        opacity = prim.opacity * gaussian_weight
        contribution = accumulated_transmittance * opacity * c_i
        accumulated_color += contribution
        accumulated_transmittance *= (1.0 - opacity)
        
        if accumulated_transmittance < 1e-4:
            break
            
    return accumulated_color

def manage_primitives(primitives, error_images, views, tau_ds=0.01, tau_tr=128):
    """
    Adjusts texture resolution and splits primitives based on error.
    """
    # 1. Compute per-primitive error
    errors = []
    for prim in primitives:
        prim_error = 0.0
        total_weight = 0.0
        for view in views:
            # Compute weighted error for this view
            view_error = 0.0
            view_weight = 0.0
            for pixel in prim.covered_pixels[view]:
                w = prim.contribution_to_pixel(pixel)
                e = error_images[view][pixel]
                view_error += e * w
                view_weight += w
            prim_error += view_error
            total_weight += view_weight
        
        if total_weight > 0:
            avg_error = prim_error / total_weight
        else:
            avg_error = 0.0
        errors.append((prim, avg_error))
    
    # 2. Sort by error
    errors.sort(key=lambda x: x[1], reverse=True)
    top_10_percent = errors[:len(errors)//10]
    
    # 3. Upscale high error primitives
    for prim, err in top_10_percent:
        if prim.t2pr >= t2pr_threshold:
            prim.t2pr /= 2.0
            
    # 4. Downscale low error primitives
    for prim in primitives:
        # Compute E_d (appearance error)
        E_d = compute_appearance_error(prim)
        if E_d < tau_ds:
            prim.t2pr *= 2.0
            
    # 5. Split primitives with high error and high texture resolution
    for prim, err in top_10_percent:
        if prim.t2pr > tau_tr:
            split_primitive(prim)
```

## Operating characteristics
*   **Complexity:** Training is 1.5-2x slower and rendering is 25% slower compared to [2DGS](9910-inverse-multiquadric-splatting.md) due to per-ray texture querying overhead [S1].
*   **Memory:** Uses a "jagged" tensor structure for textures. Total parameters depend on dynamic texel allocation.
*   **Performance (DeepBlending):** SSIM 0.907, PSNR 30.03, LPIPS 0.303, 222K points, 21.6M texels, 78.1M params, 70 FPS [S1].
*   **Performance (Mip-Nerf-360):** SSIM 0.795, PSNR 27.00, LPIPS 0.263, 218K points, 46.6M texels, 152.6M params, 67 FPS [S1].
*   **Performance (Tanks&Temples):** SSIM 0.835, PSNR 23.43, LPIPS 0.225, 164K points, 10.4M texels, 41.1M params, 121 FPS [S1].
*   **Strengths:** Significantly fewer parameters than [GSTex](12924-gstex.md) and [BBSplat](14740-color-and-appearance-modeling.md) while achieving competitive or superior quality [S1]. Adapts to scene content frequency [S1].
*   **Limitations:** Slower training and rendering than base 2DGS [S1]. Cloning-based densification is incompatible [S1].

## Background
The method was introduced by Papantonakis, Kopanas, Durand, and Drettakis in 2025 [S1]. It builds on the 2D Gaussian Splatting representation [S1]. It addresses the limitation of fixed spherical harmonics in capturing high-frequency details and the inefficiency of fixed texel budgets in methods like [GSTex](12924-gstex.md) [S1].

## Sources
[S1] Content-Aware Texturing for Gaussian Splatting, https://doi.org/10.2312/sr.20251190, fetched 2026-10-26
