# Optimization Baselines
Also called: Baselines, CMA-ES, Classical Optimization Baselines, Covariance Matrix Adaptation Evolution Strategy

## Overview
This family provides non-gradient-based optimization strategies for inverse volume rendering and radiance-field reconstruction, serving as robust alternatives to gradient-descent methods when the rendering pipeline is non-differentiable, noisy, or computationally prohibitive for backpropagation [S1, unofficial]. Unlike gradient-based siblings such as the [NeRF Training Loop](970-nerf-training-loop.md) or [Photometric Optimization](5790-photometric-optimization.md), these methods rely on population-based search or surrogate modeling to navigate the loss landscape, offering robustness against local minima and Monte Carlo noise [S1, unofficial].

## Shared principles
All methods in this family operate within the standard inverse rendering loop defined by the parent [Optimization Loops and Training Procedures](5622-optimization-loops-and-training-procedures.md), minimizing a photometric loss $\mathcal{L}$ between rendered color $\hat{C}(\mathbf{r})$ and ground-truth $C(\mathbf{r})$ [G]. The core distinction lies in the update mechanism: instead of computing $\nabla_\theta \mathcal{L}$ via automatic differentiation, these methods maintain a search distribution or surrogate model to propose candidate parameters $\theta$ [S1, unofficial].

The general mechanism for population-based baselines (e.g., CMA-ES) involves maintaining a multivariate normal distribution $\mathcal{N}(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ over the parameter space, where $\boldsymbol{\mu}$ is the mean and $\boldsymbol{\Sigma}$ is the anisotropic covariance matrix capturing parameter correlations [S1, unofficial]. Each iteration proceeds as follows:
1.  **Sampling**: Draw $k$ candidates $\{\mathbf{x}_i\}_{i=0}^k \sim \mathcal{N}(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ [S1, unofficial].
2.  **Evaluation**: Evaluate the cost $c_i = \mathcal{L}(\mathbf{x}_i)$ for each candidate using the forward rendering pass [S1, unofficial].
3.  **Ranking**: Sort candidates by cost $c_i$ [S1, unofficial].
4.  **Update**: Compute a weighted average of the ranked candidates to update $\boldsymbol{\mu}$ and $\boldsymbol{\Sigma}$, assigning higher weights to top-scoring candidates to adapt the distribution to the local loss landscape [S1, unofficial].

For surrogate-based baselines (e.g., ZeroGrads, DiBO), the method trains a differentiable approximation of the black-box loss landscape or combines a diffusion prior with a reward proxy to sample from an approximate posterior of high-performing solutions [S1, unofficial]. These methods are chosen when explicit gradients are unavailable, unstable, or too expensive to compute, and when the optimization landscape contains strong non-local effects or significant Monte Carlo noise [S1, unofficial].

## Subtopics

### [Feature-Guided Diffusion for Inverse Rendering](7375-feature-guided-diffusion-for-inverse-rendering.md)
Combines a diffusion-based inverse prediction with CMA-ES optimization. The diffusion model generates high-quality initial candidates, which accelerates CMA-ES convergence, while CMA-ES provides a Gaussian search distribution to train the diffusion denoiser. Choose this when the rendering is non-differentiable and standard evolutionary methods converge too slowly from random initialization.

## Background
The term "Optimization Baselines" in this context refers to a set of classical and modern non-gradient optimization techniques used to evaluate differentiable rendering pipelines. The primary evolutionary baseline, Covariance Matrix Adaptation Evolution Strategy (CMA-ES), was introduced by Hansen [2006] and is valued for its ability to adapt to local loss landscapes without gradient information [S1, unofficial]. Other classical baselines include Dual Annealing, which combines simulated annealing with local search [Tsallis and Stariolo 1996], and is often implemented via standard libraries like SciPy [Virtanen et al. 2020].

Recent developments have introduced learned optimization methods such as ZeroGrads [Fischer and Ritschel 2024], which trains a differentiable surrogate of the loss landscape, and DiBO [Yun et al. 2025], which uses diffusion priors. These methods are compared against classical approaches to demonstrate robustness in the presence of Monte Carlo noise and divergent gradients common in differentiable rendering [S1, unofficial]. Classical Bayesian Optimization (BO) is often excluded from direct comparison due to its non-linear performance overhead and high computational cost relative to other methods [S1, unofficial].

## Sources
[S1] Feature-Guided Diffusion for Non-Differentiable Inverse Rendering, https://doi.org/10.48550/arXiv.2607.17411, fetched 2026-10-29 (unofficial)
