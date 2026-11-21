# Optimization Schedule and Densification


## Overview
This family defines the temporal scheduling of primitive creation, removal, and parameter updates during 3D Gaussian Splatting (3DGS) training. It solves the problem of balancing reconstruction fidelity against optimization complexity by synchronizing the growth of Gaussian primitives with rendering resolution and error metrics. It sets itself apart from static initialization methods by employing dynamic, often concave-up, growth curves that accelerate primitive count as high-frequency details are fitted, distinct from sibling methods like [Reconstruction-aware Pruning Scheduler](16199-reconstruction-aware-pruning-scheduler.md) which focus on pruning thresholds, or [DropGaussian](16856-dropgaussian.md) which focuses on specific dropout mechanisms.

## Shared principles
All methods in this family operate on the standard 3DGS primitive representation:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$
*   Scale $\mathbf{s}_i \in \mathbb{R}^3$
*   Rotation $\mathbf{q}_i \in \mathbb{R}^4$ (quaternion)
*   Opacity $\alpha_i \in [0, 1]$
*   Color coefficients $\mathbf{c}_i$ (Spherical Harmonics)

The core mechanism involves calculating an importance score for each Gaussian and removing those below a threshold or ranking. Methods typically employ a "concave-up" growth curve for primitives, where growth accelerates over time, synchronized with increasing rendering resolution to fit high-frequency details only after low-frequency structures are established [Parent].

## Subtopics

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Error-Driven 2D Gaussian Training is a densification strategy that samples and optimizes 2D Gaussian kernels in image space using a differentiable rasterizer, then back-projects them to 3D to initialize new 3D primitives. It solves the problem of initializing view-dependent Gaussian kernels for complex specular radiance reconstruction, where proper initialization is critical. It sets itself apart from standard 3D densification by optimizing in 2D image space with the pre-rendered image as a fixed background, allowing for precise error-driven placement before 3D projection. See [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md).

### [DashGaussian](15742-dashgaussian.md)
DashGaussian is a scheduling scheme for 3D Gaussian Splatting (3DGS) that accelerates optimization by reformulating the process as a frequency component fitting task. It cooperatively allocates rendering resolution and primitive count to strip redundant complexity, distinguishing itself from vanilla 3DGS and other backbones by producing a concave-up primitive growth curve that spans the entire optimization process. Unlike methods that peak in primitive count at half-iteration or use concave-down curves, DashGaussian synchronizes primitive growth with increasing resolution to fit high-frequency details only after low-frequency structures are established. See [DashGaussian](15742-dashgaussian.md).

### [Training Configuration and Hyperparameters](16385-training-configuration-and-hyperparameters.md)
This article defines the specific hyperparameters, optimization schedules, and training protocols required to converge 3D Gaussian Splatting models, particularly in generative and downstream transfer contexts. It addresses the configuration of learning rates, iteration counts, and loss weights necessary to stabilize training, while also detailing standardized protocols for downstream tasks like classification and segmentation. It differs from sibling methods by focusing on the *configuration* of the training loop itself rather than the geometric densification logic or rendering primitives. See [Training Configuration and Hyperparameters](16385-training-configuration-and-hyperparameters.md).

### QuickSplat Optimization Process
**Definition**
An optimization framework for 3D Gaussian Splatting that uses learned networks for initialization, densification, and parameter updates to accelerate surface reconstruction [S1].

**Steps**
1.  Initialize Gaussians $G_0$ from SfM points $P$ using an initializer network $f_I$: $G_0 \leftarrow f_I(P)$ [S1].
2.  Iterate for $t = 0$ to $T - 1$ [S1].
3.  Reset gradient accumulator $\nabla G_t \leftarrow 0$ [S1].
4.  For all images, compute rendering loss $L$, compute gradient $\delta G_t$, and accumulate: $\nabla G_t \leftarrow \nabla G_t + \delta G_t$ [S1].
5.  Generate new Gaussians $\hat{G}_t$ using a densifier network $f_D$ on current Gaussians $G_t$, accumulated gradients $\nabla G_t$, and time step $t$: $\hat{G}_t \leftarrow f_D(G_t, \nabla G_t, t)$ [S1].
6.  Concatenate new Gaussians: $\bar{G}_t \leftarrow G_t \cup \hat{G}_t$ [S1].
7.  Compute update $\Delta \bar{G}_t$ using an optimizer network $f_O$: $\Delta \bar{G}_t \leftarrow f_O(\bar{G}_t, \nabla \bar{G}_t, t)$ [S1].
8.  Update Gaussians: $G_{t+1} \leftarrow \bar{G}_t + \Delta \bar{G}_t$ [S1].

### BlitzGS Optimization Schedule
**Steps**
1.  Disable the Level-of-Detail (LOD) gate during the density-control window (iterations 2,000–32,000) [S2, unofficial].
2.  Engage the LOD gate after the first importance-scoring pass ($T_1$) [S2, unofficial].
3.  After the importance-scoring pass, reweight density-control gradients using the per-Gaussian visibility factor $\phi_i$ for the remainder of the density-control window [S2, unofficial].

**Parameters**
*   Density control interval: Runs every 1,000 steps from iteration 2,000 up to 32,000 (last event at 31,000) [S2, unofficial].

**Comparison**
Extending training iterations (e.g., 500k–1000k) does not significantly improve quality on aerial scenes (e.g., Building 19.27→19.50 dB), indicating deficits stem from representation structure rather than insufficient budget [S2, unofficial].

