# 3DGS-MCMC

Also called: MCMC-based Gaussian Lifecycle

## Overview
3DGS-MCMC replaces the noisy positional gradient magnitude heuristics of standard 3D Gaussian Splatting with a Markov Chain Monte Carlo (MCMC) framework to guide the life cycle (spawn and prune) of Gaussian primitives [S1, unofficial][S2]. By treating the Gaussian cloud as samples from an underlying probability distribution, it dynamically adds and removes Gaussians based on rendering error and proposal probabilities, yielding more efficient, artifact-free clouds with sharper details compared to the original densification process [S1, unofficial][S2].

## How it works
**Inputs:**
*   Current set of Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$, where each $g_k$ has mean $\mu_k$, covariance $\Sigma_k$, opacity $\alpha_k$, and spherical harmonic coefficients $\mathbf{c}_k$ [Parent].
*   Current training loss $\mathcal{L}$ computed from rendered images.
*   Proposal distribution $q(\cdot|\cdot)$ for generating candidate Gaussian positions.

**Outputs:**
*   Updated set of Gaussians $\mathcal{G}'$ after MCMC sampling steps.

**Data Structures:**
*   The Gaussian cloud is maintained as a list or array of Gaussian objects.
*   Each Gaussian object stores: $\mu \in \mathbb{R}^3$, $\Sigma \in \mathbb{R}^{3 \times 3}$ (or parameters $s, q$), $\alpha \in [0, 1]$, $\mathbf{c}$ (SH coefficients).
*   Buffer management is required to handle dynamic addition/removal, typically via flag-based marking or compaction [Parent].

**Steps:**
1.  **Evaluate Current State:** Compute the current rendering loss $\mathcal{L}_{\text{old}}$ for the existing Gaussian configuration.
2.  **Propose New Gaussian:** Generate a candidate Gaussian $g_{\text{new}}$ with mean $\mu_{\text{new}}$ sampled from a proposal distribution $q(\mu_{\text{new}}|\mu_{\text{old}})$, where $\mu_{\text{old}}$ is the mean of an existing Gaussian or a scene-wide distribution. The covariance, opacity, and SH coefficients can be initialized based on $\mu_{\text{old}}$ or scene priors.
3.  **Compute Acceptance Probability:** Calculate the Metropolis-Hastings acceptance probability $\alpha_{\text{spawn}}$ for spawning the new Gaussian:
    $$
    \alpha_{\text{spawn}} = \min \left( 1, \frac{\mathcal{L}(\boldsymbol{\mu}_{\text{new}})}{\mathcal{L}(\boldsymbol{\mu}_{\text{old}})} \cdot \frac{q(\boldsymbol{\mu}_{\text{old}}|\boldsymbol{\mu}_{\text{new}})}{q(\boldsymbol{\mu}_{\text{new}}|\boldsymbol{\mu}_{\text{old}})} \right)
    $$
    where $\mathcal{L}(\boldsymbol{\mu})$ is the rendering loss if the Gaussian were centered at $\boldsymbol{\mu}$ (or the loss contribution associated with that position) [S1, unofficial].
4.  **Spawn or Reject:** With probability $\alpha_{\text{spawn}}$, add $g_{\text{new}}$ to the Gaussian cloud. Otherwise, reject the proposal.
5.  **Prune Low-Contribution Gaussians:** Evaluate the expected contribution of each existing Gaussian. Prune Gaussians that fall below a threshold determined by the Markov chain equilibrium, effectively removing those with low expected contribution to the rendering [S1, unofficial].
6.  **Update State:** Recompute the loss and repeat for the next densification interval.

**Parameters:**
*   Proposal distribution $q$: Not given in the sources. Typically involves sampling from a normal distribution centered on existing Gaussians or uniform distribution over the scene extent [Parent, S1].
*   Loss function $\mathcal{L}$: Standard 3DGS loss (e.g., L1 + SSIM) [Parent, S1].
*   Densification interval: Periodic evaluation, e.g., every 100 iterations [Parent].

