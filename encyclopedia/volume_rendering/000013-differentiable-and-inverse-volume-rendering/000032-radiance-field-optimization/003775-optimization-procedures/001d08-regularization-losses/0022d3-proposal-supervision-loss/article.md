# Proposal Supervision Loss

Also called: Lprop

## Overview
Proposal Supervision Loss ($L_{prop}$) is a regularization term used in differentiable volume rendering to enforce consistency between a lightweight "proposal" network and the main "NeRF" network. It solves the problem of ensuring that the proposal distribution, used for efficient sampling, accurately approximates the true scene density distribution, even when the two networks sample at different spatial locations. It sets itself apart from standard photometric losses by operating on the *weights* of the volume rendering histograms rather than pixel colors, ensuring the proposal MLP bounds the NeRF MLP's emission [S1].

## How it works
The method relies on the mathematical relationship between two histograms of 1D data (bin endpoints $t$ and weights $w$). It constructs a bound implied by the proposal weights $(\hat{t}, \hat{w})$ on the NeRF weights $(t, w)$. If the bin locations are identical, the proposal weights form a strict upper bound on the NeRF weights, resulting in zero loss. If the bin locations differ, the loss is computed as the excess mass in the NeRF histogram that violates this upper bound [S1].

**Inputs:**
- NeRF bin endpoints: $t = [t_1, \dots, t_N]$
- NeRF weights: $w = [w_1, \dots, w_N]$
- Proposal bin endpoints: $\hat{t} = [\hat{t}_1, \dots, \hat{M}]$
- Proposal weights: $\hat{w} = [\hat{w}_1, \dots, \hat{w}_M]$

**Data Structures:**
- Arrays of floats representing the sorted bin boundaries and corresponding integration weights (opacity contributions) from the volume rendering equation.

**Steps:**
1.  **Construct Histograms:** Generate the weight vectors $w$ and $\hat{w}$ from their respective bin endpoints $t$ and $\hat{t}$ using the standard emission-absorption model integration [S1][S5, unofficial].
2.  **Compute Bound:** Calculate the upper bound $w_{bound}$ for the NeRF weights $w$ implied by the proposal weights $\hat{w}$. This involves determining the maximum possible weight $w_i$ at each bin $t_i$ given the proposal distribution $\hat{w}$ and its support $\hat{t}$.
3.  **Calculate Excess:** Compute the difference between the actual NeRF weights and the bound: $\Delta w = \max(0, w - w_{bound})$.
4.  **Loss Formulation:** The loss is the sum (or mean) of this excess mass. If $t = \hat{t}$, $w_{bound} \ge w$ always holds, so $L_{prop} = 0$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def proposal_supervision_loss(t, w, t_hat, w_hat):
    """
    Computes the Proposal Supervision Loss.
    
    Args:
        t: Array of NeRF bin endpoints, shape (N,)
        w: Array of NeRF weights, shape (N,)
        t_hat: Array of Proposal bin endpoints, shape (M,)
        w_hat: Array of Proposal weights, shape (M,)
        
    Returns:
        Scalar loss value.
    """
    # Step 1: Compute the upper bound on w implied by (t_hat, w_hat)
    # This requires calculating the maximum possible weight at each t_i
    # given the proposal histogram. This is non-trivial when t != t_hat.
    # The bound w_bound[i] represents the max weight w[i] can have
    # such that it is consistent with the proposal distribution.
    w_bound = compute_proposal_upper_bound(t, t_hat, w_hat)
    
    # Step 2: Compute excess mass
    # Only penalize if NeRF weight exceeds the proposal's implied bound
    excess = torch.clamp(w - w_bound, min=0.0)
    
    # Step 3: Return sum of excess
    return torch.sum(excess)

def compute_proposal_upper_bound(t, t_hat, w_hat):
    """
    Computes the upper bound on weights w at locations t,
    given the proposal histogram defined by t_hat and w_hat.
    
    This function implements the bound described in [S1].
    If t == t_hat, w_bound = w_hat (assuming sorted and consistent).
    If t != t_hat, it calculates the envelope.
    """
    # Implementation detail: The bound is derived from the fact that
    # the proposal weights define a distribution. The NeRF weights
    # at arbitrary bins t must not exceed the mass available in the
    # proposal distribution that overlaps with those bins.
    # A precise implementation requires integrating the proposal
    # density over the intervals defined by t.
    
    # For identical bins, the bound is exact.
    if torch.allclose(t, t_hat):
        return w_hat
    
    # For different bins, we compute the upper envelope.
    # This is effectively the outer measure of the proposal weights
    # evaluated at the NeRF bin locations.
    # See [S2, unofficial] for the 'Outer Loss' variant which explicitly computes
    # this outer measure.
    
    # Placeholder for the specific geometric calculation of the bound
    # implied by [S1]. The bound ensures that for any interval,
    # the weight assigned by NeRF does not exceed the maximum
    # possible weight allowed by the proposal's histogram structure.
    w_bound = compute_outer_measure(t, t_hat, w_hat)
    
    return w_bound

def compute_outer_measure(t, t_hat, w_hat):
    """
    Computes the outer measure of the proposal weights w_hat
    at the query points t.
    
    This function calculates the smallest upper bound on the
    cumulative distribution or local weights that is consistent
    with the proposal histogram.
    """
    # This function encapsulates the logic to determine if a weight
    # at t[i] is "covered" by the proposal weights.
    # It returns an array of the same shape as t.
    # The exact algorithm involves checking the overlap of intervals
    # [t[i-1], t[i]] with the proposal intervals [t_hat[j-1], t_hat[j]].
    pass
```

**Parameters:**
- None explicitly defined in [S1] other than the input tensors.

**Edge Cases:**
- If $t = \hat{t}$, the loss is guaranteed to be zero [S1].
- Precision issues may arise if bin endpoints are very close but not identical, leading to numerical instability in bound calculation [G].

## Subtopics

### Outer Loss
A variant of Proposal Supervision Loss that explicitly ensures proposal weights form an upper envelope on the actual weights. It computes the outer measure of the environment weights `w_env` for the target `t` and applies a scaled half-quadratic loss on the excess: $(w - w_{outer})^2 / (w + \epsilon)$, clamped to non-negative values. It uses `eps` as machine epsilon for the dtype of `t` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the complexity of computing the upper bound. If bin locations are shared, it is $O(N)$. If they differ, it requires interval overlap calculations, potentially $O(N \log M)$ or $O(N \cdot M)$ depending on implementation [S1].
- **Memory Complexity:** $O(N + M)$ to store the histograms and bounds.
- **Strengths:** Encourages the proposal network to learn the true density distribution without requiring pixel-wise supervision, improving sampling efficiency.
- **Limitations:** Requires access to both NeRF and proposal weights. The bound calculation can be complex when bin locations differ significantly [S1].

## Background
Proposal Supervision Loss was introduced in the context of Mip-NeRF 360 to improve the consistency between the coarse and fine sampling networks. It addresses the issue where the proposal network, used for importance sampling, might not accurately reflect the density distribution of the main network, leading to inefficient sampling or artifacts [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
