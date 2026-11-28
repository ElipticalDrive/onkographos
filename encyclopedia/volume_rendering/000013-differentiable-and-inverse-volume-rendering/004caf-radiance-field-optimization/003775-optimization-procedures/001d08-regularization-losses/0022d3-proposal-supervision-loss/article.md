# Proposal Supervision Loss

Also called: Lprop

## Overview
Proposal Supervision Loss is a regularization technique that supervises a proposal network by minimizing excess histogram mass that violates a bound established between histograms of 1D data [S1]. It solves the problem of aligning sampling distributions when proposal and target networks use different bin locations, setting itself apart from standard weight-matching losses by operating on cumulative mass envelopes rather than pointwise comparisons [S1].

## How it works
The method minimizes the "excess histogram mass" that violates a theoretical upper bound established between two 1D histograms: one derived from the proposal network's weights $(\hat{t}, \hat{w})$ and one from the target network's weights $(t, w)$ [S1].

**Inputs and Outputs:**
*   **Inputs:** Target sample positions $t_i$ and weights $w_i$ for $i=1 \dots N$. Proposal sample positions $\hat{t}_j$ and weights $\hat{w}_j$ for $j=1 \dots M$ [S1].
*   **Output:** A scalar loss value $L_{prop} \ge 0$.

**Data Structures:**
*   Two arrays of floats for positions: $T = [t_1, \dots, t_N]$ and $\hat{T} = [\hat{t}_1, \dots, \hat{t}_M]$.
*   Two arrays of floats for weights: $W = [w_1, \dots, w_N]$ and $\hat{W} = [\hat{w}_1, \dots, \hat{w}_M]$.
*   Cumulative distribution functions (CDFs) derived from the weights.

**Algorithm:**
1.  **Construct Histograms:** Compute histograms $(t, w)$ from points $\bar{x}$ and $(\hat{t}, \hat{w})$ from points $\bar{\bar{x}}$ [S1].
2.  **Establish Bound:** Determine the upper bound on the target CDF imposed by the proposal CDF. For any target bin endpoint $t_k$, the bound $B(t_k)$ is the cumulative weight of the proposal samples that fall within the interval ending at $t_k$ [S1].
    *   If $\bar{x} = \bar{\bar{x}}$ (identical bin locations), the bound is an upper bound on the target weights, and the loss is zero [S1].
    *   If $\bar{x} \neq \bar{\bar{x}}$, the target weights may exceed the mass provided by the proposal weights at specific intervals [S1].
3.  **Compute Excess Mass:** For each target bin $k$, calculate the violation:
    $$
    \text{excess}_k = \max(0, C_t(t_k) - C_{\hat{t}}(t_k))
    $$
    where $C_t$ and $C_{\hat{t}}$ are the cumulative weight functions evaluated at $t_k$ [S1].
4.  **Aggregate Loss:** Sum the excess masses:
    $$
    L_{prop} = \sum_{k=1}^N \max(0, C_t(t_k) - C_{\hat{t}}(t_k))
    $$

**Parameters:**
*   None explicitly defined in the sources other than the sample counts $N$ and $M$.

