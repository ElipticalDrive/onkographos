# FactorSplat


## Overview
FactorSplat is a per-scene N-dimensional Gaussian splatting (N-DGS) proxy that accepts region-specific intensity-to-RGBA transfer function (TF) curves at inference time, allowing appearance changes without retraining or refitting [S1, unofficial]. It solves the problem of image-trained Gaussian proxies baking one transfer function into their appearance, which prevents supporting new presets without rebuilding the proxy [S1, unofficial]. Unlike standard Gaussian Splatting where geometry and appearance are tightly coupled to a single rendering configuration, FactorSplat decouples the geometric primitives from the appearance logic, allowing multiple TFs to share the same underlying Gaussian proxy [S1, unofficial]. It differs from region-aware VEG by using region-specific intensity TFs and cinematic-rendering supervision, combining direct lookup with an image-trained N-DGS proxy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ Gaussian primitives defined by centers $\boldsymbol{\mu}_i$, covariances $\boldsymbol{\Sigma}_i$, and opacity logits $o_i$. A set of $M$ segmented regions/materials. A target transfer function preset $T$, defined by region-specific intensity-to-RGBA curves.
*   **Output:** Rendered image with colors and opacities adjusted for preset $T$, while maintaining the shared geometry and directional appearance of the base proxy.

**Data Structures:**
*   **Gaussian Primitives:** For each primitive $i$:
    *   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Center.
    *   $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance.
    *   $o_i \in \mathbb{R}$: Opacity logit.
    *   $\mathbf{C}^{\text{base}}_i \in \mathbb{R}^{Q_b \times 3}$: Spherical Harmonic (SH) coefficients for RGB, where $Q_b=4$ (degree-one SH: 1 DC + 3 directional).
    *   $\mathbf{z}_i \in \mathbb{R}^r$: Learned low-rank residual vector (initialized to zero).
*   **Transfer Function Representation:**
    *   Base preset $T_0$ and target preset $T$.
    *   Common grid of $S$ intensity samples $\{h_s\}_{s=1}^S$.
    *   Raw TF curves averaged onto $B=64$ intensity bins: $\mathbf{R}^{(B)}_T \in [0, 1]^{M \times B \times 4}$.
    *   Region channel $m \in \{0, \dots, M-1\}$.
    *   Intensity-bin index $b \in \{0, \dots, B-1\}$.
*   **Local Support Data (per primitive $i$):**
    *   $K_i \le 27$: Number of valid samples in the $3 \times 3 \times 3$ voxel window around $\boldsymbol{\mu}_i$.
    *   $\mathbf{m}_{ik} \in \{0, \dots, M-1\}$: Region channel of the $k$-th valid sample.
    *   $\mathbf{b}_{ik} \in \{0, \dots, B-1\}$: Intensity-bin index of the $k$-th valid sample.
    *   $w_{ik} \in [0, 1]$: Normalized nonnegative weight for the $k$-th sample, such that $\sum_{k=1}^{K_i} w_{ik} = 1$.
    *   $u_{ik} = \mathbf{m}_{ik} \cdot B + \mathbf{b}_{ik}$: Encoded integer identifier for the pair $(\mathbf{m}_{ik}, \mathbf{b}_{ik})$.

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
    *   Initialize the learned residual vectors $\mathbf{z}_i = \mathbf{0}$ [S1, unofficial].
    *   Initialize the pointwise MLP $\phi_\eta(\mathbf{v}) = \mathbf{W}_2 \text{ReLU}(\mathbf{W}_1 \mathbf{v})$ with weights $\mathbf{W}_1 \in \mathbb{R}^{H \times 4}$ and $\mathbf{W}_2 \in \mathbb{R}^{r \times H}$ [S1, unofficial].

2.  **TF Discretization:**
    *   Export every preset on the common grid of $S$ intensity samples $\{h_s\}$ to create the sampled input tensor $\mathbf{R}_T \in [0, 1]^{M \times S \times 4}$ [S1, unofficial].
    *   Average raw TF curves onto $B=64$ intensity bins to get $\mathbf{R}^{(B)}_T \in [0, 1]^{M \times B \times 4}$ [S1, unofficial].

