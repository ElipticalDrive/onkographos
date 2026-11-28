# FactorSplat
## Overview
FactorSplat is a per-scene N-dimensional Gaussian splatting (N-DGS) proxy that accepts region-specific intensity-to-RGBA transfer function (TF) curves at inference time, allowing appearance changes without retraining or refitting [S1, unofficial]. It solves the problem of image-trained Gaussian proxies baking one transfer function into their appearance, which prevents supporting new presets without rebuilding the proxy [S1, unofficial]. Unlike standard Gaussian Splatting where geometry and appearance are tightly coupled to a single rendering configuration, FactorSplat decouples the geometric primitives from the appearance logic, allowing multiple TFs to share the same underlying Gaussian proxy [S1, unofficial]. It differs from region-aware VEG by using region-specific intensity TFs and cinematic-rendering supervision, combining direct lookup with an image-trained N-DGS proxy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ Gaussian primitives defined by centers $\boldsymbol{\mu}_i$, covariances $\boldsymbol{\Sigma}_i$, and opacity logits $o_i$ [S1, unofficial]. A set of $M$ segmented regions/materials [S1, unofficial]. A target transfer function preset $T$, defined by region-specific intensity-to-RGBA curves [S1, unofficial].
*   **Output:** Rendered image with colors and opacities adjusted for preset $T$, while maintaining the shared geometry and directional appearance of the base proxy [S1, unofficial].

**Data Structures:**
*   **Gaussian Primitives:** For each primitive $i$:
    *   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Center [S1, unofficial].
    *   $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance [S1, unofficial].
    *   $o_i \in \mathbb{R}$: Opacity logit [S1, unofficial].
    *   $\mathbf{C}^{\text{base}}_i \in \mathbb{R}^{Q_b \times 3}$: Spherical Harmonic (SH) coefficients for RGB, where $Q_b=4$ (degree-one SH: 1 DC + 3 directional) [S1, unofficial].
*   **Transfer Function Representation:**
    *   Base preset $T_0$ and target preset $T$ [S1, unofficial].
    *   Common grid of $S$ intensity samples $\{h_s\}_{s=1}^S$ [S1, unofficial].
    *   Raw TF curves averaged onto $B=64$ intensity bins: $\mathbf{R}^{(B)}_T \in [0, 1]^{M \times B \times 4}$ [S1, unofficial].
    *   Region channel $m \in \{0, \dots, M-1\}$ [S1, unofficial].
    *   Intensity-bin index $b \in \{0, \dots, B-1\}$ [S1, unofficial].
*   **Local Support Data (per primitive $i$):**
    *   $K_i \le 27$: Number of valid samples in the $3 \times 3 \times 3$ voxel window around $\boldsymbol{\mu}_i$ [S1, unofficial].
    *   $\mathbf{m}_{ik} \in \{0, \dots, M-1\}$: Region channel of the $k$-th valid sample [S1, unofficial].
    *   $\mathbf{b}_{ik} \in \{0, \dots, B-1\}$: Intensity-bin index of the $k$-th valid sample [S1, unofficial].
    *   $w_{ik} \in [0, 1]$: Normalized nonnegative weight for the $k$-th sample, such that $\sum_{k=1}^{K_i} w_{ik} = 1$ [S1, unofficial].
    *   $u_{ik} = \mathbf{m}_{ik} \cdot B + \mathbf{b}_{ik}$: Encoded integer identifier for the pair $(\mathbf{m}_{ik}, \mathbf{b}_{ik})$ [S1, unofficial].

**Parameters:**
*   $B = 64$: Number of intensity bins for TF discretization [S1, unofficial].
*   $S$: Number of intensity samples in the common grid [S1, unofficial].
*   $M$: Number of segmented regions/materials [S1, unofficial].
*   $H, r$: Hidden and output dimensions of the pointwise MLP $\phi_\eta$ [S1, unofficial].
*   $T_0$: Designated base condition preset. The method is centered on this preset, giving exact identity at $T_0$ [S1, unofficial].
*   $h$: Calibrated scalar intensity (HU for CT, normalized scalar for MR) [S1, unofficial].

