# Uncertainty and Probabilistic Modeling

Also called: Probabilistic Gaussian Splatting, Uncertainty Quantification in 3DGS

## Overview
This family of methods addresses the lack of confidence metrics in standard 3D Gaussian Splatting (3DGS) by introducing probabilistic frameworks, uncertainty quantification, and active view selection. Unlike deterministic rendering, these approaches model the scene representation or data acquisition process as stochastic, enabling the estimation of model confidence, robust handling of missing or asynchronous data, and optimization of training view selection. What sets this family apart from siblings like [Pruning and Density Control](16215-pruning-and-density-control.md) or [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md) is the explicit modeling of epistemic or aleatoric uncertainty to guide training, visualization, or active learning, rather than merely optimizing geometric fidelity or storage efficiency.

## Shared principles
The core mechanism across these methods is the introduction of stochasticity into the Gaussian Splatting pipeline, either by treating parameters as random variables, quantifying information gain via second-order derivatives, or aggregating visibility metrics as uncertainty proxies.

1.  **Probabilistic Parameterization**: In frameworks like [GraphiXS](12929-360-gaussian-splatting.md) and [latentSplat](17968-variational-bayes-gaussian-splatting.md), learnable parameters (e.g., Gaussian means, covariances, opacities) are treated as latent variables. Camera poses and frame times may also be modeled as random variables rather than fixed observations [S2]. Inference is performed via Maximum a Posteriori (MAP) estimation, incorporating prior distributions to regulate component behavior [S2].
2.  **Fisher Information as Uncertainty Proxy**: Methods like [FisherRF](15865-fisherrf.md) and [FreeFix](15863-gaussian-frosting.md) utilize the Fisher Information Matrix (FIM) to quantify uncertainty. The FIM is approximated by the Hessian of the negative log-likelihood with respect to the model parameters [S1]. For 3DGS, the uncertainty attribute $\bar{C}_{V;G}$ for a Gaussian $G$ under view $V$ is derived from the negative log-likelihood of the rendered pixel $\pi$:
    $$ \bar{C}_{V;G} = -\log p_f(\pi|V; G) $$
    This is equivalent to the expected value of the second derivative of the log-likelihood (the Hessian $H[\pi|V; G]$), provided regularity conditions are met: the partial derivative of the probability density function $p_f$ with respect to $G$ exists almost everywhere, the integral of $p_f$ can be differentiated under the integral sign, and the support of $p_f$ does not depend on $G$ [S1].
3.  **Expected Information Gain (EIG) for View Selection**: Active view selection leverages the sparsity of the Hessian. The EIG for a candidate view is approximated by the trace of the product of the Hessian of the candidate view ($H_{cand}$) and the Hessian of previous views ($H_{prev}$):
    $$ \text{EIG} \approx \text{Tr}(H_{cand} \cdot H_{prev}) $$
    This computation is efficient due to the sparsity of the Hessian in 3DGS, allowing real-time evaluation [S1].
4.  **Visibility-Based Uncertainty Aggregation**: Post-training uncertainty representations, such as in [PRIMU](12929-360-gaussian-splatting.md), aggregate contribution factors $\alpha_k(\bar{x}) T_k(\bar{x})$ over visible pixels. The training-view coverage $V(k)$ for a Gaussian $k$ is defined as:
    $$ V(k) = \max_{v \in \Omega} \text{agg}_{\bar{x} \in v^{(k)}} \alpha_k(\bar{x}) T_k(\bar{x}) $$
    where $\Omega$ is the set of training views, $v^{(k)}$ are pixels where opacity exceeds a threshold, and $\text{agg}$ is an aggregation function (max, sum, or mean) [S3, unofficial].

## Subtopics

### [FisherRF](15865-fisherrf.md)
FisherRF is a method for active view selection and uncertainty quantification in Radiance Fields. It sets itself apart by focusing on the *selection* of new training views to maximize Expected Information Gain (EIG), rather than just optimizing existing parameters. It leverages the sparsity of the Hessian to achieve real-time evaluation speeds (70 fps).

### [GraphiXS](12929-360-gaussian-splatting.md)
GraphiXS is a probabilistic framework that incorporates data uncertainty (view sparsity, missing frames, camera asynchronization) into Gaussian Splatting. It treats learnable parameters as latent variables and camera poses/times as random variables, inferred via Maximum a Posteriori (MAP). The 'X' allows instantiation with different primitives (e.g., Gaussians, Student’s-t). It is a general framework that can upgrade existing methods to handle multiple types of data uncertainty.

