# Explicit Tone Mapping

Also called: Camera response modeling, Global Tone-Mapping Function, HDR to LDR conversion, M(E, EV), Reinhard Tone-Mapping with Exposure Scaling

## Overview
Explicit Tone Mapping is a global function that converts High Dynamic Range (HDR) pixel values into Low Dynamic Range (LDR) images for evaluation, visualization, or as input for downstream tasks [S1, S2]. It simulates the image acquisition process of a digital camera by combining exposure scaling, white balance, and a parameterized Camera Response Function (CRF) [S2]. Unlike adaptive or local tone-mapping operators, it applies a uniform transformation across the entire image based on physical radiometric parameters [S1, S2].

## How it works
**Inputs and Outputs**
*   **Input:** HDR image $I_h$ (or pixel color $C_h(\mathbf{r}) \in \mathbb{R}^3$ for a specific ray $\mathbf{r}$) [S2].
*   **Parameters:**
    *   Exposure Value $EV \in \mathbb{R}$ [S1].
    *   White balance parameters $\theta_w = [w_r, w_g, w_b]^T \in \mathbb{R}^3$ [S2].
    *   CRF control points $\theta_g$ (256 values) [S2].
    *   Thresholding coefficient $\alpha$ [S2].
*   **Output:** LDR image $I_l$ (pixel values in $[0, 1]$) [S1, S2].

**Data Structures**
*   **CRF Lookup:** A 1D grid of 256 control points representing the CRF $g(x)$ over the domain $[0, 1]$ [S2]. The function is approximated as a piece-wise linear function [S2].
*   **White Balance Vector:** A 3-element vector $\theta_w$ [S2].

**Steps**
1.  **Exposure Scaling:** Scale the HDR pixel value $E$ by $2^{EV}$ to simulate exposure time $\Delta t = 2^{EV}$ [S1].
2.  **White Balance:** Apply element-wise multiplication of the scaled HDR color channels by the white balance parameters [S2].
3.  **CRF Application:** Map the white-balanced pixel value to the LDR range using the parameterized CRF $g$ [S2].
4.  **Leaky Thresholding:** Apply leaky thresholding to the CRF output to handle saturation regions during training [S2].

**Formulas**
The overall tone mapping function is $I_l = T(I_h, \theta) = g(w(I_h))$ [S2].

1.  **Exposure Scaling (S1):**
    $$ E_{scaled} = 2^{EV} E $$
    Where $E$ is the HDR pixel value and $EV$ is the exposure value [S1].

2.  **White Balance (S2):**
    $$ I_w = C_h(\mathbf{r}) \odot \theta_w = \begin{bmatrix} w_r c_h^r \\ w_g c_h^g \\ w_b c_h^b \end{bmatrix} $$
    Where $\odot$ is element-wise product, $C_h(\mathbf{r}) = [c_h^r, c_h^g, c_h^b]^T$, and $\theta_w$ is regularized to be positive [S2].

3.  **CRF Mapping (S2):**
    The CRF $g(x)$ is a piece-wise linear function defined by 256 control points. For a pixel value $x$ in $I_w$, $g(x)$ is computed by interpolating the control points in the domain $[0, 1]$ using 1D grid-sampling [S2]. Boundary conditions are $g(0) = 0$ and $g(1) = 1$ [S2].

