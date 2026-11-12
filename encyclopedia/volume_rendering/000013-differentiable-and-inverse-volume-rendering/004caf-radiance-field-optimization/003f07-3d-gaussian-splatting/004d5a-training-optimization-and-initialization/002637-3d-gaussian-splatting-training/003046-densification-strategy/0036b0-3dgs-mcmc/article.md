# 3DGS-MCMC

Also called: MCMC-based Gaussian Lifecycle, Probabilistic Densification

## Overview
3DGS-MCMC replaces the noisy positional gradient magnitude heuristic used in standard 3D Gaussian Splatting with a Markov Chain Monte Carlo (MCMC) framework to guide the spawning and pruning of Gaussian primitives [S1, unofficial]. By treating the Gaussian cloud as samples from an underlying probability distribution, it explores regions with unknown gradients more effectively, yielding denser, more efficient, and artifact-free scene representations [S1, unofficial].

## How it works
**Inputs:**
- Current set of Gaussians $\mathcal{G}$, each with mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and spherical harmonics coefficients.
- Rendering loss $\mathcal{L}$ computed for the current frame or batch.
- Proposal distribution $q(\cdot|\cdot)$ for generating candidate new Gaussian positions.

**Outputs:**
- Updated set of Gaussians $\mathcal{G}'$ with modified population based on MCMC acceptance/rejection.

**Data Structures:**
- Gaussian population $\mathcal{G}$: List or array of Gaussian objects. Each object contains fields for mean $\boldsymbol{\mu} \in \mathbb{R}^3$, covariance $\boldsymbol{\Sigma} \in \mathbb{R}^{3 \times 3}$ (or rotation/scale parameters), opacity $\alpha \in [0, 1]$, and feature vectors.
- Loss buffer: Stores $\mathcal{L}(\boldsymbol{\mu}_k)$ for each Gaussian $k$ or per-pixel loss contributions.

**Method Steps:**
1. **Evaluate Current State:** Compute the rendering loss $\mathcal{L}(\boldsymbol{\mu}_{\text{old}})$ associated with the current Gaussian configuration or specific candidate positions.
2. **Proposal Generation:** For each Gaussian or at periodic intervals, generate a candidate new Gaussian position $\boldsymbol{\mu}_{\text{new}}$ using a proposal distribution $q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})$. This distribution typically centers around the current mean or explores nearby regions.
3. **Acceptance Probability Calculation:** Calculate the Metropolis-Hastings acceptance probability $\alpha_{\text{spawn}}$ for spawning a new Gaussian at $\boldsymbol{\mu}_{\text{new}}$ based on the ratio of losses and proposal probabilities [S1, unofficial]:
   $$
   \alpha_{\text{spawn}} = \min \left( 1, \frac{\mathcal{L}(\boldsymbol{\mu}_{\text{new}})}{\mathcal{L}(\boldsymbol{\mu}_{\text{old}})} \cdot \frac{q(\boldsymbol{\mu}_{\text{old}}|\boldsymbol{\mu}_{\text{new}})}{q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})} \right)
   $$
   Here, $\mathcal{L}(\boldsymbol{\mu})$ represents the loss associated with the Gaussian at position $\boldsymbol{\mu}$. The term $\frac{q(\boldsymbol{\mu}_{\text{old}}|\boldsymbol{\mu}_{\text{new}})}{q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})}$ accounts for asymmetry in the proposal distribution.
4. **Spawn Decision:** Draw a random number $r \sim \text{Uniform}(0, 1)$. If $r < \alpha_{\text{spawn}}$, accept the proposal and add the new Gaussian to the population $\mathcal{G}$. Otherwise, reject it.
5. **Pruning:** Prune Gaussians with low expected contribution to the rendering, determined by the equilibrium distribution of the Markov chain. Gaussians that consistently fail to reduce loss or have negligible impact are removed [S1, unofficial].
6. **Update State:** Replace the old Gaussian with the new one if accepted, or keep the old one if rejected. Update the loss buffer.

**Parameters:**
- Proposal distribution $q$: Not specified in sources. Typically a Gaussian centered at $\boldsymbol{\mu}_{\text{old}}$ with a scale related to the current covariance [G].
- Loss function $\mathcal{L}$: Standard 3DGS loss (L1 + SSIM) [G].

