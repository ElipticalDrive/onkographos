# Optimization Baselines
Also called: Baselines, CMA-ES, Classical Optimization Baselines, Covariance Matrix Adaptation Evolution Strategy

## Overview
This family provides gradient-free optimization strategies for inverse rendering problems where the renderer is treated as a black box, gradients are unavailable or unstable, or the loss landscape is highly non-convex. It serves as a robust alternative to gradient-based methods like those in [NeRF Training Loop](970-nerf-training-loop.md) or [Photometric Optimization](5790-photometric-optimization.md). The defining characteristic of this family is the maintenance of a search distribution over the parameter space, adapting to the local loss landscape without explicit derivative information.

## Shared principles
All methods in this family operate within the standard inverse rendering loop defined by [Optimization Loops and Training Procedures](5622-optimization-loops-and-training-procedures.md), minimizing a photometric loss $\mathcal{L}$ between rendered color $\hat{C}(\mathbf{r})$ and ground-truth $C(\mathbf{r})$ [G]. The core mechanism involves maintaining a search distribution $\mathcal{N}(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ over the parameter space $\mathbb{R}^d$, where $\boldsymbol{\mu}$ is the mean and $\boldsymbol{\Sigma}$ is the anisotropic covariance matrix capturing correlations between parameters [S1, unofficial].

The optimization proceeds in iterations:
1.  **Sampling**: Draw $k$ candidates $\{\mathbf{x}_i\}_{i=0}^{k-1}$ from the current distribution [S1, unofficial].
2.  **Evaluation**: Evaluate the cost $c_i = \mathcal{L}(f(\mathbf{x}_i), y^*)$ for each candidate using the black-box renderer $f$ and target $y^*$ [S1, unofficial].
3.  **Update**: Update $\boldsymbol{\mu}$ and $\boldsymbol{\Sigma}$ by computing a weighted average of the candidates, ranked by their cost. Higher weights are assigned to top-scoring candidates to adapt the search distribution to the local loss landscape [S1, unofficial].

This approach provides robustness against local minima and operates effectively in the presence of strong Monte Carlo noise, which often causes gradient-based differentiable renderers to diverge if the initial estimate is far from the optimum [S1, unofficial].

## Subtopics

### [CMA-ES](6675-optimization-baselines.md)
The primary evolutionary baseline. It relies solely on the covariance matrix adaptation mechanism described in Shared principles. It is chosen for its robustness and ability to adapt to the local loss landscape without explicit gradient information.

### [Feature-Guided Diffusion for Inverse Rendering](7375-feature-guided-diffusion-for-inverse-rendering.md)
Combines CMA-ES with a diffusion-based inverse predictor. It extracts dense visual features from rendered images to guide candidate proposal, avoiding the information bottleneck of scalar loss values and mitigating mode averaging in ill-posed inverse problems. It is chosen when the solution space exhibits multi-modality and gradients are unavailable.

### [Dual Annealing](6675-optimization-baselines.md)
Combines simulated annealing with a local search strategy. It uses a standard SciPy implementation.

### [ZeroGrads](6675-optimization-baselines.md)
Trains a differentiable surrogate of the black-box function’s loss landscape and uses it to derive optimization gradients.

### [DiBO](6675-optimization-baselines.md)
A diffusion-based method that combines a diffusion prior with a learned reward proxy to sample candidates from an approximate posterior over high-performing solutions.

Note: Classical Bayesian Optimization (BO) is not compared directly in the primary literature due to performance overhead that is non-linear with iterations and several orders of magnitude higher than other methods.

## Background
The family originates from the need to optimize non-differentiable or noisy rendering pipelines where backpropagation is infeasible. CMA-ES [Hansen 2006] serves as the classical evolutionary baseline, inheriting the ability to adapt to the local loss landscape via covariance updates [S1, unofficial]. Dual annealing [Tsallis and Stariolo 1996] provides a hybrid approach using simulated annealing [S1, unofficial]. More recent developments include ZeroGrads [Fischer and Ritschel 2024] and DiBO [Yun et al. 2025], which integrate learned surrogates or diffusion priors into the black-box optimization framework [S1, unofficial].

## Sources
[S1] Feature-Guided Diffusion for Non-Differentiable Inverse Rendering, https://doi.org/10.48550/arXiv.2607.17411, fetched 2026-10-29 (unofficial)
