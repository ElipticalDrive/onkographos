# 3DGS-MCMC

Also called: MCMC-based Gaussian Lifecycle, Probabilistic Densification

## Overview
3DGS-MCMC replaces the noisy positional gradient magnitude heuristics of standard 3D Gaussian Splatting with a Markov Chain Monte Carlo (MCMC) framework to guide the spawning and pruning of Gaussian primitives [S1, unofficial]. It reinterprets the Gaussian cloud as samples from an underlying probability distribution, using Metropolis–Hastings sampling to explore regions with unknown gradients, thereby yielding more efficient and artifact-free reconstructions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Current set of Gaussians $\mathcal{G}$, current rendering loss $\mathcal{L}$, proposal distribution $q$.
*   **Output:** Updated set of Gaussians $\mathcal{G}'$ after one MCMC step (spawn/prune decision).

**Data Structures:**
*   **Gaussian Set:** A collection of Gaussian primitives, each defined by mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and spherical harmonics coefficients.
*   **Loss Buffer:** Stores the rendering error $\mathcal{L}$ for the current configuration.

**Algorithm Steps:**
1.  **Proposal:** Generate a candidate new Gaussian $\boldsymbol{\mu}_{\text{new}}$ using a proposal distribution $q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})$. The $\boldsymbol{\mu}_{\text{old}}$ is typically the mean of an existing Gaussian or a random position in the scene volume.
2.  **Evaluation:** Compute the rendering loss $\mathcal{L}(\boldsymbol{\mu}_{\text{new}})$ if the new Gaussian were added, and identify the loss $\mathcal{L}(\boldsymbol{\mu}_{\text{old}})$ associated with the source Gaussian.
3.  **Acceptance Probability:** Calculate the Metropolis–Hastings acceptance probability $\alpha_{\text{spawn}}$ [S1, unofficial]:
    $$
    \alpha_{\text{spawn}} = \min \left( 1, \frac{\mathcal{L}(\boldsymbol{\mu}_{\text{new}})}{\mathcal{L}(\boldsymbol{\mu}_{\text{old}})} \cdot \frac{q(\boldsymbol{\mu}_{\text{old}}|\boldsymbol{\mu}_{\text{new}})}{q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})} \right)
    $$
    where $\mathcal{L}(\cdot)$ represents the rendering error (e.g., L1 or SSIM-based loss) at the respective position [S1, unofficial].
4.  **Spawn Decision:** Draw a uniform random number $r \in [0, 1]$. If $r < \alpha_{\text{spawn}}$, spawn the new Gaussian $\boldsymbol{\mu}_{\text{new}}$ into the cloud.
5.  **Pruning:** Prune Gaussians that have low expected contribution to the final distribution, determined by the equilibrium state of the Markov chain [S1, unofficial]. This removes redundant or low-significance primitives that do not reduce the overall loss.

**Parameters:**
*   **Proposal Distribution $q$:** Not specified in the sources. Typically a Gaussian centered on an existing mean or uniform over the scene bounds [G].
*   **Loss Function $\mathcal{L}$:** Standard 3DGS loss (L1 + D-SSIM) [G].

**Edge Cases and Pitfalls:**
*   **Loss Stability:** If $\mathcal{L}(\boldsymbol{\mu}_{\text{old}})$ is near zero, the ratio may become unstable. A small epsilon should be added to the denominator [G].
*   **Proposal Bias:** The choice of $q$ affects convergence. Uniform proposals may lead to slow exploration, while local proposals may get stuck in local minima [G].

## Subtopics

### [Optimization and Training Strategy](9157-optimization-and-training-strategy.md)
Defines the optimization schedule, loss formulation, and regularization techniques for training radiance fields using 3DGS-MCMC. It distinguishes itself by incorporating MCMC-derived regularization terms on opacity and scale, using cosine annealing for feature/MLP learning rates, applying Exponential Moving Average (EMA) to MLP weights, and executing a final refinement phase that freezes geometry.

## Operating characteristics
*   **Time Complexity:** Higher per-step overhead than gradient-based methods due to the need to evaluate rendering loss for proposals and perform MCMC sampling [S1, unofficial].
*   **Memory Complexity:** Similar to standard 3DGS, as it manages a set of Gaussian primitives [S1, unofficial].
*   **Strengths:** Explores regions with unknown gradients more effectively, reducing artifacts and producing more efficient Gaussian clouds [S1, unofficial].
*   **Limitations:** Computationally more expensive per iteration; convergence speed depends heavily on the proposal distribution [S1, unofficial].
*   **When to Use:** When high-quality, artifact-free reconstruction is prioritized over training speed, or when gradient-based methods fail to converge in complex scenes [S1, unofficial].

## Background
3DGS-MCMC was introduced as a refinement to the original 3D Gaussian Splatting densification heuristic, which relied on noisy positional gradients [S1, unofficial]. It reinterprets the Gaussian cloud as samples from an underlying probability distribution, applying MCMC sampling to guide the life cycle of primitives [S1, unofficial]. This approach addresses the problem of overfitting to noise and geometric artifacts during iterative refinement [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