**Steps:**

1.  **Initialization:**
    *   Initialize Gaussian primitives with centers $\boldsymbol{\mu}_i$, covariances $\boldsymbol{\Sigma}_i$, and opacity logits $o_i$. The base opacity is $\sigma(o_i)$ where $\sigma$ is the logistic sigmoid [S1, unofficial].
    *   Initialize the pointwise MLP $\phi_\eta(\mathbf{v}) = \mathbf{W}_2 \text{ReLU}(\mathbf{W}_1 \mathbf{v})$ with weights $\mathbf{W}_1 \in \mathbb{R}^{H \times 4}$ and $\mathbf{W}_2 \in \mathbb{R}^{r \times H}$ [S1, unofficial].
    *   At initialization, for each primitive $i$, sample the $3 \times 3 \times 3$ voxel window around it [S1, unofficial]. Identify valid samples $k$ with region channel $\mathbf{m}_{ik}$ and intensity-bin index $\mathbf{b}_{ik}$ [S1, unofficial]. Count valid samples $K_i \le 27$ [S1, unofficial]. Compute normalized nonnegative weights $w_{ik}$ such that $\sum_{k=1}^{K_i} w_{ik} = 1$ [S1, unofficial]. Define empirical joint mass $p_i(m, b) = \sum_{k=1}^{K_i} w_{ik} \cdot \mathbb{1}[(\mathbf{m}_{ik}, \mathbf{b}_{ik}) = (m, b)]$ [S1, unofficial]. Encode each pair $(\mathbf{m}_{ik}, \mathbf{b}_{ik})$ as a single integer $u_{ik} = \mathbf{m}_{ik} \cdot B + \mathbf{b}_{ik}$ [S1, unofficial].

2.  **TF Discretization:**
    *   Export every preset on the common grid of $S$ intensity samples $\{h_s\}$ to create the sampled input tensor $\mathbf{R}_T \in [0, 1]^{M \times S \times 4}$ [S1, unofficial].
    *   Average raw TF curves onto $B=64$ intensity bins to get $\mathbf{R}^{(B)}_T \in [0, 1]^{M \times B \times 4}$ [S1, unofficial].

3.  **Appearance Conditioning (at inference for preset $T$):**
    *   Compute the raw, unpremultiplied RGBA change for preset $T$ relative to base $T_0$:
        $$ \delta \mathbf{r}_T(m, b) = \mathbf{R}^{(B)}_T(m, b) - \mathbf{R}^{(B)}_{T_0}(m, b) \in \mathbb{R}^4 $$
        where $\mathbf{R}_T(m, h) = (\mathbf{p}_T(m, h), a_T(m, h)) \in [0, 1]^3 \times [0, 1]$ [S1, unofficial].
    *   **Physical Lookup:** Compute the physical local lookup offset $\mathbf{d}_i(T)$ by weighted mean of $\delta \mathbf{r}_T$ over the primitive's samples:
        $$ \mathbf{d}_i(T) = \sum_{k=1}^{K_i} w_{ik} \cdot \delta \mathbf{r}_T(\mathbf{m}_{ik}, \mathbf{b}_{ik}) $$
        If $K_i = 0$, set $\mathbf{d}_i(T) = \mathbf{0}$ [S1, unofficial].
    *   **Learned Residual:** Embed each RGBA difference using the bias-free pointwise MLP $\phi_\eta(\mathbf{v}) = \mathbf{W}_2 \text{ReLU}(\mathbf{W}_1 \mathbf{v})$. Pool the local codes for primitive $i$ using weighted sum:
        $$ \mathbf{z}_{i,T} = \sum_{k=1}^{K_i} w_{ik} \cdot \phi_\eta(\delta \mathbf{r}_T(\mathbf{m}_{ik}, \mathbf{b}_{ik})) \in \mathbb{R}^r $$
        If $K_i = 0$, set $\mathbf{z}_{i,T} = \mathbf{0}$ [S1, unofficial].
    *   **Apply Appearance:** Both $\mathbf{d}_i(T)$ and $\mathbf{z}_{i,T}$ modify only the view-independent (DC) color and opacity. Directional SH coefficients remain TF-independent [S1, unofficial]. The backbone opacity passed to the rasterizer is $\sigma(o_i)s_{\text{view}_i}(\pi)$, where $s_{\text{view}_i}(\pi)$ is the TF-independent opacity attenuation [S1, unofficial]. The final DC color and opacity are adjusted by $\mathbf{d}_i(T)$ and $\mathbf{z}_{i,T}$ [S1, unofficial]. [G: The exact projection from $\mathbb{R}^r$ to the $\mathbb{R}^4$ color/opacity space is not specified; typically a small linear layer or bias-free MLP maps $\mathbf{z}_{i,T}$ to additive offsets, which are clamped to valid ranges.]

