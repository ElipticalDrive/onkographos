# 3DGS-MCMC

Also called: MCMC-based Gaussian Lifecycle

## Overview
3DGS-MCMC replaces the noisy positional gradient magnitude heuristics of standard 3D Gaussian Splatting with a probabilistic framework that treats the Gaussian cloud as samples from an underlying distribution [S1, unofficial]. It uses Markov Chain Monte Carlo (MCMC) sampling to guide the spawning and pruning of primitives, allowing the method to explore regions with unknown gradients and produce more efficient, artifact-free clouds [S1, unofficial].

## How it works
**Inputs:** Current set of Gaussians $\mathcal{G}$, rendering loss $\mathcal{L}$, proposal distribution $q$.
**Outputs:** Updated set of Gaussians $\mathcal{G}'$.

**Data Structures:**
*   $\mathcal{G}$: List of Gaussian primitives, each containing mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and spherical harmonics coefficients.
*   $\mathcal{L}(\boldsymbol{\mu})$: Scalar rendering loss associated with a specific mean position.

**Parameters:**
*   $q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})$: Proposal distribution for generating a candidate Gaussian position. Not given in the sources.
*   $\alpha_{\text{spawn}}$: Acceptance probability for spawning a new Gaussian.

**Steps:**
1.  **Metropolis-Hastings Spawn:** When densification is triggered, a candidate Gaussian is proposed at position $\boldsymbol{\mu}_{\text{new}}$. The probability of accepting this spawn is determined by the ratio of rendering losses and the proposal distribution ratio [S1, unofficial]:
    $$
    \alpha_{\text{spawn}} = \min \left( 1, \frac{\mathcal{L}(\boldsymbol{\mu}_{\text{new}})}{\mathcal{L}(\boldsymbol{\mu}_{\text{old}})} \cdot \frac{q(\boldsymbol{\mu}_{\text{old}}|\boldsymbol{\mu}_{\text{new}})}{q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})} \right)
    $$
    If accepted, the new Gaussian is added to $\mathcal{G}$ [S1, unofficial].
2.  **Equilibrium Pruning:** Gaussians with low expected contribution are pruned according to the Markov chain equilibrium [S1, unofficial]. The specific threshold or metric for "low expected contribution" is not defined in the sources [G].
3.  **Buffer Management:** Standard densification strategy buffer compaction is applied to maintain contiguous memory access [Parent].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mcmc_densify_step(Gaussians, LossFunction, ProposalDist):
    """
    Performs one MCMC-based densification step.
    """
    new_gaussians = []
    for g in Gaussians:
        # 1. Propose a new Gaussian position
        mu_new = ProposalDist.sample(g.mu)
        
        # 2. Calculate loss for the new position
        # Note: LossFunction must return scalar loss for a given mean
        L_new = LossFunction(mu_new)
        L_old = LossFunction(g.mu)
        
        # 3. Calculate proposal ratio
        # q(old|new) / q(new|old)
        # Implementation depends on ProposalDist definition [G]
        proposal_ratio = ProposalDist.prob(g.mu, mu_new) / ProposalDist.prob(mu_new, g.mu)
        
        # 4. Calculate acceptance probability
        alpha_spawn = min(1.0, (L_new / L_old) * proposal_ratio)
        
        # 5. Accept or reject spawn
        if random() < alpha_spawn:
            # Spawn new Gaussian (details of covariance/opacity initialization not given [G])
            new_g = Gaussian(mu=mu_new, ...)
            new_gaussians.append(new_g)
            
        # 6. Pruning logic based on equilibrium contribution
        # Specific metric for "low expected contribution" not given [G]
        if is_low_contribution(g):
            continue # Prune
        else:
            new_gaussians.append(g)
            
    return new_gaussians
```

## Subtopics
## Operating characteristics
**Strengths:** Explores regions with unknown gradients better than gradient-based heuristics, yielding more efficient and artifact-free clouds [S1, unofficial].
**Limitations:** Requires evaluation of rendering loss for proposal positions, which may be computationally expensive compared to simple gradient magnitude checks. Not given in the sources.
**Complexity:** Not given in the sources.

## Background
3DGS-MCMC was introduced to address the instability and noise inherent in the original 3D Gaussian Splatting densification heuristic [S1, unofficial]. It reinterprets the Gaussian cloud as samples from an underlying probability distribution [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