4.  **Leaky Thresholding (S2):**
    To propagate loss in saturation regions, the CRF is modified to $g_{leaky}(x)$:
    $$ g_{leaky}(x) = \begin{cases} \alpha x, & x < 0 \\ g(x), & 0 \leq x \leq 1 \\ -\frac{\alpha}{\sqrt{x}} + \alpha + 1, & x > 1 \end{cases} $$
    Where $\alpha$ is the thresholding coefficient [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def explicit_tone_mapping(hdr_image, ev, white_balance, crf_control_points, alpha):
    """
    Converts HDR image to LDR using explicit tone mapping.
    
    Args:
        hdr_image: Tensor of shape (..., 3) with HDR pixel values.
        ev: Scalar exposure value.
        white_balance: Tensor of shape (3,) with [wr, wg, wb].
        crf_control_points: Tensor of shape (256,) representing CRF values at uniform intervals in [0, 1].
        alpha: Scalar thresholding coefficient.
        
    Returns:
        ldr_image: Tensor of shape (..., 3) with LDR pixel values.
    """
    # 1. Exposure Scaling
    # Scale HDR pixel value by 2^EV
    exposure_scale = torch.pow(2.0, ev)
    scaled_hdr = hdr_image * exposure_scale
    
    # 2. White Balance
    # Element-wise product with white balance parameters
    # Regularization of white_balance to positive values is assumed to be handled 
    # by the optimizer constraints or a preceding clamp, as per [S2].
    white_balanced = scaled_hdr * white_balance
    
    # 3. CRF Application
    # Map white-balanced values to [0, 1] range for CRF lookup
    # Clamp to [0, 1] for standard CRF interpolation, then apply leaky thresholding
    # Note: The source [S2] applies thresholding to the input of the CRF logic 
    # or the output logic depending on interpretation. The formula g_leaky(x) 
    # suggests x is the input to the CRF function.
    
    # We need to interpolate the CRF. 
    # crf_control_points are defined on uniform grid in [0, 1].
    # We assume white_balanced values are normalized or the CRF is defined 
    # on the range of white_balanced. However, [S2] states "pixel value of Iw 
    # is mapped to Il by interpolating... in the domain [0,1]". 
    # This implies Iw values are expected to be in [0,1] or normalized.
    # If Iw is not in [0,1], we must normalize or the CRF definition implies 
    # the domain is the normalized intensity. 
    # Given [S2] "A range beyond the dynamic range [0,1] is thresholded when 
    # applying the CRF g", it suggests the CRF is defined on [0,1].
    
    # Let's assume Iw is normalized to [0,1] or the CRF handles the range.
    # Standard practice in [S2] context: Iw values are treated as intensities.
    # If Iw > 1, it is saturated.
    
    # To implement g_leaky, we first identify regions.
    # However, g(x) is the interpolation.
    
    # Let's normalize Iw to [0,1] if it represents relative intensity, 
    # or assume Iw is already in a range where [0,1] is the full scale.
    # The source says "g(0)=0, g(1)=1".
    
    # We will implement the interpolation for values in [0,1] and handle 
    # out-of-bounds with leaky thresholding.
    
    # Normalize Iw to [0, 1] for the CRF domain if necessary. 
    # If the source implies Iw is already in [0,1], we skip normalization.
    # "pixel value of white balance corrected image Iw is mapped to Il"
    # "divided into 256 intervals ... in [0,1]"
    # This implies the input to g is in [0,1].
    # If Iw has values outside [0,1], they are handled by the leaky threshold.
    
    # Let's assume Iw is the input x.
    x = white_balanced
    
    # Clip to [0, 1] for standard interpolation to avoid index errors
    # But we need the leaky behavior for x < 0 and x > 1.
    
    # Interpolate CRF for values in [0, 1]
    # Create grid for interpolation
    grid_x = torch.linspace(0.0, 1.0, 256, device=hdr_image.device)
    
    # Use grid_sample or manual interpolation
    # Manual interpolation for clarity and no library dependencies
    # Reshape for broadcasting if needed
    
    # Flatten for processing
    x_flat = x.view(-1, 3)
    
    # Handle x < 0
    mask_lt_0 = x_flat < 0
    # Handle x > 1
    mask_gt_1 = x_flat > 1
    mask_in_range = ~mask_lt_0 & ~mask_gt_1
    
    # Interpolate for in-range values
    # Normalize x_flat to [0, 1] if it's not already? 
    # The source says "domain [0,1]". If Iw is radiance, it might be > 1.
    # But the CRF is defined on [0,1]. 
    # "A range beyond the dynamic range [0,1] is thresholded"
    # This implies the CRF is only defined for [0,1].
    # So we must normalize Iw to [0,1] to use the CRF? 
    # Or does the CRF apply to the normalized intensity?
    # "pixel value of Iw ... mapped to Il ... interpolating ... in the domain [0,1]"
    # This strongly suggests Iw values are normalized to [0,1] before CRF, 
    # OR the CRF is applied to the raw value but the control points are 
    # spaced over the expected range.
    # Given "g(0)=0, g(1)=1", and "thresholded", it's most likely that 
    # the CRF is applied to the normalized intensity.
    
    # Let's assume Iw is normalized to [0,1] by dividing by max(Iw) or similar?
    # No, [S2] does not mention normalization. It says "pixel value of Iw".
    # If Iw is radiance, it can be > 1.
    # If the CRF is defined on [0,1], then inputs > 1 are out of domain.
    # The leaky threshold handles x > 1.
    # So, for x in [0,1], we interpolate.
    
    # We need to ensure x_flat is in [0,1] for interpolation.
    # If x_flat is not in [0,1], we clamp it for interpolation, 
    # but the leaky formula handles the deviation.
    
    x_clamped = torch.clamp(x_flat, 0.0, 1.0)
    
    # Interpolate CRF
    # grid_sample expects input in [0,1] if normalized=True
    # We reshape x_clamped to (N, 1, 1, 1) for grid_sample
    x_grid = x_clamped.unsqueeze(1).unsqueeze(1) # (N, 1, 1, 1)
    
    # CRF control points as a 1D grid
    # grid_sample requires input to be normalized coordinates
    # We pass x_grid as the coordinate
    crf_values = crf_control_points.unsqueeze(0).unsqueeze(0).unsqueeze(0) # (1, 1, 256, 1) ? 
    # Actually, grid_sample samples from a grid.
    # Let's use F.grid_sample with the CRF as the grid?
    # No, simpler: linear interpolation manually.
    
    # Indices for interpolation
    idx = x_clamped * 255.0
    idx_low = idx.floor().long()
    idx_high = idx.ceil().long()
    idx_high = torch.clamp(idx_high, 0, 255)
    
    frac = idx - idx_low
    
    crf_low = crf_control_points[idx_low]
    crf_high = crf_control_points[idx_high]
    
    crf_interp = crf_low + frac * (crf_high - crf_low)
    
    # Apply leaky thresholding
    ldr_flat = torch.zeros_like(x_flat)
    
    # Region 1: x < 0
    ldr_flat[mask_lt_0] = alpha * x_flat[mask_lt_0]
    
    # Region 2: 0 <= x <= 1
    ldr_flat[mask_in_range] = crf_interp[mask_in_range]
    
    # Region 3: x > 1
    # g_leaky(x) = -alpha/sqrt(x) + alpha + 1
    ldr_flat[mask_gt_1] = -alpha / torch.sqrt(x_flat[mask_gt_1]) + alpha + 1
    
    # Reshape back
    ldr_image = ldr_flat.view(x.shape)
    
    return ldr_image
```

**Parameters**
*   $EV$: Exposure value. Typical values depend on the desired exposure range for synthetic or real-world scenes [S1].
*   $\theta_w$: White balance parameters. Regularized to be positive [S2].
*   $\alpha$: Thresholding coefficient. Value not given in the sources [S2].
*   CRF Control Points: 256 values. Boundary conditions $g(0)=0, g(1)=1$ [S2].

**Edge Cases and Pitfalls**
*   **Saturation:** Values outside $[0, 1]$ are handled by leaky thresholding to allow gradient propagation during training [S2].
*   **White Balance Regularization:** $\theta_w$ must be positive to maintain physical propriety [S2].
*   **CRF Domain:** The CRF is defined on $[0, 1]$. Inputs outside this range require special handling (leaky thresholding) [S2].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels, as it involves element-wise operations and linear interpolation [S1, S2].
*   **Memory Complexity:** $O(N)$ for the output image and $O(1)$ for parameters (256 control points) [S1, S2].
*   **Strengths:** Differentiable, allowing end-to-end training of radiance fields with LDR supervision [S2]. Simulates camera response physically [S2].
*   **Limitations:** Global operation may not preserve local contrast as well as adaptive methods [S1]. Requires parameterization of CRF [S2].
*   **When to Use:** When generating LDR ground truth from HDR renders for evaluation or when training HDR radiance fields with LDR inputs [S1, S2].

## Background
Explicit Tone Mapping was introduced in the context of HDR-NeRF and HDR-Plenoxels to evaluate and train Neural Radiance Fields using LDR images [S1, S2]. It is based on the classical Reinhard tone-mapping operator [S1]. The method simulates the image acquisition pipeline of a digital camera, including white balance and CRF [S2].

## Sources
[S1] HDR-NeRF: High Dynamic Range Neural Radiance Fields, https://xhuangcv.github.io/hdr-nerf/, fetched 2026-10-27
[S2] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
