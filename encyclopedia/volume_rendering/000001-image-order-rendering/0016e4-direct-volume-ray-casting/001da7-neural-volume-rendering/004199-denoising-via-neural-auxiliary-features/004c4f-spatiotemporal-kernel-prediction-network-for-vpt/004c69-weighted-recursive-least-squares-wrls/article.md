# Weighted Recursive Least Squares (wRLS)

Also called: wRLS

## Overview
Weighted Recursive Least Squares (wRLS) is a spatiotemporal denoising method for Volumetric Path Tracing (VPT) that adapts the Recursive Least Squares (RLS) algorithm to handle heterogeneous noise in low-sample rendering [S1]. Unlike standard RLS which relies on noise-free G-buffers as predictors, wRLS uses a temporally stable feature derived from an exponentially weighted history buffer and assigns weights to samples based on their variance, thereby reducing spike noise while maintaining temporal stability [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Current noisy Monte Carlo estimate $\tilde{I}_j(t)$ for pixel $j$ at time $t$; previous frame's linear model coefficients $\beta_j(t-1)$ and inverse covariance matrix $P_j(t-1)$; reprojected pixel coordinate $q$ from the previous frame; optical flow/velocity data for reprojection; predictor vector $p_j(t)$ [S1].
*   **Output:** Denoised color $\hat{I}_j(t)$ for pixel $j$ at time $t$ [S1].

**Data Structures:**
*   **Linear Model Coefficients $\beta_j(t)$:** A vector of size $d$ (dimension of predictor vector $p_j$) [S1].
*   **Inverse Covariance Matrix $P_j(t)$:** A $d \times d$ matrix containing the inverse covariance of the predictor vectors [S1].
*   **Temporal Feature Buffer $z_j(t)$:** A vector of size 3 (RGB channels), stored per pixel and constructed using all three channels simultaneously to avoid color-shift effects [S1].
*   **Predictor Vector $p_j(t)$:** A vector of size $d$ [S1]. In standard RLS, this is derived from G-buffers, but wRLS avoids noisy G-buffers; the exact composition of $p_j(t)$ is implementation-dependent but must match dimension $d$ [S1].

**Parameters:**
*   $\lambda$: Forgetting factor, typically $0.998$ [S1].
*   $h$: Filtering bandwidth, typically $0.75$ [S1].
*   $\alpha$: History weight, typically $0.75$ [S1].
*   $\epsilon$: Small constant to avoid division by zero [S1].

**Steps:**
1.  **Temporal Feature Update:** Compute the temporally stable feature $z_j(t)$ for pixel $j$ at time $t$ [S1]. Determine the reprojected coordinate $q$ from the previous frame using optical flow [S1]. Update the feature using an exponentially weighted history buffer: $z_j(t) = \alpha \Psi[z_q(t-1), \tilde{I}_j(t)] + (1 - \alpha) \tilde{I}_j(t)$, where $\Psi[\cdot]$ is a neighborhood clamping operator that rectifies the reprojected history using the current-frame input estimate to reduce artifacts from stale data [S1].
2.  **Weight Calculation:** Calculate the weight $w_j(t)$ for the current noisy estimate $\tilde{I}_j(t)$ based on the distance $d_j(t)$ between the estimate and the temporal feature: $d_j(t) = \frac{\|\tilde{I}_j(t) - z_j(t)\|}{\min(\|\tilde{I}_j(t)\|, \|z_j(t)\|) + \epsilon}$ and $w_j(t) = \exp\left(-\frac{d_j(t)^2}{h}\right)$ [S1]. This assigns high weights to low-variance samples and low weights to outliers [S1].
3.  **Gain Vector Computation:** Compute the weighted gain vector $q_{wj}(t)$ using the weighted inverse covariance update: $q_{wj}(t) = \frac{P_j(t-1)p_j^T(t)}{\lambda + w_j(t)p_j(t)P_j(t-1)p_j^T(t)}$ [S1].
4.  **Linear Model Update:** Update the linear model coefficients $\beta_j(t)$ using the estimated error $\hat{e}_j(t) = \tilde{I}_j(t) - \beta_j^T(t-1)p_j(t)$ and the weighted gain: $\beta_j(t) = \beta_j(t-1) + q_{wj}(t)\hat{e}_j(t)$ [S1].
5.  **Inverse Covariance Update:** Update the inverse covariance matrix $P_j(t)$ using the matrix inversion lemma: $P_j(t) = \lambda^{-1}[P_j(t-1) - q_{wj}(t)p_j(t)P_j(t-1)]$ [S1].
6.  **Spatial Filtering:** Apply a spatial filter to the denoised output to reduce residual noise [S1]. For each pixel $i$, blend colors predicted from neighboring pixels within a $5 \times 5$ window centered at $i$ [S1]. The prediction for a neighbor $k$ is $\hat{I}_k(t) = \beta_k^T(t)p_k(t)$ [S1]. The final color is a bilateral-weighted average of these predictions [S1].

**Pitfalls:** VPT does not generate noise-free G-buffers at low sample counts, making standard RLS predictors problematic; solved by using the temporally stable feature $z_j(t)$ instead [S1]. Ambiguous world coordinates in participating media due to multiple ray intersections can cause unstable temporal reprojections if selected randomly or averaged; solved by using the closest world coordinate from the view position of the first real collision according to delta tracking [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def wrls_denoise(I_curr, beta_prev, P_prev, z_prev, flow, p_curr, 
                 alpha=0.75, h=0.75, lam=0.998, eps=1e-6):
    """
    I_curr: ndarray (H, W, 3), current noisy MC estimates.
    beta_prev: ndarray (H, W, d), previous linear model coefficients.
    P_prev: ndarray (H, W, d, d), previous inverse covariance matrices.
    z_prev: ndarray (H, W, 3), previous temporal features.
    flow: ndarray (H, W, 2), optical flow vectors (dy, dx).
    p_curr: ndarray (H, W, d), predictor vectors for current frame.
    alpha: History weight.
    h: Filtering bandwidth.
    lam: Forgetting factor.
    eps: Small constant for numerical stability.
    Returns: denoised (H, W, 3), new_beta (H, W, d), new_P (H, W, d, d), new_z (H, W, 3)
    """
    H, W, C = I_curr.shape
    d = beta_prev.shape[2]
    
    new_beta = np.zeros_like(beta_prev)
    new_P = np.zeros_like(P_prev)
    new_z = np.zeros_like(z_prev)
    
    # 1. Temporal Feature Update
    # Reproject coordinates based on flow
    y, x = np.meshgrid(np.arange(H), np.arange(W), indexing='ij')
    q_y = np.clip(y + flow[:,:,0].astype(int), 0, H-1)
    q_x = np.clip(x + flow[:,:,1].astype(int), 0, W-1)
    
    # Get reprojected history
    z_prev_reprojected = z_prev[q_y, q_x]
    
    # Neighborhood clamping operator Psi [G]
    # Clamps reprojected history to current pixel value if difference exceeds threshold
    threshold = 0.1
    diff = np.abs(z_prev_reprojected - I_curr)
    mask = diff > threshold
    z_clamped = np.where(mask, I_curr, z_prev_reprojected)
    
    # Update z
    new_z = alpha * z_clamped + (1 - alpha) * I_curr
    
    # 2. Weight Calculation
    # Distance d_j(t)
    num = np.linalg.norm(I_curr - new_z, axis=2, keepdims=True)
    norm_curr = np.linalg.norm(I_curr, axis=2, keepdims=True)
    norm_hist = np.linalg.norm(new_z, axis=2, keepdims=True)
    den = np.minimum(norm_curr, norm_hist) + eps
    d_j = num / den
    
    # Weight w_j(t)
    w_j = np.exp(-(d_j**2) / h)
    
    # 3. Gain Vector Computation
    # q_wj(t) = P_j(t-1) p_j(t)^T / (lambda + w_j(t) p_j(t) P_j(t-1) p_j(t)^T)
    # p_j P_j(t-1) p_j^T -> scalar per pixel
    pP_p = np.einsum('ijk,ijkl,ijl->ij', p_curr, P_prev, p_curr)
    denom = lam + w_j * pP_p
    denom = np.expand_dims(denom, axis=2) # (H, W, 1)
    
    # Numerator: P_j(t-1) p_j(t)^T -> (H, W, d)
    num_gain = np.einsum('ijkl,ijl->ijk', P_prev, p_curr)
    
    q_wj = num_gain / denom # (H, W, d)
    
    # 4. Linear Model Update
    # Error e_j(t) = I_j(t) - beta_j(t-1)^T p_j(t)
    pred = np.einsum('ijk,ijk->ij', beta_prev, p_curr) # (H, W)
    pred = np.expand_dims(pred, axis=2) # (H, W, 1)
    e_j = I_curr - pred # (H, W, 3) broadcasted to (H, W, 1)
    
    new_beta = beta_prev + q_wj * e_j # (H, W, d)
    
    # 5. Inverse Covariance Update
    # P_j(t) = lambda^-1 [P_j(t-1) - q_wj(t) p_j(t) P_j(t-1)]
    q_p_P = np.einsum('ijk,ijk,ijkl->ijl', q_wj, p_curr, P_prev) # (H, W, d)
    new_P = (1.0 / lam) * (P_prev - np.einsum('ijk,ijl->ijkl', q_wj, q_p_P)) # (H, W, d, d)
    
    # 6. Spatial Filtering [G]
    # Predict colors from updated models for all pixels
    I_pred_all = np.einsum('ijk,ijk->ij', new_beta, p_curr)  # (H, W)
    I_pred_all = np.expand_dims(I_pred_all, axis=2)           # (H, W, 1)
    I_pred_all = np.broadcast_to(I_pred_all, (H, W, 3))       # (H, W, 3)
    
    pad = 2
    padded = np.pad(I_pred_all, ((pad, pad), (pad, pad), (0, 0)), mode='edge')
    final_denoised = np.zeros_like(I_curr)
    weight_sum = np.zeros((H, W, 1))
    
    for dy in range(-pad, pad + 1):
        for dx in range(-pad, pad + 1):
            neighbor = padded[pad + dy:pad + dy + H, pad + dx:pad + dx + W, :]
            spatial_w = np.exp(-(dy**2 + dx**2) / (2 * pad**2))
            color_w = np.exp(-np.sum((neighbor - I_pred_all)**2, axis=2, keepdims=True) / (2 * 0.05**2))
            w = spatial_w * color_w
            final_denoised += neighbor * np.expand_dims(w, axis=2)
            weight_sum += np.expand_dims(w, axis=2)
    
    final_denoised /= np.clip(weight_sum, 1e-8, None)
    
    return final_denoised, new_beta, new_P, new_z
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method is designed for real-time VPT [S1]. It reduces spike noise and maintains temporal stability better than standard RLS in low-sample scenarios [S1].
*   **Strengths:** Handles heterogeneous noise in VPT; temporally stable; reduces spike noise; works with low sample counts [S1].
*   **Limitations:** Requires optical flow for reprojection; sensitive to ambiguous world coordinates in participating media (mitigated by using the first real collision); spatial filtering may introduce blurring if not tuned correctly [S1].
*   **When to use:** Real-time VPT with low sample counts where temporal stability is critical and spike noise is prevalent [S1].
*   **When not to use:** When high-quality G-buffers are available (standard RLS may be simpler); when computational resources are extremely limited (spatial filtering adds overhead) [S1].

## Background
Weighted Recursive Least Squares (wRLS) was proposed in the paper "Real-Time Denoising of Volumetric Path Tracing for Direct Volume Rendering" [S1]. It addresses the limitations of standard RLS approaches when applied to stochastic Monte Carlo Direct Volume Rendering (MC-DVR), specifically the lack of noise-free G-buffers and the high variability of linear model coefficients due to noise [S1]. The method was developed to enable interactive denoising for VPT sequences [S1].

## Sources
[S1] Real-Time Denoising of Volumetric Path Tracing for Direct Volume Rendering, https://doi.org/10.1109/tvcg.2020.3037680, fetched 2026-10-26