**Edge Cases and Pitfalls:**
- **Loss Evaluation Cost:** Computing $\mathcal{L}(\boldsymbol{\mu}_{\text{new}})$ for every proposal can be expensive. Approximations or batched evaluations may be necessary [G].
- **Proposal Distribution Design:** Poor choice of $q$ can lead to slow convergence or getting stuck in local minima. Symmetric proposals simplify the acceptance ratio [G].
- **Precision:** Numerical stability in loss ratio calculation, especially when losses are close to zero.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def mcmc_densification_step(gaussians, loss_function, proposal_distribution, rng):
    """
    Perform one MCMC-based densification step.
    
    Args:
        gaussians: List of Gaussian objects with attributes .mu, .cov, .alpha, etc.
        loss_function: Function that computes loss given a Gaussian or set of Gaussians.
        proposal_distribution: Function that samples new mu given old mu.
        rng: Random number generator.
        
    Returns:
        Updated list of Gaussians.
    """
    new_gaussians = []
    
    for g in gaussians:
        mu_old = g.mu
        # Compute loss for current Gaussian (or context)
        # Note: In practice, loss might be per-pixel or global. 
        # Simplified: assume loss_function returns scalar for a single Gaussian's contribution.
        loss_old = loss_function(g)
        
        # Generate proposal
        mu_new = proposal_distribution(mu_old, rng)
        
        # Compute loss for proposed position
        # Create a temporary Gaussian for loss evaluation
        g_proposed = Gaussian(mu=mu_new, cov=g.cov, alpha=g.alpha)
        loss_new = loss_function(g_proposed)
        
        # Calculate proposal probabilities
        # Assuming symmetric proposal for simplicity if not specified
        # q(mu_old | mu_new) / q(mu_new | mu_old) = 1 if symmetric
        # If asymmetric, compute these values.
        q_old_given_new = proposal_distribution(mu_new, mu_old, rng) # Placeholder for actual prob density
        q_new_given_old = proposal_distribution(mu_old, mu_new, rng) # Placeholder for actual prob density
        
        # Handle potential division by zero or undefined densities
        if q_new_given_old == 0:
            acceptance_prob = 0.0
        else:
            ratio = (loss_new / loss_old) * (q_old_given_new / q_new_given_old)
            acceptance_prob = min(1.0, ratio)
            
        # Accept or reject
        if rng.random() < acceptance_prob:
            # Accept: Add new Gaussian
            g_new = Gaussian(mu=mu_new, cov=g.cov, alpha=g.alpha)
            new_gaussians.append(g_new)
        else:
            # Reject: Keep old Gaussian
            new_gaussians.append(g)
            
    # Pruning step: Remove Gaussians with low expected contribution
    # This is based on Markov chain equilibrium, not explicitly defined in sources.
    # Heuristic: Remove if loss contribution is below a threshold or opacity is low.
    pruned_gaussians = []
    for g in new_gaussians:
        if g.alpha > 0.01: # Example threshold
            pruned_gaussians.append(g)
            
    return pruned_gaussians
```

## Subtopics

### [Optimization and Training Strategy](9157-optimization-and-training-strategy.md)
Defines the optimization schedule, loss formulation, and regularization techniques for training radiance fields using 3D Gaussian Splatting with MCMC-based densification. It addresses overfitting to noise and geometric artifacts by incorporating MCMC-derived regularization terms on opacity and scale, using cosine annealing for feature/MLP learning rates, applying Exponential Moving Average (EMA) to MLP weights, and executing a final refinement phase that freezes geometry.

## Operating characteristics
- **Time Complexity:** Higher per-step cost than gradient-based densification due to loss evaluation for proposals. However, potentially fewer total steps needed for convergence in complex scenes.
- **Memory Complexity:** Similar to standard 3DGS, but may require storing additional loss values for proposals.
- **Strengths:** More robust to noisy gradients, better exploration of unknown regions, artifact-free clouds [S1, unofficial].
- **Limitations:** Computational overhead of loss evaluation for proposals, sensitivity to proposal distribution design.
- **When to use:** Scenes with complex geometry or high-frequency details where gradient-based methods struggle.
- **When not to use:** Real-time applications requiring minimal overhead, or simple scenes where standard densification suffices.

## Background
3DGS-MCMC was introduced as a refinement to the original 3D Gaussian Splatting densification heuristic, which relied on noisy positional gradient magnitudes [S1, unofficial]. It reinterprets the Gaussian cloud as samples from an underlying probability distribution and uses MCMC sampling to guide the life cycle of primitives [S1, unofficial]. This approach aims to provide a more principled and stable method for densification and pruning.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