### Normal Gradient Based Densification
**Definition**
A densification criterion to improve normal recovery in thin regions by introducing an additional densification criterion on the gradient of normals, beyond the vanilla 3DGS gradient of view space points [S3]. It densifies Gaussians whose normal gradient exceeds a threshold $T_n$ [S3].

**Steps**
1.  Calculate the normal gradient for each Gaussian [S3].
2.  Densify Gaussians whose normal gradient exceeds a threshold $T_n$ [S3].

**Parameters**
*   $T_n$: The threshold for normal gradient magnitude used to trigger densification. Typical value: $2 \times 10^{-9}$ [S3].

### Global-to-local densification
**Definition**
A densification strategy that decouples split and clone operations across densification phases to accelerate optimization. It uses split for global spread at low resolution and suppresses clone, then reintroduces clone for local refinement at full resolution [S4, unofficial].

**Steps**
1.  Calculate the norm of the average position gradient of each Gaussian primitive every 100 iterations [S4, unofficial].
2.  If the gradient norm exceeds a predefined threshold, check the maximum scale of the Gaussian primitive [S4, unofficial].
3.  If the maximum scale exceeds a given scale threshold, split the Gaussian into smaller components [S4, unofficial].
4.  Otherwise, clone the Gaussian with the same parameters [S4, unofficial].
5.  During low-resolution training, promote Gaussian primitives' global spread with the split operation and suppress clone operations to prevent premature local clustering [S4, unofficial].
6.  When transitioning to full-resolution training, reintroduce clone operations to refine high-frequency details [S4, unofficial].

**Parameters**
*   Gradient calculation frequency: 100 iterations [S4, unofficial].

**Comparison**
Unlike standard 3DGS which applies split and clone simultaneously, this method decouples them across phases (global vs local) to address inefficient spatial spread and redundant primitives [S4, unofficial].

### Resolution Guided Primitive Scheduler
**Formula**
The final primitive count $P_{fin}^{(i)}$ at step $i$ is calculated as [S5]:
$$ P_{fin}^{(i)} = \max(P_{fin}^{(i-1)}, \gamma \cdot P_{fin}^{(i-1)} + \eta \cdot P_{add}) $$
where $P_{add}$ is the number of primitives densified in step $i$. $P_{fin}^{(0)}$ is initialized as $5 \cdot P_{init}$ [S5].

**Parameters**
*   Power factor in primitive growth: Starts at 2 and gradually decreases to 1 along the optimization process to suppress early growth and encourage mid-stage growth [S5].
*   Hyperparameter $\gamma$: Momentum decay factor for primitive budget. Default value: 0.98 [S5].
*   Hyperparameter $\eta$: Force scaling factor for primitive budget. Default value: 1 [S5].
*   Resolution modulation: The continuous scalar $r(i)$ is modulated to $\lfloor r(i) \rfloor$ in practice to encourage faster primitive growth [S5].

**Pitfall**
Too much primitive growth at the end of optimization results in poor convergence. Mitigated by the concave-up scheduling function and momentum-based budgeting [S5].

### 3D Refinement View Sampling Strategy
**Definition**
A staged sampling schedule for supervising 3D refinement that balances fitting the current refined view with preserving rendering ability in original and previously refined views [S6].

**Steps**
1.  Divide the total 3D refinement steps into three equal phases (first third, subsequent third, final third) [S6].
2.  In the first third, designate every 3 steps as 'current-refine steps' to optimize using the current refined image $\hat{I}^{i,e,f}_{ie}$ [S6].
3.  In the subsequent third, designate every 5 steps as 'current-refine steps' [S6].
4.  In the final third, designate every 8 steps as 'current-refine steps' [S6].
5.  For all non-current-refine steps, randomly sample views from the training set $S_{train}$ and the previous refined set $F_{i-1}$ [S6].

**Parameters**
*   Selection weight for $F_{i-1}$ is lower than for $S_{train}$ to prioritize original training data over previously refined views [S6].

## Background
The concept of optimizing 3D Gaussian Splatting involves balancing the number of primitives against rendering quality and training time. Early works like vanilla 3DGS used fixed schedules for densification and pruning [Parent]. Subsequent research introduced dynamic scheduling, such as DashGaussian, which reformulates optimization as frequency fitting to accelerate convergence [S5]. Other approaches, like QuickSplat, utilize learned networks for densification [S1], while BlitzGS focuses on system-level workload reduction through LOD gating [S2, unofficial]. Normal gradient-based densification was introduced to improve geometric fidelity in thin regions [S3]. Global-to-local strategies decouple split and clone operations to improve efficiency [S4, unofficial]. Finally, 3D refinement strategies employ staged view sampling to preserve rendering ability during fine-tuning [S6].

## Sources
[S1] QuickSplat: Fast 3D Surface Reconstruction via Learned Gaussian Initialization, https://doi.org/10.1109/ICCV51701.2025.02585, fetched 2026-10-29
[S2] BlitzGS: City-Scale Gaussian Splatting at Lightning Speed, https://doi.org/10.48550/arXiv.2605.13794, fetched 2026-10-29 (unofficial)
[S3] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
[S4] Decomposing Densification in Gaussian Splatting for Faster 3D Scene Reconstruction, https://doi.org/10.48550/arXiv.2507.20239, fetched 2026-10-30 (unofficial)
[S5] DashGaussian: Optimizing 3D Gaussian Splatting in 200 Seconds, https://doi.org/10.1109/cvpr52734.2025.01041, fetched 2026-10-31
[S6] FreeFix: Boosting 3D Gaussian Splatting via Fine-Tuning-Free Diffusion Models, https://doi.org/10.1109/3dv69130.2026.00109, fetched 2026-10-30