### [PRIMU](12929-360-gaussian-splatting.md)
PRIMU provides uncertainty estimation via primitive-based representations of error and coverage. It computes scalar uncertainty features per Gaussian primitive, such as the number of training views observing the primitive ($F(k)$) or the aggregated contribution factors ($V(k)$). These representations are computed in a post-processing step without modifying the rendering or optimization.

### [latentSplat](17968-variational-bayes-gaussian-splatting.md)
latentSplat uses variational Gaussians to model uncertainty. It renders the standard deviation $\sigma$ of variational Gaussian features to visualize model uncertainty. Background regions outside reference camera frustums are explicitly handled by setting the background standard deviation to one, matching the feature map sampling strategy. This results in darker regions (higher uncertainty) for invisible or partially visible backgrounds.

### 3DGS Uncertainty Calculation via Fisher Information
This variant defines the uncertainty attribute of 3DGS based on the negative log-likelihood of the rendered pixel value, interpreted as Fisher information.
- **Definition**: Computes an uncertainty attribute for 3D Gaussians based on the negative log-likelihood of the rendered pixel value, interpreted as Fisher information [S1].
- **Step**: Compute the dot product of the gradient with itself to obtain the uncertainty value $\bar{C}_{V;G}$ [S1].
- **Formula**: $\bar{C}_{V;G} = -\log p_f(\pi|V; G)$ where $p_f$ is the probability density function of the rendered pixel $\pi$ given view $V$ and Gaussian parameters $G$ [S1].
- **Pitfall**: The derivation assumes volume rendering of 3D Gaussians meets regularity conditions: partial derivative exists almost everywhere, integral differentiability, and support independence from $G$ [S1].

### Training-View Coverage Representation
This variant represents uncertainty by aggregating visibility metrics.
- **Definition**: A scalar uncertainty feature per Gaussian primitive $k$ representing the number of training views that observe the primitive, or capturing how strongly the primitive is covered in at least one training view, based on its contribution factors [S3, unofficial].
- **Formula**: $F(k) = |\Omega_k|$, where $\Omega_k$ is the set of training views where the $k$-th Gaussian primitive lies within the viewing cone. $V(k) = \max_{v \in \Omega} \text{agg}_{\bar{x} \in v^{(k)}} \alpha_k(\bar{x}) T_k(\bar{x})$, where $v^{(k)}$ denotes the pixels of view $v$ in which the opacity of the $k$-th Gaussian exceeds a small threshold, and agg is an aggregation function [S3, unofficial].
- **Parameter**: Aggregation function agg: Can be 'max', 'sum', or 'mean'. Opacity threshold: Gaussians are considered visible if $\alpha_k(\bar{x}) \ge 1/255$ and $T_k(\bar{x}) \ge 10^{-3}$ [S3, unofficial].
- **Pitfall**: To highlight low-opacity Gaussians that still influence reconstruction quality, the opacity term $\alpha_k(\bar{x})$ can be omitted from the aggregation [S3, unofficial].

### Uncertainty Visualization
This variant focuses on rendering uncertainty maps.
- **Definition**: Renders the standard deviation $\sigma$ of variational Gaussian features to visualize model uncertainty, with background regions explicitly handled [S4, unofficial].
- **Step**: Sets the background standard deviation to one for empty regions outside reference camera frustums, matching the feature map sampling strategy. Renders darker regions as higher uncertainty; main objects show lower uncertainty than background, with higher uncertainty at edges/fur compared to uniform surfaces [S4, unofficial].

## Background
The concept of uncertainty in neural rendering has evolved from error estimation in NeRFs to probabilistic modeling in Gaussian Splatting. Early works focused on post-hoc error estimation [S3, unofficial]. The integration of Fisher Information for active learning in 3DGS was introduced in [FreeFix] [S1]. The general probabilistic framework for 4DGS, GraphiXS, was proposed to handle data uncertainty holistically [S2]. Variational approaches like latentSplat introduced Bayesian inference to Gaussian Splatting [S4, unofficial].

## Sources
[S1] FreeFix: Boosting 3D Gaussian Splatting via Fine-Tuning-Free Diffusion Models, https://doi.org/10.1109/3dv69130.2026.00109, fetched 2026-10-30
[S2] Graphical X Splatting (GraphiXS): A Graphical Model for 4D Gaussian Splatting under Uncertainty, https://doi.org/10.1145/3799902.3811085, fetched 2026-11-05
[S3] PRIMU: Uncertainty Estimation for Novel Views in Gaussian Splatting from Primitive-Based Representations of Error and Coverage, https://arxiv.org/abs/2508.02443v2, fetched 2026-10-26 (unofficial)
[S4] latentSplat, https://arxiv.org/abs/2403.16292, fetched 2026-10-27 (unofficial)
