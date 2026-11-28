# Tempered Dirichlet Update

Also called: DIGS voting rule, Dirichlet–Categorical evidence model update, Evidence update, closed-form tempered update

## Overview
The Tempered Dirichlet Update is a per-Gaussian, per-frame evidence accumulation method that caps the influence of high-weight observations to prevent reinforcement of early association errors in a closed-loop system [S1, unofficial]. It updates Gaussian evidence using continuous rendering weights instead of discrete counts, preventing scale dominance and closed-loop feedback errors by tempering the learning rate [S1, unofficial]. It builds on [Bayesian Inference Surrogate Framework](11307-bayesian-inference-surrogate-framework.md) by applying Bayesian inference to obtain the posterior distribution over parameters conditioned on a user-specified spatial feature.

## How it works
**Inputs and Outputs:**
*   **Input:** Current evidence accumulator $H \in \mathbb{R}^{N \times J}$ (where $N$ is the number of Gaussians and $J$ is the number of labels, with label 0 reserved for background), continuous rendering weights $W^{(t)} \in \mathbb{R}^{N \times J}$ for the current frame $t$, and hyperparameters $\tau > 0$ and $\gamma \in (0, 1)$ [S1, unofficial].
*   **Output:** Updated evidence accumulator $H^{(t)}$ [S1, unofficial].

**Data Structures:**
*   $H$: A 2D tensor of shape $(N, J)$. $H_{i,j}$ stores the accumulated evidence for Gaussian $i$ and label $j$. Initialized to zeros [S1, unofficial].
*   $W^{(t)}$: A 2D tensor of shape $(N, J)$. $W^{(t)}_{i,j}$ represents the continuous rendering weight contributed by Gaussian $i$ to label $j$ in frame $t$. Computed via scatter-add of pixel weights based on rendered labels [S1, unofficial].

**Parameters:**
*   $\tau$: A constant controlling the evidence scale and learning rate decay. It bounds the maximum evidence injected per frame. Value: Not given in the sources.
*   $\gamma$: Visibility-aware decay factor. Value: 0.95 [S1, unofficial].
*   $w_{min}$: A threshold for visibility. Gaussians with total weight below this are not decayed. Value: Not given in the sources.

**Steps:**
1.  **Compute Totals:** For each Gaussian $i$, calculate the total current evidence $S^{(t)}_i$ and the total weight $W^{(t)}_i$ for the current frame:
    $$ S^{(t)}_i = \sum_{j=0}^{J-1} H_{i,j} $$
    $$ W^{(t)}_i = \sum_{j=0}^{J-1} W^{(t)}_{i,j} $$
    [S1, unofficial]

2.  **Calculate Learning Rate:** Compute the per-Gaussian learning rate $\eta^{(t)}_i$:
    $$ \eta^{(t)}_i = \frac{\tau}{S^{(t)}_i + W^{(t)}_i + \tau} $$
    [S1, unofficial]

3.  **Apply Visibility-Aware Decay (Optional but standard in pipeline):** Before updating, decay evidence for sufficiently visible Gaussians to prevent erosion of occluded primitives:
    $$ H_{i,j} \leftarrow \begin{cases} \gamma \cdot H_{i,j} & \text{if } W^{(t)}_i > w_{min} \\ H_{i,j} & \text{otherwise} \end{cases} $$
    [S1, unofficial]

4.  **Update Evidence:** Update the evidence accumulator for each Gaussian $i$ and label $j$:
    $$ H_{i,j} \leftarrow H_{i,j} + \eta^{(t)}_i \cdot W^{(t)}_{i,j} $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tempered_dirichlet_update(H, W_t, tau, gamma, w_min):
    """
    Updates the evidence accumulator H using the Tempered Dirichlet Update.
    
    Args:
        H: Tensor of shape (N, J), current evidence accumulator.
        W_t: Tensor of shape (N, J), continuous rendering weights for current frame.
        tau: Scalar, constant controlling evidence scale and learning rate decay.
        gamma: Scalar, visibility-aware decay factor (default 0.95).
        w_min: Scalar, threshold for visibility-aware decay.
        
    Returns:
        H: Updated evidence accumulator.
    """
    # Step 1: Compute totals
    # S_i is sum over labels j for each Gaussian i
    S = H.sum(dim=1)  # Shape (N,)
    # W_i is sum over labels j for each Gaussian i
    W_total = W_t.sum(dim=1)  # Shape (N,)
    
    # Step 2: Calculate learning rate
    # eta_i = tau / (S_i + W_i + tau)
    eta = tau / (S + W_total + tau)  # Shape (N,)
    
    # Step 3: Apply visibility-aware decay
    # Decay only if total weight for the Gaussian exceeds w_min
    # Note: The source specifies decay per element based on total weight of the Gaussian
    visible_mask = W_total > w_min  # Shape (N,)
    # Expand mask to match H dimensions for broadcasting
    visible_mask_expanded = visible_mask.unsqueeze(1)  # Shape (N, 1)
    H = torch.where(visible_mask_expanded, gamma * H, H)
    
    # Step 4: Update evidence
    # H_ij <- H_ij + eta_i * W_t_ij
    # eta needs to be broadcasted to (N, J)
    eta_expanded = eta.unsqueeze(1)  # Shape (N, 1)
    H = H + eta_expanded * W_t
    
    return H
```

**Edge Cases and Pitfalls:**
*   **Scale Dominance:** Naive additive accumulation ($H_i += W_i^{(t)}$) allows arbitrarily large evidence increments in one frame, giving large-scale Gaussians disproportionate influence [S1, unofficial]. The tempered update bounds this.
*   **Closed-Loop Feedback:** In a closed-loop system, rendered label maps drive next-frame association, potentially reinforcing early errors. The tempered update mitigates this by capping the injected mass [S1, unofficial].
*   **Injection Bound:** The update satisfies $\eta^{(t)}_i \cdot W^{(t)}_i < \tau$ for $\tau > 0, S_i \ge 0, W^{(t)}_i \ge 0$. This ensures the total evidence injected by one frame is bounded by $\tau$ [S1, unofficial].
*   **Zero Evidence:** At zero stored evidence, the injected mass approaches $\tau$ as the vote grows [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Executes in a single parallel pass per frame [S1, unofficial].
*   **Memory Complexity:** Requires storage for the evidence accumulator $H$ of size $N \times J$ [S1, unofficial].
*   **Strengths:** Prevents reinforcement of early association errors; handles continuous weights stably; bounds evidence injection; adapts learning rate based on accumulated evidence without manual scheduling [S1, unofficial].
*   **Limitations:** Requires careful tuning of $\tau$ and $w_{min}$; decay factor $\gamma$ must be chosen to balance erosion of occluded primitives vs. stability [S1, unofficial].
*   **When to Use:** When using continuous rendering weights for evidence accumulation in a closed-loop system where early errors can be reinforced.
*   **When Not to Use:** When discrete counts are sufficient and closed-loop feedback is not a concern.

## Background
The method is motivated by a Dirichlet–Categorical evidence model and generalised Bayesian tempering [S1, unofficial]. It was introduced in the DIGS (Distribution-Informed Gaussian Splatting) paper [S1, unofficial]. The approach addresses instability in direct addition of continuous weights, which can lead to scale dominance and closed-loop feedback errors [S1, unofficial]. It is part of the DIGS pipeline for training-free open-vocabulary 3D segmentation [S1, unofficial].

## Sources
[S1] DIGS: Distribution-Informed Gaussian Splatting for Training-Free Open-Vocabulary 3D Segmentation, https://doi.org/10.48550/arxiv.2610.16177, fetched 2026-10-26 (unofficial)
