# Regularization and Priors


## Overview
Regularization and Priors in NeRF training introduce constraints beyond photometric reconstruction to enforce physical plausibility, structural stability, and uncertainty awareness. This family addresses issues such as scale ambiguity in material decomposition, instability in dynamically growing networks, and overfitting to noisy observations. It is distinct from sibling methods like [Learning Rate Scheduling](2701-learning-rate-scheduling.md) or [Adam Optimisation with Adaptive Scaling](3093-adam-optimisation-with-adaptive-scaling.md) by modifying the loss landscape or network outputs directly rather than the optimization trajectory or parameter updates.

## Shared principles
The core mechanism involves augmenting the primary photometric loss $\mathcal{L}_{photo}$ with auxiliary terms $\mathcal{L}_{reg}$ derived from domain-specific assumptions (e.g., smoothness, intrinsic decomposition, or probabilistic uncertainty). The total training loss is typically a weighted sum:
$$
\mathcal{L}_{total} = \mathcal{L}_{photo} + \sum_k \lambda_k \mathcal{L}_{reg}^{(k)}
$$
where $\lambda_k$ are hyperparameters balancing the influence of each prior. These priors often operate on intermediate representations (such as reflectance/shading fields or uncertainty maps) rather than just the final rendered pixel values.

## Subtopics

### [Multi-scale Joint Training Parameters](2967-multi-scale-joint-training-parameters.md)
This method balances photometric and uncertainty-based losses across multiple stages of a dynamically growing neural radiance field [2967-multi-scale-joint-training-parameters.md]. It addresses the instability of training deep, adaptive networks by jointly optimizing coarse and fine representations, ensuring that uncertainty estimates guide structural growth while maintaining fidelity. It is distinct from static loss weighting by coupling loss coefficients with a multi-stage optimization loop where the number of active stages $D$ increases dynamically.

### [Reflectance and Shading Decomposition Priors](4624-reflectance-and-shading-decomposition-priors.md)
This method enforces physical plausibility by decoupling rendered color into material reflectance $\mathbf{R}$ and shading $\mathbf{S}$ [4624-reflectance-and-shading-decomposition-priors.md]. It solves the problem of unstable material properties in novel view synthesis, particularly for object insertion or complex lighting. It distinguishes itself from standard photometric priors by explicitly regularizing intermediate decomposition components, mitigating scale ambiguity and ensuring consistent brightness levels.

### Weight Annealing
A technique to anneal proposal weights by raising them to a power that increases from 0 to 1 over training steps, encouraging exploration by presenting a wider range of proposal intervals early in training [S1]. The annealed weight $\hat{\mathbf{w}}_n$ at step $n$ (out of $N$ total steps) is computed as:
$$
\hat{\mathbf{w}}_n \propto \hat{\mathbf{w}}^{\frac{bn/N}{(b-1)n/N+1}}
$$
where $b$ is the bias hyperparameter [S1]. In experiments, $b=10$ is used [S1]. Ablating this annealing causes SSIM on the bicycle scene to decrease from 0.687 to 0.679 [S1].

### Laplace Approximation for Neural Networks
This method approximates the posterior distribution of network parameters as a multivariate Gaussian centered at the optimal weights $\omega^*$, using the inverse Hessian of the negative log-posterior as the covariance matrix [S2]. The covariance is derived as $\Sigma = -H(\omega^*)^{-1}$ [S2]. However, directly applying this to NeRF parameters is impracticable due to high correlation between parameters, making accurate covariance estimation challenging, and the expensive sampling overhead [S2].

## Background
Regularization in NeRFs evolved from basic $L_2$ losses to more sophisticated priors addressing specific failure modes. Weight annealing was introduced in mip-NeRF 360 to improve proposal network training stability [S1]. Intrinsic decomposition priors (reflectance/shading) were developed to handle lighting variations and object insertion [4624-reflectance-and-shading-decomposition-priors.md]. Uncertainty-aware losses in multi-scale training address the instability of dynamically growing architectures [2967-multi-scale-joint-training-parameters.md]. Bayesian approaches using Laplace approximation have been explored for uncertainty quantification but face scalability issues [S2].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Bayesian uncertainty analysis for underwater 3D reconstruction with neural radiance fields, https://doi.org/10.1016/j.apm.2024.115806, fetched 2026-10-29
