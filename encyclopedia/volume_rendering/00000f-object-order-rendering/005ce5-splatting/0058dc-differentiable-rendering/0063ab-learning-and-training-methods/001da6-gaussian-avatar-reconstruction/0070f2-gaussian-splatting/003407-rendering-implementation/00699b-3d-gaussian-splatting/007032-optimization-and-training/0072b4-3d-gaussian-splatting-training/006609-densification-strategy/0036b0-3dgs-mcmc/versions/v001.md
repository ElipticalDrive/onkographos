# 3DGS-MCMC

Also called: MCMC-based Gaussian Lifecycle

## Overview
3DGS-MCMC [S1, unofficial] replaces the noisy positional gradient magnitude heuristics of standard 3D Gaussian Splatting with a probabilistic framework that interprets the Gaussian cloud as samples from an underlying probability distribution. It uses Markov Chain Monte Carlo (MCMC) sampling to guide the life cycle (spawn/prune) of primitives, specifically addressing the instability and artifacts caused by gradient-based densification [S1, unofficial]. Unlike sibling methods that rely on fixed thresholds or local error maps, 3DGS-MCMC explores regions with unknown gradients through Metropolis–Hastings acceptance criteria, yielding more efficient and artifact-free point clouds [S1, unofficial].

## How it works
**Inputs:** Current set of Gaussians $\mathcal{G}$, rendering loss $\mathcal{L}$, proposal distribution $q$.
**Outputs:** Updated set of Gaussians $\mathcal{G}'$ after densification step.

**Data Structures:**
The Gaussian set is managed in GPU buffers. Each Gaussian $i$ stores:
- Mean position $\boldsymbol{\mu}_i$ (3D vector)
- Covariance $\boldsymbol{\Sigma}_i$ (derived from scale $\mathbf{s}_i$ and rotation $\mathbf{q}_i$)
- Opacity $\alpha_i$ (scalar)
- Spherical Harmonics coefficients (for color/view-dependence)
- Feature vector (if applicable)

**Algorithm Steps:**
1.  **Evaluate Current State:** Compute the rendering loss $\mathcal{L}(\boldsymbol{\mu}_{\text{old}})$ for the current Gaussian configuration.
2.  **Proposal Generation:** Generate a candidate new Gaussian position $\boldsymbol{\mu}_{\text{new}}$. This is typically sampled from a proposal distribution $q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})$, often centered around existing Gaussians or random locations in high-error regions [S1, unofficial].
3.  **Metropolis–Hastings Acceptance:** Calculate the acceptance probability $\alpha_{\text{spawn}}$ for spawning the new Gaussian:
    $$
    \alpha_{\text{spawn}} = \min \left( 1, \frac{\mathcal{L}(\boldsymbol{\mu}_{\text{new}})}{\mathcal{L}(\boldsymbol{\mu}_{\text{old}})} \cdot \frac{q(\boldsymbol{\mu}_{\text{old}}|\boldsymbol{\mu}_{\text{new}})}{q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})} \right)
    $$
    where $\mathcal{L}(\cdot)$ is the rendering loss at the specified position, and $q$ is the proposal distribution [S1, unofficial].
4.  **Spawn or Reject:** If a random number $r \in [0, 1]$ is less than $\alpha_{\text{spawn}}$, spawn the new Gaussian at $\boldsymbol{\mu}_{\text{new}}$ with initial low opacity (e.g., logit -1.386) and scale. Otherwise, reject the proposal [S1, unofficial].
5.  **Pruning:** Prune Gaussians that have low expected contribution to the equilibrium distribution. This is determined by the Markov chain equilibrium, effectively removing primitives that do not significantly reduce the loss or contribute to the probability mass [S1, unofficial].
6.  **Buffer Management:** Compact the Gaussian buffer to remove pruned primitives and allocate space for new ones, ensuring capacity constraints are met [Parent].

**Parameters:**
- Proposal distribution $q$: Not explicitly defined in sources, but implied to be related to spatial proximity or error regions [S1, unofficial].
- Initial opacity for spawned Gaussians: Low value (consistent with standard 3DGS, e.g., logit -1.386) [Parent].

**Edge Cases and Pitfalls:**
- **Precision:** Computing $\mathcal{L}(\boldsymbol{\mu}_{\text{new}})$ requires a full or partial rasterization pass, which can be computationally expensive if done naively. [G] Efficient approximation or batched evaluation is necessary.
- **Boundaries:** The proposal distribution must cover the scene adequately to avoid missing regions. [G] Uniform sampling or error-guided sampling is recommended.
- **Degenerate Input:** If $\mathcal{L}(\boldsymbol{\mu}_{\text{old}})$ is zero or near-zero, the ratio may become unstable. [G] Add a small epsilon to the denominator.

## Subtopics

### Optimization and Training Strategy
This variant, described in [S2], adopts the densification strategy, loss function, and regularization terms of 3DGS-MCMC [S2]. It introduces specific training optimizations:
- **Learning Rate Scheduling:** Uses a cosine annealing scheduler for feature and MLP learning rates [S2].
- **EMA Filter:** Applies an Exponential Moving Average (EMA) filter to MLP weights $\theta$:
  $$
  \bar{\theta}_t \leftarrow \gamma \bar{\theta}_{t-1} + (1 - \gamma) \theta_t
  $$
  where $\bar{\theta}_t$ is the filtered weight at step $t$ and $\gamma$ is the decay factor [S2].
- **Refinement Phase:** Dedicates the last 3000 training iterations to optimize only feature and MLP weights, disabling regularization terms and freezing all other parameters [S2].
- **Regularization Terms:**
  - Opacity regularization: $\mathcal{R}_\alpha = \frac{1}{P} \sum_{i=1}^P \alpha_i$ [S2].
  - Scale regularization: $\mathcal{R}_s = \frac{1}{P} \sum_{i=1}^P \|\mathbf{s}_i\|_1$ [S2].
  - Total loss: $\mathcal{L} = (1 - \lambda) \mathcal{L}_{L_1} + \lambda \mathcal{L}_{\text{D-SSIM}} + \lambda_\alpha \mathcal{R}_\alpha + \lambda_s \mathcal{R}_s$ [S2].

## Operating characteristics
- **Time Complexity:** Higher per-step cost than gradient-based methods due to the need to evaluate $\mathcal{L}(\boldsymbol{\mu}_{\text{new}})$ for proposals. However, it may reduce the total number of Gaussians required, potentially lowering rendering time [S1, unofficial].
- **Memory Complexity:** Similar to standard 3DGS, but with potentially fewer primitives due to more efficient densification [S1, unofficial].
- **Performance:** Yields more efficient and artifact-free clouds by exploring regions with unknown gradients [S1, unofficial].
- **Strengths:** Reduces artifacts caused by noisy gradients; more stable densification process.
- **Limitations:** Computationally more intensive during the densification step; requires careful tuning of the proposal distribution.
- **When to Use:** When standard gradient-based densification produces artifacts or unstable training; for scenes requiring high fidelity with fewer primitives.
- **When Not to Use:** When computational resources are extremely limited and real-time training speed is critical.

## Background
3DGS-MCMC was introduced as a refinement to the original 3D Gaussian Splatting densification heuristic, which relied on noisy positional gradient magnitudes [S1, unofficial]. It reinterprets the Gaussian cloud as samples from an underlying probability distribution, using MCMC sampling to guide the life cycle of primitives [S1, unofficial]. This approach aims to enforce stability and extreme compactness in the reconstructed scene [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
