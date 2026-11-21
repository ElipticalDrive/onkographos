# Optimization Loops and Training Procedures

Also called: Inverse Rendering Loop

## Overview
This family defines the iterative optimization procedure that minimizes a photometric or physical loss between rendered images and ground-truth observations by updating the parameters of a scene representation (e.g., neural radiance fields, meshes, or transfer functions) [S1, unofficial]. It solves the inverse rendering problem by differentiating the volume rendering equation with respect to scene parameters, enabling gradient-based learning of geometry and appearance [44-radiance-field-optimization.md]. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by using stochastic ray sampling and adaptive integration strategies to facilitate gradient propagation, rather than deterministic rendering for metric assessment [S2, unofficial][S3].

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [44-radiance-field-optimization.md]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [44-radiance-field-optimization.md].

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [44-radiance-field-optimization.md]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [44-radiance-field-optimization.md].

Differentiation strategies fall into three main categories [44-radiance-field-optimization.md]:
1. **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [44-radiance-field-optimization.md].
2. **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [44-radiance-field-optimization.md].
3. **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [44-radiance-field-optimization.md].

## Subtopics

### [NeRF Training Loop](970-nerf-training-loop.md)
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth observations by updating the parameters of a neural radiance field. It solves the inverse rendering problem by differentiating the volume rendering equation with respect to the scene representation, enabling gradient-based learning of geometry and appearance [44-radiance-field-optimization.md]. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by using stochastic ray sampling and adaptive integration strategies to facilitate gradient propagation, rather than deterministic rendering for metric assessment.

### [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md)
This configuration defines the standard training loop for neural radiance fields and implicit surface representations, combining the Adam optimizer with a staircase exponential learning rate schedule and selective L2 weight decay. It addresses the instability of training coordinate-based neural fields by providing robust convergence parameters, particularly for noisy real-world scan data or complex geometric priors. It sets itself apart from generic optimization loops by explicitly integrating domain-specific augmentations and specific regularization techniques (like CVT for discretization stability) into the core training loop.

### [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md)
Neural Volume Rendering Training (MSE) using Hierarchical Geometric Guidance (HGG) is a training procedure for Neural Radiance Fields (NeRF) that optimizes scene density and color representations by minimizing the Mean Squared Error (MSE) between rendered and ground-truth images, while using sparse depth priors solely to guide the sampling region initialization. It differs from standard NeRF training by avoiding direct depth supervision, which introduces geometric bias due to keypoint mismatching and translation errors, and instead uses depth to define a local-to-global sampling schedule that expands from the depth prior to full scene bounds. It builds on [Optimization Loops and Training Procedures](5622-optimization-loops-and-training-procedures.md) by implementing a specific initialization and expansion strategy for the volume rendering integration bounds.

### [Optimization Baselines](6675-optimization-baselines.md)
This family provides black-box optimization strategies for inverse rendering problems where the forward rendering operator $f$ is non-differentiable or gradient computation is prohibitively expensive. Unlike gradient-based siblings that rely on automatic differentiation or adjoint methods, these methods treat the renderer as an opaque function, minimizing a photometric loss $\mathcal{L}$ by iteratively sampling and evaluating parameter candidates. The family is distinguished by its reliance on population-based search distributions (typically Gaussian) that adapt to the local loss landscape without explicit gradient information, offering robustness against local minima and Monte Carlo noise.

### [Phase-Aware Gradient Descent (PAGD)](12876-phase-aware-gradient-descent-pagd.md)
Phase-Aware Gradient Descent (PAGD) is an optimization method used to encode complex wave fields into phase-only holograms for 3D holographic displays. It solves the problem of rendering accurate depth cues and view-dependent effects by supervising the phase of the reconstructed wave field, rather than relying solely on amplitude. Unlike amplitude-only methods which suffer from unnatural blur spreading and ringing artifacts, PAGD adheres to the physical two-step hologram process (recording and reconstruction) interconnected through phase information. It is distinguished from its siblings, such as the Gerchberg-Saxton Algorithm, by using gradient-based loss minimization with a learned forward model rather than iterative hard amplitude constraints.

### [Staged and Alternating Optimization](12945-staged-and-alternating-optimization.md)
Staged and Alternating Optimization addresses the ill-posed nature and computational bottlenecks of differentiable volume and radiance field reconstruction by decoupling the optimization of coupled parameters (such as geometry, material, lighting, or camera poses) into sequential phases or alternating sub-problems. It distinguishes itself from simultaneous joint optimization methods by preventing gradient interference and local minima, allowing the solver to establish a coarse global alignment or geometric proxy before refining fine-scale details, materials, or uncertain parameters.