3.  **Local Support Sampling (at initialization):**
    *   For each primitive $i$, sample the $3 \times 3 \times 3$ voxel window around it [S1, unofficial].
    *   Identify valid samples $k$ with region channel $\mathbf{m}_{ik}$ and intensity-bin index $\mathbf{b}_{ik}$ [S1, unofficial].
    *   Count valid samples $K_i \le 27$ [S1, unofficial].
    *   Compute normalized nonnegative weights $w_{ik}$ such that $\sum_{k=1}^{K_i} w_{ik} = 1$ [S1, unofficial].
    *   Define empirical joint mass $p_i(m, b) = \sum_{k=1}^{K_i} w_{ik} \cdot \mathbb{1}[(\mathbf{m}_{ik}, \mathbf{b}_{ik}) = (m, b)]$ [S1, unofficial].
    *   Encode each pair $(\mathbf{m}_{ik}, \mathbf{b}_{ik})$ as a single integer $u_{ik} = \mathbf{m}_{ik} \cdot B + \mathbf{b}_{ik}$ [S1, unofficial].

4.  **Appearance Conditioning (at inference for preset $T$):**
    *   Compute the raw, unpremultiplied RGBA change for preset $T$ relative to base $T_0$:
        $$ \delta \mathbf{r}_T(m, b) = \mathbf{R}^{(B)}_T(m, b) - \mathbf{R}^{(B)}_{T_0}(m, b) \in \mathbb{R}^4 $$
        where $\mathbf{R}_T(m, h) = (\mathbf{p}_T(m, h), a_T(m, h)) \in [0, 1]^3 \times [0, 1]$ [S1, unofficial].
    *   **Physical Lookup:** Compute the physical local lookup offset $\mathbf{d}_i(T)$ by weighted mean of $\delta \mathbf{r}_T$ over the primitive's samples:
        $$ \mathbf{d}_i(T) = \sum_{k=1}^{K_i} w_{ik} \cdot \delta \mathbf{r}_T(\mathbf{m}_{ik}, \mathbf{b}_{ik}) $$
        If $K_i = 0$, set $\mathbf{d}_i(T) = \mathbf{0}$ [S1, unofficial].
    *   **Learned Residual:** Embed each RGBA difference using the bias-free pointwise MLP $\phi_\eta(\mathbf{v}) = \mathbf{W}_2 \text{ReLU}(\mathbf{W}_1 \mathbf{v})$. Pool the local codes for primitive $i$ using weighted sum:
        $$ \mathbf{z}_{i,T} = \sum_{k=1}^{K_i} w_{ik} \cdot \phi_\eta(\delta \mathbf{r}_T(\mathbf{m}_{ik}, \mathbf{b}_{ik})) \in \mathbb{R}^r $$
        If $K_i = 0$, set $\mathbf{z}_{i,T} = \mathbf{0}$ [S1, unofficial].
    *   **Apply Appearance:** Both $\mathbf{d}_i(T)$ and $\mathbf{z}_{i,T}$ modify only the view-independent (DC) color and opacity. Directional SH coefficients remain TF-independent [S1, unofficial].
        *   Let $\mathbf{C}^{\text{base}}_{i,0} \in \mathbb{R}^3$ be the DC color coefficient from $\mathbf{C}^{\text{base}}_i$.
        *   Let $\alpha_i = \sigma(o_i) s_{\text{view}_i}(\pi)$ be the backbone opacity, where $s_{\text{view}_i}(\pi)$ is the TF-independent opacity attenuation [S1, unofficial].
        *   The final DC color $\mathbf{C}^{\text{final}}_{i,0}$ and opacity $\alpha^{\text{final}}_i$ are adjusted by $\mathbf{d}_i(T)$ and $\mathbf{z}_{i,T}$. The exact mapping from $\mathbf{z}_{i,T}$ to color/opacity offset is not explicitly detailed in the sources beyond being a residual correction [S1, unofficial]. [G: Typically, $\mathbf{z}_{i,T}$ is projected to $\mathbb{R}^4$ (RGB+opacity) and added to the base values, clamped to valid ranges.]

