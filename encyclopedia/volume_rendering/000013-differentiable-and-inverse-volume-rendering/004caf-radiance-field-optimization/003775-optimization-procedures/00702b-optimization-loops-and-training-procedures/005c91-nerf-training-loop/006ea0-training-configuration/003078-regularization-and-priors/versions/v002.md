# Regularization and Priors


## Overview
Regularization and Priors in NeRF training apply constraints to the optimization process to enforce physical plausibility, stability, or structural coherence that the photometric reconstruction loss alone cannot guarantee. This family addresses issues such as scale ambiguity in intrinsic decomposition, overfitting to sparse samples, and instability in dynamically growing networks. It is distinct from sibling methods like [Learning Rate Scheduling](2701-learning-rate-scheduling.md) or [Mixed Precision Training](2766-mixed-precision-training.md) because it modifies the loss landscape or network outputs directly via explicit penalty terms or weighted combinations, rather than adjusting the optimizer's state or numerical precision.

## Shared principles
The core mechanism involves augmenting the base photometric reconstruction loss $\mathcal{L}$ with additional regularization terms $\mathcal{L}_{reg}$:
$$
\mathcal{L}_{total} = \mathcal{L} + \sum_i \lambda_i \mathcal{L}_{reg}^i
$$
where $\lambda_i$ are hyperparameters balancing the contribution of each prior. These priors typically target specific intermediate representations (e.g., reflectance, shading, or multi-stage outputs) or statistical properties of the parameter space (e.g., posterior approximation). The implementation requires extracting these intermediate fields from the network forward pass and computing their respective penalty metrics before backpropagation.

## Subtopics

### [Multi-scale Joint Training Parameters](2967-multi-scale-joint-training-parameters.md)
This method applies specific weights to loss components calculated at each stage of a dynamically growing network, ensuring that uncertainty-aware losses and standard reconstruction losses contribute appropriately as the network expands [2967-multi-scale-joint-training-parameters]. It is distinct from static regularization by adapting weights to the hierarchical structure of the model's growth. Choose this method when training architectures that grow depth or resolution dynamically. See [Multi-scale Joint Training Parameters](2967-multi-scale-joint-training-parameters.md).

### [Reflectance and Shading Decomposition Priors](4624-reflectance-and-shading-decomposition-priors.md)
This method enforces physical plausibility by decoupling rendered color into material reflectance and shading, regularizing these intermediate components to mitigate scale ambiguity and ensure consistent brightness [4624-reflectance-and-shading-decomposition-priors]. It distinguishes itself from standard photometric priors by explicitly constraining intrinsic properties rather than just final pixel values. Choose this method for novel view synthesis involving object insertion or complex lighting where material consistency is critical. See [Reflectance and Shading Decomposition Priors](4624-reflectance-and-shading-decomposition-priors.md).

### Weight Annealing
A technique to anneal proposal weights by raising them to a power that increases from 0 to 1 over training steps, encouraging exploration by presenting a wider range of proposal intervals early in training [S1]. The annealed weight $\hat{\mathbf{w}}_n$ at step $n$ is computed as [S1]:
$$
\hat{\mathbf{w}}_n \propto \hat{\mathbf{w}}^{\frac{bn/N}{(b-1)n/N+1}}
$$
where $N$ is the total number of training steps and $b$ is the bias hyperparameter. The exponent follows Schlick's bias function, starting at 0 (flat distribution) and saturating at 1 (proposal distribution) [S1]. The bias hyperparameter is set to $b = 10$ in experiments [S1]. Ablating this annealing causes SSIM on the bicycle scene to decrease from 0.687 to 0.679 [S1].

### Laplace Approximation for Neural Networks
This method approximates the posterior distribution of network parameters as a multivariate Gaussian centered at the optimal weights $\omega^*$, using the inverse Hessian of the negative log-posterior as the covariance matrix [S2]. The covariance matrix $\Sigma$ is derived as [S2]:
$$
\Sigma = -H(\omega^*)^{-1}
$$
where $H(\omega^*)$ is the Hessian of the negative log-likelihood at the optimum [S2]. A key pitfall is that directly applying this to NeRF parameters is impracticable due to high correlation between parameters, making accurate covariance estimation challenging, and the sampling overhead is expensive [S2].

## Background
Regularization techniques in NeRFs evolved from the need to stabilize training in unbounded scenes and to extract physically meaningful properties beyond color. Weight annealing was introduced in mip-NeRF 360 to improve sampling diversity in unbounded, anti-aliased radiance fields [S1]. Reflectance and shading decomposition priors address the scale ambiguity inherent in intrinsic image decomposition within neural rendering [4624-reflectance-and-shading-decomposition-priors]. Laplace approximation for uncertainty estimation in NeRFs has been explored in contexts like underwater 3D reconstruction, though its direct application faces computational hurdles [S2].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Bayesian uncertainty analysis for underwater 3D reconstruction with neural radiance fields, https://doi.org/10.1016/j.apm.2024.115806, fetched 2026-10-29
