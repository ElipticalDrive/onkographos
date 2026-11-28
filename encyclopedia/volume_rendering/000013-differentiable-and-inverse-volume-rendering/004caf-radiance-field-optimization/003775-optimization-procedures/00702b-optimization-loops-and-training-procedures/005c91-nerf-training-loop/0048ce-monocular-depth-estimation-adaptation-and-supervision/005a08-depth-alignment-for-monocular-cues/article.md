# Depth Alignment for Monocular Cues
Also called: Least-squares Depth Alignment, Scale and Shift Alignment

## Overview
Depth Alignment for Monocular Cues solves the scale and shift ambiguity inherent in monocular depth estimation (MDE) by linearly aligning relative MDE outputs with rendered depth values during NeRF training [S1]. It differs from sibling methods like [NerfingMVS](9518-nerfingmvs.md) which use multi-view stereo consistency, by optimizing alignment parameters per mini-batch to enforce geometric consistency against a single-view depth prior [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Rendered depth map $\bar{D}$ from the NeRF, monocular depth estimate $D^*$ from a frozen pre-trained network (e.g., Omnidata), and a set of sampled pixel indices $P$ for the current mini-batch [S1].
*   **Outputs:** Alignment parameters (scale $w$, shift $q$) and the aligned depth loss contribution to the total objective [S1].

**Data Structures:**
*   $D^*$: 2D tensor of shape $(H, W)$, values in $[0, 1]$ (normalized MDE output) [S1].
*   $\bar{D}$: 2D tensor of shape $(H, W)$, rendered depth values [S1].
*   $P$: List of $(u, v)$ pixel coordinates sampled from the image [S1].

**Steps:**
1.  **Sample Pixels:** Select $N$ pixels from the image to compute alignment statistics [S1]. Let the sampled values be $D^*_p$ and $\bar{D}_p$ for $p \in P$ [S1].
2.  **Stop-Gradient:** Apply stop-gradient to $D^*$ to prevent gradients from flowing into the frozen MDE network: $D^*_{sg} = \text{sg}(D^*)$ [S1].
3.  **Least-Squares Optimization:** Solve for scale $w$ and shift $q$ that minimize the squared error between the aligned MDE depth and rendered depth:
    $$ \min_{w, q} \sum_{p \in P} \| (w \cdot D^*_{sg}(p) + q) - \bar{D}(p) \|^2 $$
    The closed-form solution minimizes this objective [S1].
4.  **Apply Alignment:** Compute the aligned MDE depth: $\tilde{D}(p) = w \cdot D^*(p) + q$ [S1].
5.  **Compute Loss:** Calculate the depth consistency loss $L_{depth}$ using the aligned depth $\tilde{D}$ and rendered depth $\bar{D}$ [S1].

**Formulas:**
Let $\mu_{MDE} = \frac{1}{N} \sum_{p \in P} D^*_{sg}(p)$ and $\mu_{R} = \frac{1}{N} \sum_{p \in P} \bar{D}(p)$ [G].
Let $\sigma_{MDE, R} = \frac{1}{N} \sum_{p \in P} (D^*_{sg}(p) - \mu_{MDE})(\bar{D}(p) - \mu_{R})$ [G].
Let $\sigma^2_{MDE} = \frac{1}{N} \sum_{p \in P} (D^*_{sg}(p) - \mu_{MDE})^2$ [G].

$$ w = \frac{\sigma_{MDE, R}}{\sigma^2_{MDE} + \epsilon} $$
$$ q = \mu_{R} - w \cdot \mu_{MDE} $$

Where $\epsilon$ is a small constant (e.g., $10^{-8}$) to prevent division by zero [G].

**Parameters:**
*   $N$: Number of sampled pixels. Typical value: Not given in the sources.
*   $\epsilon$: Regularization constant. Typical value: $10^{-8}$ [G].

**Edge Cases and Pitfalls:**
*   **Zero Variance:** If $\sigma^2_{MDE} \approx 0$ (e.g., flat regions), $w$ becomes unstable [G]. Add $\epsilon$ to the denominator to stabilize computation [G].
*   **Scale Ambiguity:** Without alignment, the NeRF may converge to an arbitrary scale [G]. This method ensures the scale matches the MDE prior relative to the rendered depth [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_alignment(D_mde, D_rendered, num_samples=1024):
    """
    Align monocular depth with rendered depth using least-squares.
    
    Args:
        D_mde: 2D array (H, W), monocular depth estimate [0, 1].
        D_rendered: 2D array (H, W), rendered depth from NeRF.
        num_samples: Number of pixels to sample for alignment.
        
    Returns:
        w: Scale parameter.
        q: Shift parameter.
    """
    import random
    H, W = len(D_mde), len(D_mde[0])
    # Sample random pixel indices
    indices = list(range(H * W))
    sampled_indices = random.sample(indices, min(num_samples, H * W))
    
    d_mde_vals = []
    d_rendered_vals = []
    for idx in sampled_indices:
        u, v = divmod(idx, W)
        d_mde_vals.append(float(D_mde[u][v]))
        d_rendered_vals.append(float(D_rendered[u][v]))
    
    # Stop gradient on MDE depth (detach values)
    d_mde_vals = [float(x) for x in d_mde_vals]
    
    # Compute means
    mu_mde = sum(d_mde_vals) / len(d_mde_vals)
    mu_rendered = sum(d_rendered_vals) / len(d_rendered_vals)
    
    # Compute covariance and variance
    cov = sum((dm - mu_mde) * (dr - mu_rendered) for dm, dr in zip(d_mde_vals, d_rendered_vals)) / len(d_mde_vals)
    var_mde = sum((dm - mu_mde) ** 2 for dm in d_mde_vals) / len(d_mde_vals)
    
    # Compute scale and shift
    epsilon = 1e-8
    w = cov / (var_mde + epsilon)
    q = mu_rendered - w * mu_mde
    
    return w, q
```

## Subtopics

### Depth Normalization for Scale/Shift Invariance
A preprocessing step that normalizes monocular depth estimates to remove scale and shift ambiguities before alignment [S2]. It sets itself apart from direct least-squares alignment by explicitly standardizing depth statistics using mean and mean absolute deviation rather than solving for linear parameters [S2]. Choose this variant when the MDE prior exhibits large dynamic range variations or when comparing depth maps across different scenes requires statistical standardization [S2]. See [Incremental Joint Learning of Depth, Pose, and Implicit Scene Representation on Monocular Camera in Large-Scale Scenes](https://doi.org/10.1109/tase.2025.3617654).

## Operating characteristics
*   **Time Complexity:** The method requires $O(N)$ time per mini-batch for sampling and computing statistics, where $N$ is the number of sampled pixels [G]. This overhead is negligible compared to the NeRF forward pass [G].
*   **Memory Complexity:** The method requires $O(N)$ memory to store sampled depth values [G].
*   **Strengths:** The method is simple to implement, effectively resolves scale ambiguity, and works with any pre-trained MDE model [G].
*   **Limitations:** The method relies heavily on the quality of the MDE prior and assumes a linear relationship between MDE and rendered depth [S1].
*   **When to Use:** Use this method when training NeRFs from monocular images with a reliable MDE prior (e.g., Omnidata) and no sparse point cloud or multi-view stereo data is available [S1].
*   **When Not to Use:** Avoid this method when high-precision absolute scale is required and MDE is known to be inaccurate, or when multi-view consistency is available and preferred [G].

## Background
The method originated in approaches like MonoSDF and Neuralangelo for inferring signed distance functions from monocular images [S1]. It addresses the challenge of using monocular depth priors in volume rendering by aligning them with the rendered depth to enforce geometric consistency [S1, S2]. The method is widely used in recent monocular NeRF and SDF reconstruction pipelines [S1].

## Sources
[S1] Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions, https://doi.org/10.1609/aaai.v39i4.32426, fetched 2026-10-26
[S2] Incremental Joint Learning of Depth, Pose, and Implicit Scene Representation on Monocular Camera in Large-Scale Scenes, https://doi.org/10.1109/tase.2025.3617654, fetched 2026-10-29