5.  **Pruning:**
    *   Prune primitives based on maximum opacity over the training bank of TFs to preserve coverage and protect primitives needed by different training TFs [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def factor_splat_inference(
    primitives,  # List of dicts with keys: mu, Sigma, o, C_base, z
    tf_curves,   # Dict: preset_id -> (M, S, 4) array of RGBA values
    base_preset_id, # str: ID of T0
    target_preset_id, # str: ID of T
    M,           # int: number of regions
    B,           # int: number of intensity bins (64)
    S,           # int: number of intensity samples
    W1, W2       # np.arrays for MLP phi_eta
):
    """
    Computes the appearance-modified DC color and opacity for each primitive
    for a given target transfer function preset.
    """
    # 1. Discretize TFs
    # Average raw TF curves onto B bins. 
    # Note: In practice, this is done once at initialization or when TFs change.
    # Here we assume R_B is precomputed or computed on the fly.
    R_B_base = average_tf_bins(tf_curves[base_preset_id], S, B)
    R_B_target = average_tf_bins(tf_curves[target_preset_id], S, B)
    
    # 2. Compute RGBA change delta_r
    # delta_r_T(m, b) = R_T^(B)(m, b) - R_T0^(B)(m, b)
    delta_r = R_B_target - R_B_base  # Shape: (M, B, 4)
    
    results = []
    
    for prim in primitives:
        mu = prim['mu']
        K_i = prim['K_i']
        m_ik = prim['m_ik']  # Array of shape (K_i,)
        b_ik = prim['b_ik']  # Array of shape (K_i,)
        w_ik = prim['w_ik']  # Array of shape (K_i,)
        z_learned = prim['z'] # Array of shape (r,)
        C_base = prim['C_base'] # Shape: (4, 3) -> 1 DC + 3 directional
        o = prim['o']
        
        # Handle edge case: no valid samples
        if K_i == 0:
            d_i = np.zeros(4) # Or (3,) for color? Source says d_i = (d_c_i, d_alpha_i) in R^3 x R
            z_i_T = np.zeros_like(z_learned)
        else:
            # 3. Physical Lookup d_i(T)
            # delta_r_T(m_ik, b_ik)
            # m_ik and b_ik are indices into delta_r
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
        # o is opacity logit
        # d_i contains color offset (3) and opacity offset (1)? 
        # Source says d_i = (d_c_i, d_alpha_i) in R^3 x R. 
        # Let's assume d_i[:3] is color offset, d_i[3] is opacity offset.
        
        dc_color_offset = d_i[:3]
        alpha_offset = d_i[3]
        
        # z_i_T is residual for DC color and opacity. 
        # Source doesn't specify exact mapping from R^r to R^4. 
        # [G] Assuming a linear projection or that z_i_T is already in R^4? 
        # Source says z_i,T in R^r. 
        # Let's assume there is a projection layer P_z: R^r -> R^4 for color/opacity.
        # Since not given, we note this as a gap. 
        # However, source says "Both branches modify only the view-independent (DC) color and opacity."
        # And "z_i,T ... model the cinematic renderer's residual response."
        # We will assume a simple projection for the sake of completeness, marked [G].
        # In the actual paper, there might be a small MLP or linear layer.
        # For this article, we state the gap.
        
        # Final DC Color
        final_dc_color = C_base[0] + dc_color_offset + [G: projection of z_i_T to RGB]
        
        # Final Opacity
        # Backbone opacity is sigma(o) * s_view
        # d_i has opacity offset. 
        # z_i_T has opacity residual.
        # [G] Assuming alpha_offset is added to the logit or opacity directly?
        # Source says "modify ... opacity". Usually additive to opacity or logit.
        # Let's assume additive to opacity for simplicity, clamped.
        base_opacity = sigmoid(o)
        final_opacity = np.clip(base_opacity + alpha_offset + [G: projection of z_i_T to alpha], 0, 1)
        
        results.append({
            'dc_color': final_dc_color,
            'opacity': final_opacity,
            'directional_sh': C_base[1:] # Unchanged
        })
        
    return results

def phi_eta(v, W1, W2):
    """Bias-free pointwise MLP."""
    return W2 @ np.maximum(0, W1 @ v)

def average_tf_bins(tf_array, S, B):
    """
    Averages TF curves from S samples to B bins.
    tf_array shape: (M, S, 4)
    Returns shape: (M, B, 4)
    """
    # This is a placeholder for the binning logic.
    # In practice, this involves histogramming or averaging intensity values.
    # The source says "Average raw TF curves onto B=64 intensity bins".
    # We assume the intensity grid {h_s} is mapped to B bins.
    # [G] Specific binning algorithm not provided.
    pass
```

**Edge Cases and Pitfalls:**
*   If a primitive has no valid samples ($K_i = 0$), the lookup offset $\mathbf{d}_i$ and residual $\mathbf{z}_{i,T}$ are set to $\mathbf{0}$ [S1, unofficial].
*   The region index $m$ only selects the corresponding TF curve; permuting the curve axis and identifiers together leaves the lookup unchanged [S1, unofficial].
*   Centering the branches on an authored base TF $T_0$ ensures exact identity at $T_0$ (i.e., $\mathbf{d}_i(T_0) = \mathbf{0}$ and $\mathbf{z}_{i,T_0} = \mathbf{0}$) [S1, unofficial].

## Subtopics

### [Factorized Appearance Response](4717-factorized-appearance-response.md)
Factorized Appearance Response is the mechanism within FactorSplat that maps a transfer function (TF) $T$ to per-Gaussian color and opacity adjustments using low-rank factorizations. It enables appearance changes without retraining by computing residuals relative to a base appearance, conditioned on the TF. It differs from standard Gaussian Splatting by decoupling the geometric primitives from the appearance logic, allowing multiple TFs to share the same underlying Gaussian proxy.

## Operating characteristics
*   **Time Complexity:** The appearance conditioning is computed once per TF switch. The lookup and residual computation depend on the number of primitives $N$ and the local support size $K_i \le 27$. The MLP evaluation is constant time per sample. Rasterization remains $O(N)$ per pixel.
*   **Memory Complexity:** Storage includes the Gaussian primitives ($O(N)$), the TF curves ($O(M \cdot S \cdot 4)$), and the MLP weights ($O(H \cdot 4 + r \cdot H)$). The local support data is stored per primitive ($O(N)$).
*   **Measured Performance:** On seven CT and MR scans, FactorSplat improves mean PSNR and changed-region error over region-aware VEG across validation, interpolation, unseen composition, and out-of-distribution (OOD) edits [S1, unofficial].
*   **Strengths:** Allows appearance changes without retraining or refitting. Decouples geometry from appearance. Handles region-specific intensity TFs.
*   **Limitations:** Requires region segmentation. Assumes a common intensity grid. Performance depends on the quality of the base proxy and the TF curves.
*   **When to use:** When dynamic appearance changes are needed for a fixed geometry, especially in medical volume rendering with region-specific TFs.
*   **When not to use:** When geometry also needs to change with the TF, or when region segmentation is not available.

## Background
FactorSplat was introduced in the paper "FactorSplat: Appearance-Controllable Gaussian Proxies for Medical Volume Rendering" [S1, unofficial]. It builds on N-DGS [S1, unofficial] and addresses the limitations of image-trained Gaussian proxies that bake one transfer function into their appearance. It is compared to VEG [S1, unofficial] and DeepONet [S1, unofficial].

## Sources
[S1] FactorSplat: Appearance-Controllable Gaussian Proxies for Medical Volume Rendering, https://doi.org/10.48550/arxiv.2610.02382, fetched 2026-10-26 (unofficial)