### [Continuous Coarse-To-Fine Optimization](13025-continuous-coarse-to-fine-optimization.md)
Continuous Coarse-To-Fine Optimization is a progressive training strategy that mitigates shape-radiance ambiguity and convergence to low-quality local optima by restricting the Level of Detail (LOD) of the scene representation to coarse levels initially, then gradually relaxing this regularization. Unlike discrete multi-stage approaches, it operates continuously in LOD space, allowing for dynamic resolution extension without modifying previously trained parameters or decoder weights. It sets itself apart from sibling methods like the standard NeRF Training Loop by explicitly managing the degrees of freedom of the spatial representation alongside the optimization variables.

### [Genetic Algorithm Optimization](13247-genetic-algorithm-optimization.md)
Genetic Algorithm (GA) Optimization is a stochastic search method used to automatically generate Direct Volume Rendering Images (DVRIs) that guarantee the visibility of specific structures of interest within a Region of Interest (ROI). Unlike gradient-based methods that optimize continuous parameters via backpropagation, this approach treats the selection of opacity seed values for atomic structures as a discrete optimization problem, using an energy function to balance visibility, coherence, and context. It is distinct from standard differentiable optimization loops because it does not rely on gradients of the rendering equation; instead, it iteratively evolves a population of candidate transfer function configurations until user-defined quality thresholds are met.

### [Shape-from-Projections Optimization Pipeline](13355-shape-from-projections-optimization-pipeline.md)
An iterative optimization pipeline that initializes a 3D mesh, refines its geometry through subdivision and cleaning, and performs differentiable rendering optimization to align the mesh with projection data. It is distinguished from general radiance-field optimization by its explicit mesh-based representation and the use of external mesh-cleaning tools to maintain manifold validity during refinement.

### [Temporal Optimization Heuristics](13688-temporal-optimization-heuristics.md)
Temporal Optimization Heuristics is a method for modeling dynamic content in differentiable bitmap primitive (DiffBMP) rendering by optimizing video frames sequentially to achieve both per-frame fidelity and temporal anti-flicker. It addresses two specific failure modes in sequential optimization: "stuck" primitives that block gradients in changing regions, and parameter drift in static regions that causes flicker. Unlike standard sequential optimization which simply warm-starts from the previous frame, this method applies three lightweight heuristics—warm-starting, removing stuck primitives, and freezing unchanged regions—to balance these competing goals.

### [Framework and Baseline Implementations](16220-framework-and-baseline-implementations.md)
This family encompasses the software architectures and training protocols used to execute differentiable volume rendering optimization. It provides the computational infrastructure for evaluating the rendering equation and propagating gradients to scene parameters, serving as the operational layer for specific methods like NeRF and Gaussian Splatting. It is distinct from its siblings by defining the low-level execution model (e.g., graph-based vs. eager-mode) and the high-level training schedule (e.g., distractor separation, multi-stage optimization) rather than the mathematical loss functions or camera models themselves.

### Geometry Focused Model Optimization
Initial Stage: Apply the rendering equation directly to each Gaussian primitive to compute outgoing radiance using its material and geometry properties, then alpha-blend during rasterization. This stabilizes gradient transfer to geometry in the initial phase [S2, unofficial].

### Polyak-Ruppert Averaging
Optimization technique to reduce wandering around the true solution due to Monte Carlo noise by maintaining a running average of parameter values [S3]. Maintain a running average of parameter values over the last 10% of the optimization steps [S3].

## Background
The concept of using a differentiable renderer inside an optimizer to estimate pose, geometry, materials, or illumination from images or other measured responses dates to the emergence of differentiable rendering [S1, unofficial]. The loop is simple; choosing a faithful model, identifiable parameters, a stable estimator, and a good initialization is difficult [S1, unofficial].

## Sources
[S1] Differentiable ray tracing — from paths to gradients | Bai Liping, https://bailiping.com/differentiable-ray-tracing/, fetched 2026-10-28 (unofficial)
[S2] Reflective Gaussian Splatting, https://doi.org/10.48550/arxiv.2412.19282, fetched 2026-10-26 (unofficial)
[S3] Material and Lighting Reconstruction for Complex Indoor Scenes with Texture-space Differentiable Rendering, https://doi.org/10.2312/sr.20211292, fetched 2026-10-29
