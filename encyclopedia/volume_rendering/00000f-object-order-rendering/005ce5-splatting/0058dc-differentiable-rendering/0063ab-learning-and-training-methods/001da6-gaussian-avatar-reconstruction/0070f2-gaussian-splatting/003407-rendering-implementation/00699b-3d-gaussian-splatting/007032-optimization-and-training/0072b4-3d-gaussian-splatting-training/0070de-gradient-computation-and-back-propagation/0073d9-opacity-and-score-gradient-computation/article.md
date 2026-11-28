# Opacity and Score Gradient Computation

Also called: Score Term Gradient Computation

## Overview
This family defines the gradient computation for opacity parameters in differentiable volume rendering, specifically addressing standard alpha-compositing, probabilistic masking, and Time-of-Flight (ToF) signal formation. It distinguishes itself from standard 3D Gaussian Splatting backpropagation by handling non-linear opacity interactions in ToF models [9733-tof-gradient-computation-for-gaussian-splatting.md] and explicit mask modulation [4254-maskgaussian-gradient-computation.md], as well as score-based probability gradients for triangle-soup representations [S1, unofficial].

## Shared principles
The core mechanism relies on the differentiable alpha-compositing equation. For a pixel $\mathbf{x}$, the rendered color $\mathbf{C}(\mathbf{x})$ is computed as:
$$
\mathbf{C}(\mathbf{x}) = \sum_{k=1}^{N} \mathbf{c}_k \alpha_k(\mathbf{x}) T_k(\mathbf{x})
$$
where $\mathbf{c}_k$ is the color of the $k$-th Gaussian, $\alpha_k(\mathbf{x})$ is its opacity at pixel $\mathbf{x}$, and $T_k(\mathbf{x})$ is the transmittance up to the $k$-th Gaussian:
$$
T_k(\mathbf{x}) = \prod_{j=1}^{k-1} (1 - \alpha_j(\mathbf{x}))
$$
The Gaussians are sorted by depth from closest to farthest [18365-3d-gaussian-splatting-backward-pass.md].

The gradient of the loss $L$ with respect to the parameters $\theta_k$ of the $k$-th Gaussian is computed via the chain rule:
$$
\frac{\partial L}{\partial \theta_k} = \frac{\partial L}{\partial \mathbf{C}(\mathbf{x})} \frac{\partial \mathbf{C}(\mathbf{x})}{\partial \alpha_k(\mathbf{x})} \frac{\partial \alpha_k(\mathbf{x})}{\partial \theta_k}
$$
The term $\frac{\partial \mathbf{C}(\mathbf{x})}{\partial \alpha_k(\mathbf{x})}$ represents the contribution of the $k$-th Gaussian to the pixel color gradient. It is computed efficiently using recursive accumulators traversed in reverse depth order to avoid $O(N^2)$ complexity [18365-3d-gaussian-splatting-backward-pass.md]. Specifically:
$$
\frac{\partial \mathbf{C}(\mathbf{x})}{\partial \alpha_k(\mathbf{x})} = T_k(\mathbf{x}) \mathbf{c}_k - \alpha_k(\mathbf{x}) \sum_{j=k+1}^{N} \mathbf{c}_j \alpha_j(\mathbf{x}) T_j(\mathbf{x}) \frac{1}{1-\alpha_k(\mathbf{x})}
$$
In practice, this is implemented by maintaining a running transmittance $T$ and a running accumulated color $C_{acc}$ [18365-3d-gaussian-splatting-backward-pass.md].

The term $\frac{\partial \alpha_k(\mathbf{x})}{\partial \theta_k}$ depends on the parameterization:
*   **Opacity**: $\alpha_k$ is typically the sigmoid of a learnable scalar. The gradient is $\frac{\partial \alpha_k}{\partial \text{opacity}_{raw}} = \alpha_k (1 - \alpha_k)$ [18365-3d-gaussian-splatting-backward-pass.md].
*   **2D Projection**: The 2D covariance $\Sigma_{2D}$ is derived from the 3D covariance $\Sigma_{3D}$ and the view matrix. The conic matrix $C = \Sigma_{2D}^{-1}$ is used for efficient evaluation. Gradients $\frac{\partial \alpha_k}{\partial C}$ are computed based on the Gaussian PDF derivative [18365-3d-gaussian-splatting-backward-pass.md].
*   **3D Covariance**: $\Sigma_{3D}$ is parameterized by scale and rotation ($\Sigma_{3D} = R S S^T R^T$). Gradients flow to scale and rotation via the chain rule through $\Sigma_{3D}$ and then to $\Sigma_{2D}$ [18365-3d-gaussian-splatting-backward-pass.md].
*   **Color/SH**: $\mathbf{c}_k$ is computed from spherical harmonic (SH) coefficients. Gradients flow directly to SH coefficients based on the basis function evaluation at the viewing direction [18365-3d-gaussian-splatting-backward-pass.md].

## Subtopics

### [MaskGaussian Gradient Computation](4254-maskgaussian-gradient-computation.md)
Computes gradients for probabilistic masks $\mathcal{M}_i$ that modulate Gaussian opacity, distinguishing itself by accounting for mask effects on both foreground accumulation and background transmittance.

### [ToF Gradient Computation for Gaussian Splatting](9733-tof-gradient-computation-for-gaussian-splatting.md)
Computes opacity gradients for Time-of-Flight signals, distinguished by squared transmittance terms $(1-\alpha)^2$ in the recursive accumulator to handle non-linear ToF signal integration.

### [3D Gaussian Splatting Backward Pass](18365-3d-gaussian-splatting-backward-pass.md)
The standard RGB alpha-compositing backward pass using linear transmittance and color accumulation.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
Optimizes the backward pass for GPU parallel execution.

### [AbsGS](6127-absgs.md)
Uses absolute gradients for training stability.

### [Gradient-Weighted Feature Back-Projection](9769-gradient-weighted-feature-back-projection.md)
Back-projects gradients to features.

### [SNS Backward Pass with Intermediate Variable Q](10040-sns-backward-pass-with-intermediate-variable-q.md)
Uses intermediate variable Q for specific rendering pipelines.

### [Specular-Aware Gradient Gating](13432-specular-aware-gradient-gating.md)
Gates gradients based on specular highlights.

### [Deferred Back-propagation](13506-deferred-back-propagation.md)
Defers gradient computation for memory efficiency.

### [Weight Freezing by Mask](15877-weight-freezing-by-mask.md)
Freezes weights based on mask values.

### [Softmax Depth Gradient Derivation](17405-softmax-depth-gradient-derivation.md)
Derives gradients for softmax-based depth sorting.

### Score Term Gradient Computation
#### definition
- [S1, unofficial] The gradient of the log probability with respect to opacity parameters, used for computing the loss gradient in triangle-soup representations.
#### formula
- [S1, unofficial] \frac{\partial \log p(f)}{\partial \alpha_{f'}} = \begin{cases} 1/\alpha_f, & f' = f \\ -1/(1 - \alpha_{f'}), & D_{f'} < D_f \end{cases}

## Background
Originated in the context of differentiable rasterization for triangle-soup geometry, specifically in the DiffSoup method [S1, unofficial]. The method addresses the optimization of triangle-based representations by deriving gradients for opacity parameters that influence the visibility probability of triangles. The score term gradient is derived from the chain rule applied to the probability product expression of triangle visibility [S1, unofficial].

## Sources
[S1] DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification, https://doi.org/10.48550/arXiv.2603.27151, fetched 2026-10-30 (unofficial)