**Edge Cases:**
*   **Identical Bins:** If sample positions are identical, the loss is guaranteed to be zero [S1].
*   **Precision:** Floating-point errors in cumulative sums should be handled by clamping negative excesses to zero [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sort_pairs(keys, values):
    """Sort values according to keys."""
    paired = sorted(zip(keys, values), key=lambda x: x[0])
    return [p[0] for p in paired], [p[1] for p in paired]

def cumulative_sum(arr):
    """Compute cumulative sum of an array."""
    result = [0.0] * len(arr)
    current = 0.0
    for i, val in enumerate(arr):
        current += val
        result[i] = current
    return result

def searchsorted_right(sorted_arr, value):
    """Find the rightmost insertion point for value in sorted_arr."""
    lo, hi = 0, len(sorted_arr)
    while lo < hi:
        mid = (lo + hi) // 2
        if value >= sorted_arr[mid]:
            lo = mid + 1
        else:
            hi = mid
    return lo

def proposal_supervision_loss(t, w, t_hat, w_hat):
    """
    Computes the Proposal Supervision Loss.
    t, w: arrays of target sample positions and weights.
    t_hat, w_hat: arrays of proposal sample positions and weights.
    """
    t, w = sort_pairs(t, w)
    t_hat, w_hat = sort_pairs(t_hat, w_hat)
    
    C_t = cumulative_sum(w)
    C_hat = cumulative_sum(w_hat)
    
    loss = 0.0
    for i in range(len(t)):
        t_i = t[i]
        # Find cumulative proposal mass up to t_i
        j = searchsorted_right(t_hat, t_i) - 1
        if j >= 0:
            bound = C_hat[j]
        else:
            bound = 0.0
        # Excess mass violating the proposal bound
        excess = max(0.0, C_t[i] - bound)
        loss += excess
    return loss
```

## Subtopics

### Outer Loss
A variant that ensures proposal weights form an upper envelope on the actual weights [S2, unofficial]. It computes the outer measure of environment weights $w_{env}$ for target positions $t$, then applies a scaled half-quadratic loss: $(w - w_{outer})^2 / (w + \epsilon)$ [S2, unofficial]. Parameter $\epsilon$ is the machine epsilon for the dtype of $t$ [S2, unofficial].

### Proposal Network Supervision
A training procedure used in high-resolution GAN-based volume rendering that supervises the proposal network by comparing predicted high-resolution weights against ground truth weights computed from a small high-resolution patch [S3]. It traces 192 coarse samples at high resolution for a $64 \times 64$ patch to compute ground truth weights $P_{patch} \in \mathbb{R}^{192 \times 64 \times 64}$ [S3]. These are cleaned by applying a 1D Gaussian blur, suppressing values $< 5 \times 10^{-3}$, and L1 normalization to create $\bar{P}_{patch}$ [S3]. The loss is the Cross-Entropy between $\bar{P}_{patch}$ and the corresponding patch of predicted weights $\hat{P}_{patch}$: $-\sum \bar{p}_j \log \hat{p}_j$ [S3].

### Prior Model Training Loss
A combined objective function $\mathcal{L}_{prior} = \mathcal{L}_{recon} + \lambda_{prop} \mathcal{L}_{prop}$ that uses $\mathcal{L}_{recon} = \|F_\theta(r, w) - c\|_1$ and sets $\lambda_{prop} = 1$ [S4]. It refrains from regularizing the latent space, disables the distortion loss, and disables the 360-parameterisation or space-warping of MipNeRF360 because the scene is not unbounded [S4]. It is trained for 1 Mio. steps on multi-view images of resolution $512 \times 768$ [S4].

## Operating characteristics
*   **Time Complexity:** $O(N \log M)$ due to sorting and binary search for bin matching, where $N$ and $M$ are the number of samples [S1].
*   **Memory Complexity:** $O(N+M)$ to store the sample positions, weights, and cumulative arrays [S1].
*   **Strengths:** Encourages consistency between proposal and target distributions without requiring identical sample locations [S1].
*   **Limitations:** Requires access to both proposal and target network outputs. May be sensitive to noise in weight estimation if not regularized, necessitating smoothing operations in high-resolution variants [S3].
*   **When to use:** Two-stage sampling strategies in neural radiance fields where coarse proposals guide fine sampling [S1].

## Background
The concept was introduced in Mip-NeRF 360 to supervise a proposal MLP by bounding the weights emitted by a NeRF MLP [S1]. It addresses the difficulty of aligning discrete sample sets from different networks by leveraging the properties of 1D histograms [S1]. The method is applied in volumetric reconstruction pipelines for few-shot ultra high-resolution face synthesis, trained for 1 Mio. steps on $512 \times 768$ images [S4].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S3] What You See is What You GAN: Rendering Every Pixel for High-Fidelity Geometry in 3D GANs, https://doi.org/10.1109/cvpr52733.2024.02148, fetched 2026-10-26
[S4] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
