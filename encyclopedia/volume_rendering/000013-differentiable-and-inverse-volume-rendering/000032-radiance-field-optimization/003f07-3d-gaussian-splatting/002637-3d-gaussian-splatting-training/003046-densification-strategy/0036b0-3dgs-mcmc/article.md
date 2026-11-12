# 3DGS-MCMC
Also called: MCMC-based Gaussian Lifecycle
## Overview
3DGS-MCMC replaces the noisy positional gradient magnitude heuristics of standard Adaptive Density Control with a probabilistic framework that reinterprets the Gaussian cloud as samples from an underlying probability distribution [S1, unofficial]. It uses Markov Chain Monte Carlo (MCMC) sampling to guide the spawning and pruning of primitives, allowing the method to explore regions with unknown gradients and yield more efficient, artifact-free clouds [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** Current set of Gaussians $\mathcal{G}$, current rendering loss $\mathcal{L}$, proposal distribution $q$.
*   **Output:** Updated set of Gaussians $\mathcal{G}'$ after one MCMC lifecycle step.

**Data Structures:**
*   Gaussian $g_k$: Position $\boldsymbol{\mu}_k$, covariance $\boldsymbol{\Sigma}_k$, opacity $\alpha_k$, spherical harmonics coefficients.
*   Loss $\mathcal{L}(\boldsymbol{\mu})$: Scalar rendering error (e.g., L1 or SSIM-based) at a specific position.

**Steps:**
1.  **Proposal:** For each Gaussian or a subset of candidates, propose a new position $\boldsymbol{\mu}_{\text{new}}$. The proposal distribution $q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})$ determines how the new candidate is generated relative to the current state $\boldsymbol{\mu}_{\text{old}}$ [S1, unofficial].
2.  **Evaluation:** Compute the rendering loss $\mathcal{L}(\boldsymbol{\mu}_{\text{new}})$ for the proposed position and compare it to the loss $\mathcal{L}(\boldsymbol{\mu}_{\text{old}})$ of the current Gaussian [S1, unofficial].
3.  **Acceptance Probability:** Calculate the Metropolis-Hastings acceptance probability $\alpha_{\text{spawn}}$ using Equation (11) [S1, unofficial]:
    $$
    \alpha_{\text{spawn}} = \min \left( 1, \frac{\mathcal{L}(\boldsymbol{\mu}_{\text{new}})}{\mathcal{L}(\boldsymbol{\mu}_{\text{old}})} \cdot \frac{q(\boldsymbol{\mu}_{\text{old}}|\boldsymbol{\mu}_{\text{new}})}{q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})} \right)
    $$
    where $\mathcal{L}(\boldsymbol{\mu}_{\text{new}})$ is the loss at the new position, $\mathcal{L}(\boldsymbol{\mu}_{\text{old}})$ is the loss at the old position, and $q$ represents the proposal distribution densities [S1, unofficial].
4.  **Spawn/Prune Decision:**
    *   If a random number $r \in [0, 1]$ is less than $\alpha_{\text{spawn}}$, spawn a new Gaussian at $\boldsymbol{\mu}_{\text{new}}$ [S1, unofficial].
    *   Prune Gaussians with low expected contribution based on the Markov chain equilibrium criteria [S1, unofficial].
5.  **Update:** Add spawned Gaussians to $\mathcal{G}$ and remove pruned ones [S1, unofficial].

**Parameters:**
*   Proposal distribution $q$: Not given in the sources.
*   Loss function $\mathcal{L}$: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Loss values close to zero may cause numerical instability in the ratio. [G] Add a small epsilon to denominators.
*   **Boundaries:** Proposal distributions must handle scene boundaries to avoid invalid positions. [G] Clamp or reflect proposals.
*   **Degenerate Input:** If $\mathcal{L}(\boldsymbol{\mu}_{\text{old}}) = 0$, the ratio is undefined. [G] Handle zero-loss cases by skipping or using a default acceptance probability.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mcmc_step(gaussians, proposal_distribution, loss_function):
    new_gaussians = []
    for g in gaussians:
        mu_old = g.position
        L_old = loss_function(mu_old)
        
        # Propose new position
        mu_new = proposal_distribution.sample(mu_old)
        L_new = loss_function(mu_new)
        
        # Calculate acceptance probability
        q_old_given_new = proposal_distribution.log_prob(mu_old, mu_new)
        q_new_given_old = proposal_distribution.log_prob(mu_new, mu_old)
        
        # Avoid log(0)
        if L_old == 0 or L_new == 0:
            alpha_spawn = 0.0
        else:
            log_ratio = (math.log(L_new) - math.log(L_old) + 
                         q_old_given_new - q_new_given_old)
            alpha_spawn = min(1.0, math.exp(log_ratio))
        
        # Decide spawn/prune
        if random.random() < alpha_spawn:
            new_gaussians.append(Gaussian(position=mu_new, ...))
        else:
            # Prune logic based on equilibrium
            if is_low_contribution(g):
                continue
            else:
                new_gaussians.append(g)
    return new_gaussians
```
## Subtopics
## Operating characteristics
*   **Time Complexity:** Depends on the cost of evaluating the loss function for each proposal. Typically higher per-step cost than gradient-based methods due to loss recomputation.
*   **Memory Complexity:** Similar to standard 3DGS, as it manages a set of Gaussians.
*   **Performance:** Yields more efficient and artifact-free clouds by exploring regions with unknown gradients [S1, unofficial].
*   **Strengths:** Reduces noise from positional gradient heuristics; better exploration of the scene.
*   **Limitations:** Computationally expensive due to loss evaluations; requires careful tuning of the proposal distribution.
*   **When to use:** When gradient-based methods produce noisy or artifact-prone results; when scene exploration is critical.
*   **When not to use:** When computational resources are limited; when fast convergence is prioritized over artifact reduction.
## Background
3DGS-MCMC was introduced as a refinement to the original 3D Gaussian Splatting densification heuristic [S1, unofficial]. It reinterprets the Gaussian cloud as samples from an underlying probability distribution and uses MCMC sampling to guide the life cycle of primitives [S1, unofficial]. This approach addresses the problem of noisy positional gradients by providing a probabilistic framework for densification and pruning [S1, unofficial].
## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
