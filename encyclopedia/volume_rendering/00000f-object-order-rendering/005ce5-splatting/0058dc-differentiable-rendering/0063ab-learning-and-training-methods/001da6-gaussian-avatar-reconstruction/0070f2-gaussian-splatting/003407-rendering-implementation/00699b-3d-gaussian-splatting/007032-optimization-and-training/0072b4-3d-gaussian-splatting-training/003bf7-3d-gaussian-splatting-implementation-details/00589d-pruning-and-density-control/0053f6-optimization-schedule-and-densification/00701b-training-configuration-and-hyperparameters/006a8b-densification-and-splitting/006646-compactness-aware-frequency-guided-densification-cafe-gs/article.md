# Compactness-Aware Frequency-Guided Densification (CAFe-GS)

Also called: CAFe-GS, CAFe-GS Densification

## Overview
CAFe-GS is a densification strategy for 3D Gaussian Splatting that replaces the traditional positional-gradient criterion with a per-Gaussian error score modulated by local frequency guidance [S1]. It solves the problem of redundant primitive generation in smooth regions and insufficient detail in underfit areas, providing a single interpretable parameter $\tau$ to control the quality-compactness trade-off [S1]. Unlike the gradient-based baseline which suffers from steep quality drops and narrow usable ranges, CAFe-GS offers a smoother trade-off, achieving comparable PSNR with 2–4× fewer Gaussians or up to 12–15× fewer Gaussians with a controlled 2–2.5 dB PSNR drop [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussian primitives $G$ (initialized via Structure-from-Motion), camera parameters, and a sequence of training images $\{I_v\}$ with corresponding rendered images $\{\hat{I}_v\}$.
*   **Outputs:** An updated set of Gaussians with densified primitives (cloned or split) based on the score $S_g$ and threshold $\tau$.

**Data Structures:**
Each Gaussian primitive $g$ is defined by:
*   Center $\mu_g \in \mathbb{R}^3$.
*   Covariance $\Sigma_g \in \mathbb{R}^{3\times3}$, parameterized as $\Sigma_g = R_g \text{diag}(s_g^2) R_g^\top$, where $R_g$ is a rotation matrix (from quaternion $q_g \in \mathbb{R}^4$) and $s_g \in \mathbb{R}^{3+}$ is the per-axis scale vector [S1].
*   Opacity $\alpha_g \in [0, 1]$.
*   Spherical Harmonics coefficients $a_{gk} \in \mathbb{R}^3$ for view-dependent color [S1].

**Parameters:**
*   $\tau$: Densification threshold. Typical values: $\{10, 25, 50, 75, 100\}$. Higher values favor compactness [S1].
*   $\sigma_{LoG}$: Standard deviation for the Laplacian-of-Gaussian filter. Value: $1$ [S1].
*   $t_h$: Clone/split scale threshold. Value: $0.01$ [S1].
*   Densification Schedule: Begins after 500-iteration warm-up, applied every 100 iterations, ending after 15,000 iterations. Total training continues to 30,000 iterations [S1].

**Algorithm Steps:**

1.  **Initialization and Optimization:**
    Initialize camera parameters and sparse Gaussians using Structure-from-Motion. Optimize all parameters $\{\mu_g, R_g, s_g, \alpha_g, a_{gk}\}$ to minimize photometric distortion loss (L1 + DSSIM) between rendered and ground-truth images [S1].

2.  **Compute Per-Pixel Distortion:**
    For each view $v$, compute the per-pixel distortion $d_v(u)$ as 1 minus the Structural Similarity Index (SSIM) between the input image $I_v$ and the rendered image $\hat{I}_v$ [S1]:
    $$ d_v(u) = 1 - \text{SSIM}(I_v, \hat{I}_v)(u) $$

3.  **Compute Frequency Magnitude Maps:**
    For each view $v$, compute the local frequency-magnitude map for the input image $I_v$ by applying a Laplacian-of-Gaussian (LoG) filter with $\sigma_{LoG}=1$ to the luminance channel and taking the absolute response [S1]:
    $$ F(I_v)(u) = |(LoG_{\sigma_{LoG}} * I_v)|(u) $$
    Similarly, compute $F(\hat{I}_v)(u)$ for the rendered image. Normalize both $F(I_v)$ and $F(\hat{I}_v)$ independently per view to the range $[0, 1]$ [S1].

4.  **Compute Frequency Gap Map:**
    Calculate the frequency gap $\Delta F_v(u)$ to identify under-detailing (input has more frequency than render) and over-detailing [S1]:
    $$ \Delta F_v(u) = F(I_v)(u) - F(\hat{I}_v)(u) + 1 $$
    Values $> 1$ indicate under-detailing; values $< 1$ indicate over-detailing [S1].

5.  **Compute Per-Gaussian Error Score ($E_g$):**
    Redistribute pixel-wise distortion to contributing primitives using effective opacity weights (transmittance) $C(g, u)$ [S1]. The effective contribution $C(g, u)$ is the weight of Gaussian $g$ in the front-to-back compositing at pixel $u$ [S1].
    Compute the per-view error attributed to Gaussian $g$:
    $$ e_{g,v} = \sum_{u \in \Omega_v} C(g, u) \cdot d_v(u) $$
    Compute the view-agnostic error score $E_g$ by taking the maximum error across all views to prevent averaging out localized underfitting [S1]:
    $$ E_g = \max_v e_{g,v} $$

6.  **Compute Frequency Modulation ($M_g^\Delta$):**
    Redistribute the frequency gap to Gaussians using the same effective opacity weights [S1]:
    $$ M_g^\Delta = \max_v \sum_{u \in \Omega_v} C(g, u) \cdot \Delta F_v(u) $$
    Taking the maximum over views avoids averaging away thin structures visible only from limited viewpoints [S1].

7.  **Compute Final Densification Score ($S_g$):**
    $$ S_g = E_g \cdot M_g^\Delta $$
    Select candidate Gaussians for densification if $S_g \ge \tau$ [S1].

8.  **Clone or Split Selected Gaussians:**
    For each selected Gaussian $g$ with per-axis scales $s_g = (s_{g,x}, s_{g,y}, s_{g,z})$:
    *   **Clone:** If $s_{g,i} \le t_h$ for all $i \in \{x, y, z\}$, duplicate the primitive by copying all attributes ($\mu_g, R_g, s_g, \alpha_g, SH$). Shift the child's position along the dominant scale axis [S1].
    *   **Split:** Otherwise, duplicate attributes, halve the scale along all axes ($s_{g,i} \leftarrow \frac{1}{2} s_{g,i}$), and slightly displace the child to prevent overlap. Orientation and color parameters remain unchanged [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_log_filter(image, sigma=1.0):
    """
    Computes the absolute Laplacian-of-Gaussian response for the luminance channel.
    image: H x W x 3 RGB array.
    Returns: H x W float array.
    """
    # Convert to luminance (standard BT.601)
    luminance = 0.299 * image[:, :, 0] + 0.587 * image[:, :, 1] + 0.114 * image[:, :, 2]
    
    # Create LoG kernel (simplified for description, assume standard scipy/numpy convolution)
    # In implementation, use a separable Gaussian derivative or precomputed kernel
    log_kernel = create_log_kernel(sigma) 
    response = np.abs(np.convolve(luminance, log_kernel, mode='same'))
    return response

def normalize_map(map_array):
    """Min-max normalize to [0, 1]"""
    min_val = np.min(map_array)
    max_val = np.max(map_array)
    if max_val == min_val:
        return np.zeros_like(map_array)
    return (map_array - min_val) / (max_val - min_val)

def cafe_gs_densification_step(gaussians, views, tau, t_h=0.01, sigma_log=1.0):
    """
    One step of CAFe-GS densification.
    gaussians: List of Gaussian objects with attributes mu, R, s, alpha, SH.
    views: List of view dicts containing 'I_v' (input image), 'I_hat_v' (rendered image),
           and 'C_g_u' (effective contribution weights for each Gaussian at each pixel).
    tau: Densification threshold.
    """
    candidates = []
    
    for g in gaussians:
        max_error = 0.0
        max_freq_mod = 0.0
        
        for view in views:
            I_v = view['I_v']
            I_hat_v = view['I_hat_v']
            C_g_u = view['C_g_u'][g.id] # Shape: (H, W)
            
            # 1. Per-pixel distortion
            # Assuming SSIM is computed per-pixel or locally
            ssim_map = compute_ssim_map(I_v, I_hat_v)
            d_v = 1.0 - ssim_map
            
            # 2. Per-view error
            e_g_v = np.sum(C_g_u * d_v)
            if e_g_v > max_error:
                max_error = e_g_v
                
            # 3. Frequency Guidance
            # Compute LoG for input and render
            F_I_v = normalize_map(compute_log_filter(I_v, sigma_log))
            F_I_hat_v = normalize_map(compute_log_filter(I_hat_v, sigma_log))
            
            # Frequency gap
            delta_F_v = F_I_v - F_I_hat_v + 1.0
            
            # Modulation
            M_g_delta_v = np.sum(C_g_u * delta_F_v)
            if M_g_delta_v > max_freq_mod:
                max_freq_mod = M_g_delta_v
                
        # Final Score
        S_g = max_error * max_freq_mod
        
        if S_g >= tau:
            candidates.append(g)
            
    # Perform Clone/Split
    new_gaussians = []
    for g in candidates:
        s = g.s # Scale vector (x, y, z)
        if np.all(s <= t_h):
            # Clone
            child = clone_gaussian(g)
            # Shift child along dominant scale axis
            dominant_axis = np.argmax(s)
            child.mu = shift_position(child.mu, dominant_axis, step=0.01) # [G] Step size heuristic
            new_gaussians.append(child)
        else:
            # Split
            child = clone_gaussian(g)
            child.s = g.s / 2.0
            # Slight displacement to prevent overlap
            child.mu = shift_position(child.mu, axis=np.random.randint(3), step=0.001) # [G] Heuristic
            new_gaussians.append(child)
            
    return new_gaussians

def clone_gaussian(g):
    """Deep copy of Gaussian attributes"""
    import copy
    return copy.deepcopy(g)

def shift_position(mu, axis, step):
    """Shift position along axis"""
    mu_new = mu.copy()
    mu_new[axis] += step
    return mu_new
```

## Operating characteristics
*   **Time and Memory Complexity:** The method adds overhead for computing SSIM and LoG filters per view during densification steps. However, it reduces the total number of Gaussians required for a given quality, lowering rendering memory and compute costs.
*   **Measured Performance:**
    *   On Mip-NeRF 360 [1], CAFe-GS with $\tau=10$ achieves 0.895 SSIM, 29.92 dB PSNR, 0.119 LPIPS, and 2.22M Gaussians. With $\tau=100$, it achieves 0.807 SSIM, 27.50 dB PSNR, 0.238 LPIPS, and 0.26M Gaussians [S1].
    *   On Tanks and Temples [12], CAFe-GS (Best quality) achieves PSNR 23.89, LPIPS 0.232, 4.5M Gaussians. CAFe-GS (Compactness-first) achieves PSNR 23.93, LPIPS 0.230, 2.9M Gaussians. The baseline 3DGS achieves PSNR 21.60, LPIPS 0.405, 0.3M Gaussians [S1].
*   **Scaling:** CAFe-GS provides a smoother and more predictable quality-compactness trade-off than the gradient-based baseline, which suffers from steep quality drops and narrow usable ranges [S1].
*   **Strengths:** Reduces Gaussian count by 2–4× at matched quality and up to 12–15× with controlled quality loss. Preserves high-frequency details (edges, thin contours) better than baseline [S1].
*   **Limitations:** On Tanks and Temples, parity with the gradient-based baseline is reached only at the lowest $\tau$ threshold, likely due to lower input image resolution limiting the effectiveness of frequency analysis [S1].
*   **When to use:** When model compactness is a priority or when the gradient-based densification produces excessive redundancy in smooth regions.
*   **When not to use:** In scenes with very low-resolution input images where frequency analysis is ineffective [S1].

## Background
CAFe-GS was introduced by Huar et al. from Inria and InterDigital [S1]. It addresses the limitations of the original 3D Gaussian Splatting (3DGS) Adaptive Density Control (ADC), which relies on positional gradients and often densifies already well-reconstructed areas [S1]. The method builds on the error-redistribution strategy of Bulò et al. [2] and adapts it with frequency guidance [S1]. It is evaluated on Mip-NeRF 360 [1] and Tanks and Temples [12] datasets [S1].

## Sources
[S1] CAFe-GS: Compactness-Aware Frequency-Guided Densification for 3D Gaussian Splatting, https://hal.science/hal-05583323v1, fetched 2026-10-26
