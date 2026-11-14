# Proposal Supervision Loss

Also called: Lprop

## Overview
Proposal Supervision Loss ($L_{prop}$) is a regularization term that supervises a proposal MLP by minimizing excess histogram mass that violates a bound established between histograms of 1D data [S1]. It encourages consistency between proposal weights $(\hat{t}, \hat{w})$ and NeRF weights $(t, w)$ despite their bin locations differing [S1]. It sets itself apart from standard photometric losses by operating directly on the volume rendering histogram weights rather than rendered pixel colors [S1].

## How it works
**Inputs:** NeRF bin endpoints $t$, NeRF weights $w$, proposal bin endpoints $\hat{t}$, proposal weights $\hat{w}$ [S1].
**Outputs:** Scalar loss value $L_{prop}$ [S1].
**Data Structures:** Two arrays of floats for each histogram: sorted bin boundaries and corresponding integration weights (opacity contributions) derived from the volume rendering equation [G].
**Steps:**
1. Construct histograms $(t, w)$ from points $\bar{x}$ and $(\hat{t}, \hat{w})$ from points $\bar{\bar{x}}$ [S1].
2. Establish a bound implied by $(\hat{t}, \hat{w})$ on $(t, w)$ [S1].
3. If $\bar{x} = \bar{\bar{x}}$, the bound is guaranteed to be an upper bound on $(t, w)$, resulting in zero loss [S1].
4. If $\bar{x} \neq \bar{\bar{x}}$, compute the loss for excess mass in $(t, w)$ that exceeds the established bound [S1].
**Formula:** $L_{prop} = \sum_{i} \max(0, w_i - w_{\text{bound}, i})$, where $w_{\text{bound}, i}$ is the upper bound for weight $w_i$ implied by the proposal distribution [S1].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def proposal_supervision_loss(t, w, t_hat, w_hat):
    # Step 1: Histograms (t, w) and (t_hat, w_hat) are constructed from points x_bar and x_double_bar [S1].
    # Step 2: Establish bound implied by (t_hat, w_hat) on (t, w) [S1].
    # The exact geometric calculation of the upper bound w_bound when t != t_hat is not provided in the sources [G].
    w_bound = [0.0] * len(t)  # Implementation missing in sources [G]
    
    # Step 3 & 4: If points are identical, loss is zero. Otherwise, compute excess mass [S1].
    if t == t_hat:
        return 0.0
    excess = sum(max(0.0, w[i] - w_bound[i]) for i in range(len(w)))
    return excess
```
**Parameters:** None explicitly defined in the sources other than the input tensors [S1].
**Edge Cases:** If the sampling points are identical ($\bar{x} = \bar{\bar{x}}$), the bound is guaranteed to be an upper bound and the loss is exactly zero [S1]. Numerical precision issues may arise when bin endpoints are very close but not identical [G].

## Subtopics

### Outer Loss
A variant that ensures proposal weights form an upper envelope on the actual weights [S2, unofficial]. It computes the outer measure of the environment weights `w_env` for the target `t` using `_, w_outer = inner_outer(t, t_env, w_env)` [S2, unofficial]. It applies a scaled half-quadratic loss on the excess: `(w - w_outer).clamp_min(0) ** 2 / (w + eps)` [S2, unofficial]. The parameter `eps` is the machine epsilon for the dtype of `t` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources. [G]
- **Memory Complexity:** Not given in the sources. [G]
- **Strengths:** Encourages the proposal network to learn the true density distribution and maintain consistency with the NeRF weights despite different bin locations [S1].
- **Limitations:** Requires access to both NeRF and proposal weights; the bound calculation for differing bin locations is complex and not fully detailed in the sources [S1].

## Background
Proposal Supervision Loss was introduced in the context of Mip-NeRF 360 to improve consistency between coarse and fine sampling networks [S1]. It addresses the issue where the proposal network might not accurately reflect the density distribution of the main network, which can lead to inefficient sampling or artifacts [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
