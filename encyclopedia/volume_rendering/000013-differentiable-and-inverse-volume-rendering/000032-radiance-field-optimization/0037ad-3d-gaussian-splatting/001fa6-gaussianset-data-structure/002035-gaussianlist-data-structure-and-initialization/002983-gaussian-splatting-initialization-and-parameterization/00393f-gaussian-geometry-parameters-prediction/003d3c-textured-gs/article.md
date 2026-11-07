# Textured-GS

Also called: Gaussian Splatting with Spatially Defined Color and Opacity

## Overview
Textured-GS enhances the textural representation of individual 3D Gaussians by modeling spatially defined color and opacity variations across the Gaussian ellipsoidal surface using Spherical Harmonics (SH), without increasing the number of Gaussians or adding extra parameters beyond the existing SH framework [S1]. Unlike standard 3D Gaussian Splatting (3DGS) where each Gaussian retains a fixed opacity and shows a single color when viewed from a specific direction, Textured-GS enables color variation across the Gaussian ellipsoidal surface and opacity variations, allowing for deviations from the standard ellipsoidal shape such as 'unidirectional Gaussians' that are opaque on one side and transparent on the other [S1]. It differs from [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md) by using the surface intersection vector for SH evaluation instead of the viewing vector, allowing spatial variation of color/opacity within a single Gaussian [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians initialized from Mini-Splatting output [S1]. View-dependent color is not loaded as input [S1].
*   **Output:** Rendered images with spatially varying color and opacity per Gaussian.

**Data Structures:**
Each Gaussian $i$ retains the standard 3DGS parameters:
*   Position: $x \in \mathbb{R}^3$
*   Rotation: $q \in \mathbb{R}^4$ (quaternion)
*   Scale: $s \in \mathbb{R}^3$
*   Opacity: $\alpha \in [0, 1]$ (base opacity)
*   Color Texture: 16 SH coefficients ($\gamma_{lm}$) for 3 color channels [S1].
*   Opacity Texture: 16 SH coefficients ($\alpha_{lm}$) for 1 opacity channel [S1].
*   Total SH Channels: 4 (3 for color, 1 for opacity) [S1]. This increases storage size slightly compared to baseline 3DGS which uses 3 channels [S1].

**Ellipsoid Definition:**
The Gaussian is modeled as an ellipsoid in 3D space using the equation:
$$ \frac{x^2}{s_x^2} + \frac{y^2}{s_y^2} + \frac{z^2}{s_z^2} = 1 $$
where $s_x, s_y, s_z$ are scale parameters [S1].

**Rendering Steps:**
1.  **Ray Intersection:** For a viewing ray $r(t) = o + td$, where $o$ and $d$ are the ray’s origin and direction transformed into the Gaussian's local coordinate system, calculate the intersection points with the ellipsoid [S1].
2.  **Quadratic Solution:** Substitute the ray equation into the ellipsoid equation to obtain a quadratic equation $At^2 + Bt + C = 0$ [S1].
    *   $A = \frac{d_x^2}{s_x^2} + \frac{d_y^2}{s_y^2} + \frac{d_z^2}{s_z^2}$
    *   $B = 2\left(\frac{o_x d_x}{s_x^2} + \frac{o_y d_y}{s_y^2} + \frac{o_z d_z}{s_z^2}\right)$
    *   $C = \frac{o_x^2}{s_x^2} + \frac{o_y^2}{s_y^2} + \frac{o_z^2}{s_z^2} - 1$
    *   Solve for $t$ using the quadratic formula: $t = \frac{-B \pm \sqrt{B^2 - 4AC}}{2A}$ [S1].
    *   Select the intersection point with the lower $t$ value (closer to ray origin) [S1].
3.  **Normalization:** Transform the selected intersection point into the Gaussian's local coordinate system and normalize each dimension based on the scale $s$ to obtain the normalized vector $n$ [S1].
    *   Note: The scale parameters used in the ellipsoid equation (Equation 8 in source) are the square root of the stored scale parameters, $\sqrt{s}$ [S1]. This allows larger ellipsoids to yield better performance during optimization without altering the actual Gaussian size in splatting [S1].
4.  **SH Evaluation:** Evaluate SH basis functions $Y_{lm}$ using the normalized vector $n$ instead of the viewing vector $v$ [S1].
    *   Color: $c_i = \sum_{l} \sum_{m} \gamma_{lm}(i) Y_{lm}(n)$ [S1].
    *   Opacity: $\alpha_i = \sum_{l} \sum_{m} \alpha_{lm}(i) Y_{lm}(n)$ [S1].
    *   Apply Sigmoid activation function to both color ($\gamma_c$) and opacity ($\alpha_\alpha$) coefficients to ensure smooth gradients [S1].
5.  **Compositing:** Composite the resulting color and opacity with other Gaussians using standard alpha blending [S1].

**Optimization:**
*   **Incremental Updates:** Unlike standard 3DGS where view-dependent color loss is aggregated and propagated to SH coefficients in a single update at the end of each iteration, Textured-GS updates SH coefficients incrementally after each pixel is processed [S1]. This is necessary because the parameterization vector $n$ varies with each pixel, unlike the constant view vector in standard 3DGS [S1].
*   **Loss Function:** The loss function is identical to the original 3DGS framework:
    $$ L = (1 - \lambda)L_1 + \lambda L_{D-SSIM} $$
    where $\lambda = 0.2$ [S1].
*   **Learning Rates:** 0.0025 for color texture, 0.005 for opacity texture [S1].
*   **Adaptive Control:** Remove adaptive control modules (splitting/cloning) from the 3DGS framework to streamline optimization [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def evaluate_textured_gs(gaussians, rays, sh_degree=3):
    """
    Renders a scene using Textured-GS.
    
    Args:
        gaussians: List of Gaussian objects. Each has:
            - pos: (3,) position
            - rot: (4,) quaternion
            - scale: (3,) scale parameters (stored as s, used as sqrt(s) in ellipsoid)
            - base_alpha: (1,) base opacity
            - color_sh: (16, 3) SH coefficients for color
            - opacity_sh: (16, 1) SH coefficients for opacity
        rays: Tensor of shape (H, W, 3) containing ray origins and directions in world space.
        sh_degree: Degree of SH (default 3).
        
    Returns:
        rendered_color: (H, W, 3)
        rendered_alpha: (H, W, 1)
    """
    H, W, _ = rays.shape
    rendered_color = torch.zeros(H, W, 3)
    rendered_alpha = torch.zeros(H, W, 1)
    
    # Precompute SH basis functions for all possible normalized vectors n
    # In practice, this is done per-pixel or per-gaussian intersection
    
    for y in range(H):
        for x in range(W):
            ray_o, ray_d = rays[y, x, :3], rays[y, x, 3:] # Assuming rays packed as o+d or separate
            
            # Sort gaussians by depth or process all
            # For simplicity, assume we iterate through relevant gaussians for this pixel
            # In a real rasterizer, this is done via spatial hashing
            
            for g in gaussians:
                # 1. Transform ray to Gaussian local space
                # Note: Standard 3DGS transforms Gaussian to camera space. 
                # Textured-GS description implies transforming ray to ellipsoid local space.
                # We assume standard 3DGS projection logic for position/scale, 
                # but use local space for intersection.
                
                # Get Gaussian covariance in world space
                # Sigma = R * S * S^T * R^T
                # However, Textured-GS uses explicit ellipsoid intersection.
                
                # Transform ray origin and direction to Gaussian local coordinate system
                # This requires the inverse rotation of the Gaussian
                local_o = transform_to_local(g.pos, g.rot, ray_o)
                local_d = transform_to_local(g.pos, g.rot, ray_d)
                
                # 2. Solve quadratic for intersection
                # Ellipsoid: x^2/sx^2 + y^2/sy^2 + z^2/sz^2 = 1
                # Use sqrt(s) for scale in ellipsoid equation
                sx, sy, sz = torch.sqrt(g.scale)
                
                A = (local_d[0]**2 / sx**2) + (local_d[1]**2 / sy**2) + (local_d[2]**2 / sz**2)
                B = 2 * ((local_o[0]*local_d[0] / sx**2) + (local_o[1]*local_d[1] / sy**2) + (local_o[2]*local_d[2] / sz**2))
                C = (local_o[0]**2 / sx**2) + (local_o[1]**2 / sy**2) + (local_o[2]**2 / sz**2) - 1
                
                discriminant = B**2 - 4*A*C
                
                if discriminant < 0:
                    continue # No intersection
                
                sqrt_disc = torch.sqrt(discriminant)
                t1 = (-B - sqrt_disc) / (2*A)
                t2 = (-B + sqrt_disc) / (2*A)
                
                # Select closest intersection
                t = t1 if t1 > 0 else (t2 if t2 > 0 else None)
                if t is None:
                    continue
                    
                # 3. Calculate intersection point in local space
                local_p = local_o + t * local_d
                
                # 4. Normalize to get vector n
                n = local_p / torch.tensor([sx, sy, sz])
                # n is now on the unit sphere surface relative to the scaled axes
                
                # 5. Evaluate SH for Color and Opacity using n
                # SH Basis functions Y_lm(n)
                # For degree 3, there are 16 coefficients per channel
                
                color_val = evaluate_sh(g.color_sh, n) # Shape (3,)
                opacity_val = evaluate_sh(g.opacity_sh, n) # Shape (1,)
                
                # Apply Sigmoid
                color_val = torch.sigmoid(color_val)
                opacity_val = torch.sigmoid(opacity_val)
                
                # 6. Composite
                # Standard alpha blending
                alpha = opacity_val * g.base_alpha # Combine with base opacity? 
                # Source says "integrate an opacity channel into the SH framework". 
                # It implies the SH output IS the opacity variation. 
                # Let's assume the SH output is the final alpha for this Gaussian.
                alpha = opacity_val
                
                if alpha > 0:
                    # Update pixel
                    # Note: In a full rasterizer, we would sort and blend.
                    # Here we just show the contribution.
                    rendered_color[y, x] += color_val * alpha
                    rendered_alpha[y, x] += alpha
                    
    # Normalize by accumulated alpha
    rendered_alpha = torch.clamp(rendered_alpha, max=1.0)
    rendered_color = rendered_color / (rendered_alpha + 1e-6)
    
    return rendered_color, rendered_alpha

def evaluate_sh(sh_coeffs, n):
    """
    Evaluates Spherical Harmonics for a given direction n.
    sh_coeffs: (N, C) where N is number of coefficients (16 for deg 3), C is channels (3 or 1)
    n: (3,) normalized vector
    """
    # Precomputed SH basis functions for degree 3
    # Y_00, Y_1-1, Y_10, Y_11, Y_2-2, Y_2-1, Y_20, Y_21, Y_22, Y_3-3, Y_3-2, Y_3-1, Y_30, Y_31, Y_32, Y_33
    # Standard SH basis evaluation
    x, y, z = n[0], n[1], n[2]
    
    # Simplified basis evaluation for demonstration
    # In practice, use a library or precomputed table
    basis = compute_sh_basis(n, degree=3) # Returns (16,)
    
    # Dot product
    return torch.dot(basis, sh_coeffs)

def compute_sh_basis(n, degree=3):
    """
    Computes SH basis functions for degree up to 'degree'.
    Returns tensor of shape (16,) for degree 3.
    """
    x, y, z = n
    # Normalization constants for SH
    # C0 = sqrt(1/4pi)
    # C1 = sqrt(3/4pi)
    # etc.
    
    # Degree 0
    y00 = 0.282095 # sqrt(1/4pi)
    
    # Degree 1
    y1m1 = 0.488603 * y # sqrt(3/4pi) * (-y/sqrt(2))? Standard convention varies.
    y10 = 0.488603 * z
    y11 = 0.488603 * x
    
    # Degree 2
    y2m2 = 1.092548 * x * y
    y2m1 = 1.092548 * y * z
    y20 = 0.946175 * (3*z*z - 1) / 2 # Simplified
    y21 = 1.092548 * x * z
    y22 = 0.946175 * (x*x - y*y)
    
    # Degree 3
    y3m3 = 0.590044 * y * (3*x*x - y*y)
    y3m2 = 2.890611 * z * (x*x - y*y)
    y3m1 = 0.457046 * y * (5*z*z - 1)
    y30 = 0.373176 * (5*z*z*z - 3*z)
    y31 = 0.457046 * x * (5*z*z - 1)
    y32 = 1.445306 * z * (x*x - y*y)
    y33 = 0.590044 * x * (x*x - 3*y*y)
    
    # Note: The exact coefficients and signs depend on the SH convention used.
    # The source uses standard SH basis functions [22].
    # This is a placeholder for the actual basis evaluation.
    return torch.tensor([y00, y1m1, y10, y11, y2m2, y2m1, y20, y21, y22, y3m3, y3m2, y3m1, y30, y31, y32, y33])

def transform_to_local(pos, rot, point):
    """
    Transforms a point from world space to Gaussian local space.
    """
    # Inverse rotation
    inv_rot = inverse_quaternion(rot)
    # Translate to center
    translated = point - pos
    # Rotate
    local_point = quaternion_rotate(inv_rot, translated)
    return local_point
```

## Operating characteristics
*   **Time Complexity:** Rendering speed is comparable to standard 3DGS but slightly slower due to the intersection calculation and per-pixel SH evaluation. On the Bicycle Scene (Mip-NeRF 360), it achieves 161 FPS [S1].
*   **Memory Complexity:** Storage size is slightly larger than baseline 3DGS with the same Gaussian count because it requires 4 channels of SH coefficients (3 for color, 1 for opacity) instead of 3 [S1]. On the Bicycle Scene, it uses 4.53GB memory and 164MB file size [S1].
*   **Training Time:** On the Bicycle Scene, training takes 10m52s [S1].
*   **Strengths:**
    *   Enhances textural representation without increasing Gaussian count [S1].
    *   Outperforms standard 3DGS and Mini-Splatting in SSIM and PSNR on Mip-NeRF 360, Tanks&Temples, and Deep Blending datasets [S1].
    *   Achieves comparable rendering quality with significantly fewer Gaussians (e.g., 0.49M on Mip-NeRF 360 vs 6.03M for 3DGS) [S1].
    *   Better representation of sharp features and detailed structures (e.g., table legs) compared to Mini-Splatting [S1].
*   **Limitations:**
    *   Requires incremental SH updates, which may complicate optimization compared to aggregated updates [S1].
    *   Does not include adaptive control (splitting/cloning), which may limit its ability to adapt to complex scene details dynamically [S1].
    *   Rendering speed and training time still have room for optimization [S1].
*   **When to use:** When high-fidelity textural details are required with limited Gaussian counts, or when post-processing pre-optimized Gaussian scenes (e.g., from Mini-Splatting) to improve quality [S1].
*   **When not to use:** When adaptive control (splitting/cloning) is necessary for scene complexity, or when minimal storage size is critical (due to 4-channel SH) [S1].

## Background
Textured-GS was introduced in the paper "Textured-GS: Gaussian Splatting with Spatially Defined Color and Opacity" [S1]. It builds on the 3D Gaussian Splatting (3DGS) framework [11] and Mini-Splatting [5]. The method addresses the limitation of standard 3DGS where each Gaussian exhibits only a single color from a specific viewing angle, limiting its representational capability [S1]. By incorporating spatially defined color and opacity variations, it enhances rendering quality [S1]. The method was validated on the Mip-NeRF 360, Tanks&Temples, and Deep Blending datasets [S1].

## Sources
[S1] Textured-GS: Gaussian Splatting with Spatially Defined Color and Opacity, https://core.ac.uk/works/391755422, fetched 2026-10-30
