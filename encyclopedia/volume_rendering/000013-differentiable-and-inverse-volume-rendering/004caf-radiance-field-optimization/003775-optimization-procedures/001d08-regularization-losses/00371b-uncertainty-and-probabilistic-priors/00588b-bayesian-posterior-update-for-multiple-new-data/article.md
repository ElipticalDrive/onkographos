# Bayesian Posterior Update for Multiple New Data
Also called: ActiveNeRF posterior distribution derivation, Gaussian posterior update for radiance fields

## Overview
This method computes the posterior distribution of a scene point's color given a Gaussian prior and multiple new observed data points, enabling uncertainty-aware view selection in active reconstruction [S1]. It distinguishes itself from sibling methods like [Uncertainty Prediction Loss](2961-uncertainty-prediction-loss.md) by explicitly modeling the affine transformation of observed colors and analytically deriving the updated mean and variance for Bayesian inference, rather than relying on differentiable rendering pipelines to predict uncertainty [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - Prior mean $\bar{c}(\text{x})$ and prior variance $\beta_0^2$ for a scene point $\text{x}$ [S1].
  - $K$ new observations $\text{r}_1, \dots, \text{r}_K$ [S1].
  - For each observation $i$: observed color $C(\text{r}_i)$, affine scale $\alpha_i$, affine bias $b_i$, and observation noise variance $\beta_i^2$ [S1].
- **Outputs:**
  - Posterior mean $\mu_{\text{post}}$ and posterior variance $\sigma_{\text{post}}^2$ [S1].

**Data Structures:**
- Scalars for all parameters ($\bar{c}, \beta_0, \alpha, b, \beta, C$) [S1].
- Arrays for weights $\gamma$ if $K > 2$ [S1].

**Numbered Steps:**
1. Define the prior distribution $P^{(\text{pri})}$ as a Gaussian $\mathcal{N}(\bar{c}(\text{x}), \beta_0^2)$ [S1].
2. Define the likelihood for each observation $i$ as a Gaussian $p(\text{r}_i|\text{x}) \sim \mathcal{N}(\alpha_i \text{x}+b_i, \beta_i^2)$ [S1].
3. Compute the posterior distribution $P^{(\text{post})}$ by combining the prior and all likelihoods [S1].
4. Calculate the posterior weights $\gamma_i$ for $i=1 \dots K$ and $\gamma_{K+1}$ for the prior term [S1].
5. Compute the posterior mean as the weighted sum of the inverse-affine observed colors and the prior mean [S1].
6. Compute the posterior variance as the weight of the prior term multiplied by the prior variance [S1].

**Formulas:**
For $K=2$ observations, the posterior is:
$$
P^{(\text{post})} \sim \mathcal{N}\left( \gamma_1 \frac{C(\text{r}_1)-b_1}{\alpha_1} + \gamma_2 \frac{C(\text{r}_2)-b_2}{\alpha_2} + \gamma_3 \bar{c}(\text{x}), \gamma_3 \beta_0^2 \right)
$$
where the weights are:
$$
\gamma_1 = \frac{\alpha_1^2\beta_0^2\beta_2^2}{\alpha_1^2\beta_0^2\beta_2^2 + \alpha_2^2\beta_0^2\beta_1^2 + \beta_1^2\beta_2^2}
$$
$$
\gamma_2 = \frac{\alpha_2^2\beta_0^2\beta_1^2}{\alpha_1^2\beta_0^2\beta_2^2 + \alpha_2^2\beta_0^2\beta_1^2 + \beta_1^2\beta_2^2}
$$
$$
\gamma_3 = \frac{\beta_1^2\beta_2^2}{\alpha_1^2\beta_0^2\beta_2^2 + \alpha_2^2\beta_0^2\beta_1^2 + \beta_1^2\beta_2^2}
$$
[S1]