**Edge Cases and Pitfalls:**
*   **Loss Computation:** Computing $\mathcal{L}(\boldsymbol{\mu}_{\text{new}})$ requires rendering or estimating the loss for the proposed Gaussian, which can be computationally expensive. Approximations or incremental updates may be needed.
*   **Proposal Distribution:** The choice of $q$ affects the exploration efficiency. Poor proposals may lead to slow convergence or sampling in irrelevant regions.
*   **Pruning Threshold:** Determining the equilibrium threshold for pruning is non-trivial and may require adaptive mechanisms.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mcmc_densification_step(Gaussians, LossFunction, ProposalDistribution, CurrentIteration):
    """
    Perform one MCMC-based densification step.
    
    Args:
        Gaussians: List of Gaussian objects.
        LossFunction: Function to compute rendering loss given Gaussians.
        ProposalDistribution: Function to sample new Gaussian means.
        CurrentIteration: Current training iteration number.
        
    Returns:
        Updated list of Gaussians.
    """
    # Periodic densification check (e.g., every 100 iterations)
    if CurrentIteration % 100 != 0:
        return Gaussians

    new_Gaussians = []
    current_loss = LossFunction(Gaussians)
    
    # MCMC Sampling Loop
    # Number of proposals per step is not specified; assume a fixed number or proportional to N
    num_proposals = len(Gaussians) 
    
    for _ in range(num_proposals):
        # Select a parent Gaussian for proposal
        parent_idx = random.randint(0, len(Gaussians) - 1)
        parent_g = Gaussians[parent_idx]
        
        # Propose new Gaussian
        mu_new = ProposalDistribution(parent_g.mu)
        g_new = Gaussian(mu=mu_new, sigma=parent_g.sigma, alpha=parent_g.alpha, c=parent_g.c)
        
        # Compute loss for proposed Gaussian (approximation or full render)
        # Note: Full render for each proposal is expensive. 
        # The source implies using L(mu_new) / L(mu_old). 
        # L(mu_old) is implicitly the current loss contribution or a baseline.
        # For implementation, we might approximate L(mu_new) by rendering with g_new added 
        # or by using a local error metric.
        
        # Simplified: Assume we can compute a local loss metric for the proposal
        # In practice, this might involve a lightweight render or error estimation.
        loss_new = LossFunction(Gaussians + [g_new]) # Approximation
        
        # Compute acceptance probability
        # q(mu_old | mu_new) / q(mu_new | mu_old)
        # If proposal is symmetric (e.g., Gaussian centered on parent), ratio is 1.
        q_ratio = 1.0 # Assumption: Symmetric proposal
        
        alpha_spawn = min(1.0, (loss_new / current_loss) * q_ratio)
        
        if random.random() < alpha_spawn:
            new_Gaussians.append(g_new)
            
    # Pruning step
    # Prune Gaussians with low expected contribution
    # Threshold based on Markov chain equilibrium is not explicitly defined.
    # Heuristic: Remove Gaussians with opacity below a threshold or low gradient magnitude.
    pruned_Gaussians = []
    for g in Gaussians:
        if g.alpha > alpha_threshold: # alpha_threshold not specified
            pruned_Gaussians.append(g)
            
    # Combine new and pruned Gaussians
    final_Gaussians = pruned_Gaussians + new_Gaussians
    
    return final_Gaussians
```

## Subtopics

### [Optimization and Training Strategy](9157-optimization-and-training-strategy.md)
Defines the optimization schedule, loss formulation, and regularization techniques for training radiance fields using 3D Gaussian Splatting with MCMC-based densification. It addresses overfitting by incorporating MCMC-derived regularization terms on opacity and scale, using cosine annealing for feature/MLP learning rates, applying Exponential Moving Average (EMA) to MLP weights, and executing a final refinement phase that freezes geometry.

## Operating characteristics
*   **Time Complexity:** The MCMC step involves proposing and evaluating new Gaussians, which can be more computationally expensive than the simple gradient thresholding in standard 3DGS, especially if full loss evaluation is performed for each proposal. However, it may reduce the total number of iterations needed for convergence by exploring the space more efficiently.
*   **Memory Complexity:** Similar to standard 3DGS, memory usage depends on the number of Gaussians. The dynamic addition/removal requires efficient buffer management.
*   **Performance:** MCMC densification yields sharper details and better performance compared to the original 3DGS densification process, particularly in scenes with high-frequency details, such as underwater scenes [S2]. It produces more efficient and artifact-free clouds by exploring regions with unknown gradients [S1, unofficial].
*   **Strengths:** Reduces noise and artifacts associated with gradient-based heuristics; better handles high-frequency details; probabilistic framework allows for exploration of unknown regions.
*   **Limitations:** Computational overhead of MCMC sampling; sensitivity to proposal distribution and pruning thresholds.

## Background
3DGS-MCMC was introduced as an improvement to the densification process in 3D Gaussian Splatting, replacing the original heuristic approach with an MCMC-based method [S2]. It reinterprets the Gaussian cloud as samples from an underlying probability distribution, using Markov Chain Monte Carlo sampling to guide the life cycle of primitives [S1, unofficial]. This approach was notably adopted in research on underwater volumetric rendering to capture high-frequency details [S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