4.  **Pruning:**
    *   Prune primitives based on maximum opacity over the training bank of TFs to preserve coverage and protect primitives needed by different training TFs [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sigmoid(x):
    return 1.0 / (1.0 + np.exp(-x))

def phi_eta(v, W1, W2):
    """Bias-free pointwise MLP: phi_eta(v) = W2 ReLU(W1 v)."""
    return W2 @ np.maximum(0, W1 @ v)

def factor_splat_inference(
    primitives,  # List of dicts with keys: mu, Sigma, o, C_base, m_ik, b_ik, w_ik, K_i
    tf_curves,   # Dict: preset_id -> (M, S, 4) array of RGBA values
    base_preset_id, # str: ID of T0
    target_preset_id, # str: ID of T
    M,           # int: number of regions
    B,           # int: number of intensity bins (64)
    S,           # int: number of intensity samples
    W1, W2       # np.arrays for MLP phi_eta (W1: Hx4, W2: rxH)
):
    """
    Computes the appearance-modified DC color and opacity for each primitive
    for a given target transfer function preset.
    """
    # 1. Discretize TFs
    # Average raw TF curves onto B bins. 
    # Note: The exact binning algorithm from the sources is not provided. [G]
    R_B_base = np.zeros((M, B, 4))
    R_B_target = np.zeros((M, B, 4))
    # [G] Implementation of binning from S samples to B bins goes here.
    
    # 2. Compute RGBA change delta_r
    # delta_r_T(m, b) = R_T^(B)(m, b) - R_T0^(B)(m, b)
    delta_r = R_B_target - R_B_base  # Shape: (M, B, 4)
    
    results = []
    
    for prim in primitives:
        K_i = prim['K_i']
        m_ik = prim['m_ik']  # Array of shape (K_i,)
        b_ik = prim['b_ik']  # Array of shape (K_i,)
        w_ik = prim['w_ik']  # Array of shape (K_i,)
        C_base = prim['C_base'] # Shape: (4, 3) -> 1 DC + 3 directional
        o = prim['o']
        s_view = prim.get('s_view', 1.0) # TF-independent opacity attenuation
        
        # Handle edge case: no valid samples
        if K_i == 0:
            d_i = np.zeros(4) # (d_c_i, d_alpha_i) in R^3 x R
            z_i_T = np.zeros(W2.shape[0])
        else:
            # 3. Physical Lookup d_i(T)
            # delta_r_T(m_ik, b_ik)
            delta_r_samples = delta_r[m_ik, b_ik, :]  # Shape: (K_i, 4)
            
            # Weighted mean
            d_i = np.sum(w_ik[:, np.newaxis] * delta_r_samples, axis=0) # Shape: (4,)
            
            # 4. Learned Residual z_i,T
            # phi_eta(v) = W2 ReLU(W1 v)
            # Embed each RGBA difference
            phi_eta_samples = np.array([phi_eta(v, W1, W2) for v in delta_r_samples]) # Shape: (K_i, r)
            
            # Weighted sum
            z_i_T = np.sum(w_ik[:, np.newaxis] * phi_eta_samples, axis=0) # Shape: (r,)
            
        # 5. Apply to DC color and opacity
        # C_base[0] is DC color (RGB)
        # d_i contains color offset (3) and opacity offset (1)
        dc_color_offset = d_i[:3]
        alpha_offset = d_i[3]
        
        # [G] Projection of z_i_T to R^4 for color/opacity residual is not specified.
        # Assuming a linear projection P_z: R^r -> R^4 for demonstration.
        # In practice, this would be a learned layer or fixed mapping.
        z_proj = np.zeros(4) # Placeholder for [G: projection of z_i_T]
        
        # Final DC Color
        final_dc_color = C_base[0] + dc_color_offset + z_proj[:3]
        
        # Final Opacity
        # Backbone opacity is sigma(o) * s_view
        base_opacity = sigmoid(o) * s_view
        final_opacity = np.clip(base_opacity + alpha_offset + z_proj[3], 0.0, 1.0)
        
        results.append({
            'dc_color': final_dc_color,
            'opacity': final_opacity,
            'directional_sh': C_base[1:] # Unchanged
        })
        
    return results
```

**Edge Cases and Pitfalls:**
*   If a primitive has no valid samples ($K_i = 0$), the lookup offset $\mathbf{d}_i$ and residual $\mathbf{z}_{i,T}$ are set to $\mathbf{0}$ [S1, unofficial].
*   The region index $m$ only selects the corresponding TF curve; permuting the curve axis and identifiers together leaves the lookup unchanged [S1, unofficial].
*   Centering the branches on an authored base TF $T_0$ ensures exact identity at $T_0$ (i.e., $\mathbf{d}_i(T_0) = \mathbf{0}$ and $\mathbf{z}_{i,T_0} = \mathbf{0}$) [S1, unofficial].

## Subtopics

### [Factorized Appearance Response](4717-factorized-appearance-response.md)
Factorized Appearance Response is the mechanism within FactorSplat that maps a transfer function (TF) $T$ to per-Gaussian color and opacity adjustments using low-rank factorizations. It enables appearance changes without retraining by computing residuals relative to a base appearance, conditioned on the TF. It differs from standard Gaussian Splatting by decoupling the geometric primitives from the appearance logic, allowing multiple TFs to share the same underlying Gaussian proxy.

## Operating characteristics
*   **Time Complexity:** The appearance conditioning is computed once per TF switch [S1, unofficial]. The lookup and residual computation depend on the number of primitives $N$ and the local support size $K_i \le 27$ [S1, unofficial]. The MLP evaluation is constant time per sample [S1, unofficial]. Rasterization remains $O(N)$ per pixel [S1, unofficial].
*   **Memory Complexity:** Storage includes the Gaussian primitives ($O(N)$), the TF curves ($O(M \cdot S \cdot 4)$), and the MLP weights ($O(H \cdot 4 + r \cdot H)$) [S1, unofficial]. The local support data is stored per primitive ($O(N)$) [S1, unofficial].
*   **Measured Performance:** On seven CT and MR scans, FactorSplat improves mean PSNR and changed-region error over region-aware VEG across validation, interpolation, unseen composition, and out-of-distribution (OOD) edits [S1, unofficial].
*   **Strengths:** Allows appearance changes without retraining or refitting. Decouples geometry from appearance. Handles region-specific intensity TFs [S1, unofficial].
*   **Limitations:** Requires region segmentation. Assumes a common intensity grid. Performance depends on the quality of the base proxy and the TF curves [S1, unofficial].
*   **When to use:** When dynamic appearance changes are needed for a fixed geometry, especially in medical volume rendering with region-specific TFs [S1, unofficial].
*   **When not to use:** When geometry also needs to change with the TF, or when region segmentation is not available [S1, unofficial].

## Background
FactorSplat was introduced in the paper "FactorSplat: Appearance-Controllable Gaussian Proxies for Medical Volume Rendering" [S1, unofficial]. It builds on N-DGS [S1, unofficial] and addresses the limitations of image-trained Gaussian proxies that bake one transfer function into their appearance [S1, unofficial]. It is compared to VEG [S1, unofficial] and DeepONet [S1, unofficial].

## Sources
[S1] FactorSplat: Appearance-Controllable Gaussian Proxies for Medical Volume Rendering, https://doi.org/10.48550/arxiv.2610.02382, fetched 2026-10-26 (unofficial)