For general $K$, the derivation extends by analogous substitutions [S1]. The posterior mean is a weighted sum of $\frac{C(\text{r}_i)-b_i}{\alpha_i}$ and $\bar{c}(\text{x})$, and the posterior variance is determined by the weight associated with the prior term [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_posterior(k, C, alpha, b, beta_sq, c_prior, beta0_sq):
    """
    Compute posterior mean and variance for K observations.
    C: list of observed colors [C(r_1), ..., C(r_K)]
    alpha: list of affine scales [alpha_1, ..., alpha_K]
    b: list of affine biases [b_1, ..., b_K]
    beta_sq: list of observation variances [beta_1^2, ..., beta_K^2]
    c_prior: prior mean c(x)
    beta0_sq: prior variance beta_0^2
    """
    # Source [S1] gives explicit formulas for K=2; generalization to K observations 
    # follows standard Bayesian precision weighting [G].
    
    # Calculate product of all observation variances
    prod_beta_sq = 1.0
    for bs in beta_sq:
        prod_beta_sq *= bs
        
    # Calculate denominator: product of all beta_i^2 + sum over i of (alpha_i^2 * beta0^2 * product_{j!=i} beta_j^2)
    denom = prod_beta_sq
    for i in range(k):
        prod_beta_sq_excl_i = prod_beta_sq / beta_sq[i]
        denom += (alpha[i]**2) * beta0_sq * prod_beta_sq_excl_i
        
    # Calculate prior weight (gamma_3 for K=2)
    gamma_prior = prod_beta_sq / denom
    
    # Calculate posterior mean
    mean = gamma_prior * c_prior
    for i in range(k):
        obs_val = (C[i] - b[i]) / alpha[i]
        prod_beta_sq_excl_i = prod_beta_sq / beta_sq[i]
        gamma_i = (alpha[i]**2) * beta0_sq * prod_beta_sq_excl_i / denom
        mean += gamma_i * obs_val
        
    # Calculate posterior variance
    var = gamma_prior * beta0_sq
    
    return mean, var
```

**Parameters:**
- $\alpha_i, b_i$: Affine parameters mapping scene color to observed color. Not given in the sources.
- $\beta_i^2$: Observation noise variance. Not given in the sources.
- $\beta_0^2$: Prior variance. Not given in the sources.

**Edge Cases and Pitfalls:**
- Division by zero if $\alpha_i = 0$ [G].
- Numerical instability if variances are very small or very large [G].
- The derivation assumes Gaussian distributions for prior and likelihood [S1].

## Subtopics

### [Uncertainty Guided Policy with Region Clustering](17856-uncertainty-guided-policy-with-region-clustering.md)
This method is an active view selection policy for robotic 3D reconstruction using Neural Radiance Fields (NeRF). It selects the next best views by maximizing information gain, approximated via the mean entropy of a candidate view's uncertainty map. It distinguishes itself from global greedy selection by enforcing spatial diversity through region clustering, preventing the selection of redundant, similar viewpoints that cause overfitting, while avoiding the slow coverage of single-view-per-iteration strategies.

### [Open-Set Semantic Fusion via Normal-Inverse-Gamma Conjugacy](21630-open-set-semantic-fusion-via-normal-inverse-gamma-conjugacy.md)
This method fuses continuous, open-set semantic feature vectors into volumetric voxels by modeling each feature dimension as a Gaussian distribution with unknown mean and variance. It employs a Normal-Inverse-Gamma conjugate prior to compute a closed-form posterior, enabling uncertainty-aware semantic mapping. Unlike closed-set fusion which uses Dirichlet-Categorical conjugacy for discrete classes, this approach models the full feature space, allowing for the representation of unseen or ambiguous semantic concepts.

## Operating characteristics
- **Time Complexity:** $O(K)$ for computing weights and mean, where $K$ is the number of observations [G].
- **Memory Complexity:** $O(K)$ to store observation parameters [G].
- **Strengths:** Analytical solution allows efficient uncertainty propagation [G].
- **Limitations:** Assumes Gaussianity and affine color transformation [G].
- **When to use:** Active view selection where uncertainty estimation is critical [S1].
- **When not to use:** Non-Gaussian noise or complex color transformations [G].

## Background
Originated in ActiveNeRF: Learning Where to See with Uncertainty Estimation by X. Pan et al. [S1]. The method is used in active 3D reconstruction to guide camera placement based on uncertainty [S1].

## Sources
[S1] ActiveNeRF: Learning Where to See with Uncertainty Estimation, https://link.springer.com/chapter/10.1007/978-3-031-19827-4_14, fetched 2026-10-28
