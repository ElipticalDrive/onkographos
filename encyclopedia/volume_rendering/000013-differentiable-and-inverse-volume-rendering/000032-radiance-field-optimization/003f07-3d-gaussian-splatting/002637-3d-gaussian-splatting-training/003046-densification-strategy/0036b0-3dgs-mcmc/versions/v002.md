# 3DGS-MCMC

Also called: MCMC-based Gaussian Lifecycle

## Overview
3DGS-MCMC replaces the noisy positional gradient magnitude heuristics of standard 3D Gaussian Splatting with a probabilistic framework that treats the Gaussian cloud as samples from an underlying probability distribution [S1, unofficial]. It uses Markov Chain Monte Carlo (MCMC) sampling to guide the life cycle (spawn/prune) of primitives, allowing the method to explore regions with unknown gradients and yielding more efficient, artifact-free clouds [S1, unofficial].

## How it works
**Inputs:** Current set of Gaussians $\mathcal{G}$, rendering loss $\mathcal{L}$, proposal distribution $q$.
**Outputs:** Updated set of Gaussians $\mathcal{G}'$ after MCMC steps.

**Data Structures:**
The Gaussian set is managed as a list of primitives. Each primitive $i$ has parameters: mean $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$ (or scale/rotation), opacity $\alpha_i$, and spherical harmonics coefficients.

**Algorithm Steps:**
1.  **Evaluate Current State:** Compute the rendering loss $\mathcal{L}(\boldsymbol{\mu}_{\text{old}})$ for the current Gaussian configuration.
2.  **Proposal:** Generate a candidate new Gaussian position $\boldsymbol{\mu}_{\text{new}}$. The proposal distribution $q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})$ defines how candidates are generated (e.g., random perturbation or sampling from error map).
3.  **Evaluate Candidate:** Compute the rendering loss $\mathcal{L}(\boldsymbol{\mu}_{\text{new}})$ if the new Gaussian were added or if the position were moved.
4.  **Metropolis-Hastings Acceptance:** Calculate the acceptance probability $\alpha_{\text{spawn}}$ for spawning a new Gaussian or accepting a move:
    $$ \alpha_{\text{spawn}} = \min \left( 1, \frac{\mathcal{L}(\boldsymbol{\mu}_{\text{new}})}{\mathcal{L}(\boldsymbol{\mu}_{\text{old}})} \cdot \frac{q(\boldsymbol{\mu}_{\text{old}}|\boldsymbol{\mu}_{\text{new}})}{q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})} \right) $$
    [S1, unofficial]
    *   If $\alpha_{\text{spawn}}$ is accepted, spawn the new Gaussian at $\boldsymbol{\mu}_{\text{new}}$ (or accept the move).
    *   If rejected, retain the old state.
5.  **Pruning:** Prune Gaussians with low expected contribution according to the Markov chain equilibrium. This removes primitives that do not significantly reduce the loss or contribute to the probability distribution [S1, unofficial].

**Parameters:**
*   Proposal distribution $q$: Not explicitly defined in the source, but typically involves sampling near high-error regions or random perturbations.
*   Loss function $\mathcal{L}$: Standard 3DGS loss (L1 + D-SSIM) [S2].

**Edge Cases:**
*   **Zero Loss:** If $\mathcal{L}(\boldsymbol{\mu}_{\text{old}}) = 0$, the ratio is undefined. In practice, a small epsilon or clamping is required [G].
*   **Proposal Bias:** If $q$ is not symmetric, the ratio term $\frac{q(\boldsymbol{\mu}_{\text{old}}|\boldsymbol{\mu}_{\text{new}})}{q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})}$ is critical for detailed balance [S1, unofficial].

## Subtopics

### Optimization and Training Strategy
Uses the densification strategy, loss function, and regularization terms of 3DGS-MCMC [S2]. Applies a cosine annealing scheduler for feature and MLP learning rates [S2]. Uses an Exponential Moving Average (EMA) filter on MLP weights $\theta$: $\bar{\theta}_t \leftarrow \gamma \bar{\theta}_{t-1} + (1 - \gamma) \theta_t$, where $\bar{\theta}_t$ is the filtered weight at step $t$ and $\gamma$ is the decay factor [S2]. Dedicates the last 3000 training iterations to optimize only feature and MLP weights, disabling regularization terms and freezing all other parameters [S2].

## Operating characteristics
*   **Complexity:** MCMC sampling adds overhead per step compared to deterministic gradient-based densification, as it requires evaluating loss for proposed states.
*   **Performance:** Yields more efficient and artifact-free clouds by exploring regions with unknown gradients [S1, unofficial].
*   **Strengths:** Reduces reliance on noisy positional gradients; better handles regions with sparse gradient information.
*   **Limitations:** Computational cost of MCMC steps; requires careful tuning of proposal distribution.

## Background
Originated from the need to refine the densification dynamics of 3D Gaussian Splatting to enforce stability and extreme compactness [S1, unofficial]. Reinterprets the Gaussian cloud as samples from an underlying probability distribution [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
