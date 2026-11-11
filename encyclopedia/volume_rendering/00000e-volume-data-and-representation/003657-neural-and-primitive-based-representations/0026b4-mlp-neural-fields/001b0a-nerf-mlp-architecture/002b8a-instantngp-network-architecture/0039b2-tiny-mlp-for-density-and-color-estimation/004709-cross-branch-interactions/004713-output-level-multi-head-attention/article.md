# Output-level Multi-head Attention

Also called: III. Output-level Multi-head Attention

## Overview
Output-level Multi-head Attention is a module applied to concatenated preliminary densities and colors to learn their interactions before producing final predictions [S1, unofficial]. It sets itself apart from sibling methods by operating on the raw output vectors of the neural branches rather than on input encodings, ensuring that the final density and color estimates are mutually consistent [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Preliminary density $\sigma$ and preliminary color $c$ produced by the preceding MLP branches.
*   **Output:** Final density $\sigma'$ and final color $c'$.

**Data Structures:**
The module operates on a single tensor $\zeta$ formed by concatenating the scalar density and the 3-channel color vector.
*   $\zeta = [\sigma, c]$
*   Shape: $[1, 4]$ per point (1 for density, 3 for RGB).

**Steps:**
1.  **Concatenation:** Combine the preliminary density $\sigma$ and color $c$ into a single vector $\zeta = [\sigma, c]$.
2.  **Attention Setup:** Set the Query ($Q$), Key ($K$), and Value ($V$) matrices to $\zeta$.
3.  **Attention Application:** Apply multi-head attention to $\zeta$ to produce an attended output $\zeta'$.
4.  **Un-concatenation:** Split $\zeta'$ back into its components:
    *   $\sigma'$ is the first dimension of $\zeta'$.
    *   $c'$ consists of the remaining three dimensions of $\zeta'$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def output_level_multi_head_attention(preliminary_sigma, preliminary_color, num_heads=2):
    """
    Applies multi-head attention to concatenated density and color.
    
    Args:
        preliminary_sigma: Tensor of shape (..., 1) or scalar, representing preliminary density.
        preliminary_color: Tensor of shape (..., 3), representing preliminary RGB color.
        num_heads: Integer, number of attention heads (typically 2 for LLFF, 1 for NeRF-Synthetic).
        
    Returns:
        final_sigma: Tensor of shape (..., 1), final density.
        final_color: Tensor of shape (..., 3), final color.
    """
    # Step 1: Concatenate density and color
    # Assuming inputs are batched or have leading dimensions. 
    # We concatenate along the last dimension.
    # sigma shape: (..., 1)
    # color shape: (..., 3)
    # zeta shape: (..., 4)
    zeta = torch.cat([preliminary_sigma, preliminary_color], dim=-1)
    
    # Step 2: Set Q, K, V to zeta
    Q = zeta
    K = zeta
    V = zeta
    
    # Step 3: Apply multi-head attention
    # Note: The specific implementation of MultiHead attention is assumed to be standard.
    # It projects Q, K, V into head dimensions, computes attention, concatenates heads,
    # and projects back.
    # For a single point, this learns the interaction between the density and color channels.
    attended_zeta = multi_head_attention(Q, K, V, num_heads=num_heads)
    
    # Step 4: Un-concatenate attended output
    # The first dimension is density, the remaining three are color.
    final_sigma = attended_zeta[..., 0:1]
    final_color = attended_zeta[..., 1:4]
    
    return final_sigma, final_color
```

**Parameters:**
*   **Number of attention heads ($H$):** 2 for the LLFF dataset; 1 for the NeRF-Synthetic dataset [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Dimension Mismatch:** The un-concatenation step assumes the attention output preserves the channel structure (1 density, 3 color). If the attention mechanism alters the output dimensionality, this step fails.
*   **Over-parameterization:** While not explicitly a pitfall of this specific module, it builds on the input-level module which trims the hash encoding to avoid over-parameterizing view-independent densities [S1, unofficial].

## Subtopics

### Input-level Multi-head Attention
A multi-head attention module applied to hybrid encodings to learn interactions between view directions and hash encoding dimensions, and among neighboring rays [S1, unofficial]. It sets itself apart by operating on the input feature space before the density/color split. Choose it when modeling view-dependent effects and ray correlations early in the pipeline. It uses $H=2$ heads for LLFF and $H=1$ for NeRF-Synthetic. The input tensor $\gamma''(d, x)$ has size $|R| \times (N_x - 1 + N_d)$, where $|R|$ is batch size, $N_x$ is hash encoding dimension, and $N_d$ is view encoding dimension. The hash encoding for density is trimmed to $\gamma'_0(x)$ to avoid over-parameterization, while the hybrid encoding retains all but the first dimension [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of attention heads and the dimensionality of the concatenated vector (which is small, 4).
*   **Memory Complexity:** Minimal, as it operates on small vectors per point.
*   **Strengths:** Explicitly models the correlation between density and color, which can improve consistency in novel view synthesis, particularly in few-shot settings [S1, unofficial].
*   **Limitations:** Adds computational overhead compared to simple concatenation or no interaction.
*   **When to use:** When using the DWTNeRF architecture or similar multi-head attention-based NeRF variants, specifically for refining final predictions [S1, unofficial].
*   **When not:** When computational resources are extremely constrained and the interaction between density and color is assumed to be negligible or handled by the MLPs alone.

## Background
Originating from the DWTNeRF paper, this method is part of a broader approach using multi-head attention for cross-branch interactions in neural radiance fields [S1, unofficial]. It was developed to boost few-shot neural radiance fields via Discrete Wavelet Transform, addressing the need for better interaction modeling between density and color branches in low-data regimes [S1, unofficial].

## Sources
[S1] DWTNeRF: Boosting Few-shot Neural Radiance Fields via Discrete Wavelet Transform, https://arxiv.org/abs/2501.12637v3, fetched 2026-10-27 (unofficial)
