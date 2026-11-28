# Smoothness Priors

Also called: Geometric Smoothness Regularization, Gradient Smoothness Loss, SDF Total Variation Loss, Scene flow smoothness loss, Spatial and temporal smoothness priors

## Overview
Smoothness priors are regularization terms applied during training and post-processing to reduce surface noise and improve geometric smoothness in neural radiance fields and signed distance fields [S1, unofficial][S2]. This family builds on [Output Noise Regularization](928-output-noise-regularization.md) and applies distinct loss formulations to enforce smoothness across different geometric representations.

## Shared principles
The family applies regularization losses to output fields to reduce artifacts [S1, unofficial][S3, unofficial]. The core mechanism involves penalizing differences between neighboring values [S2] or enforcing alignment with ground-truth geometric cues [S4].

## Subtopics

### Scene Flow Smoothness
A spatial and temporal smoothness prior used to regularize scene flow in dynamic scene representations [S2]. It is chosen when optimizing for dynamic scenes where consistent motion fields are required [S2].

For a ray $r$, let $x$ and $x_0$ be neighboring points along the ray, $t$ index the time step, and $s_t^0(x, t)$ denote the scene flow value at point $x$ and time $t$ [S2]. The smoothness loss $L_{sm}(r)$ is defined as [S2]:

$$
L_{sm}(r) = \sum_t \sum_x \exp\left(-2|x - x_0|\right) \left(s_t^0(x, t) - s_t^0(x_0, t)\right)^2
$$

The exponential term $\exp\left(-2|x - x_0|\right)$ serves as a distance weight [S2]. The total loss sums over all time steps $t$ and spatial points $x$ along the ray [S2].

### Opacity and Normal Regularization Priors
Two loss terms applied during training to enforce physical plausibility of the learned occupancy and normal fields [S3, unofficial]. They are chosen to prevent specific artifacts in surface reconstruction, particularly for human faces or complex geometry [S3, unofficial].

The method applies an opacity loss $L_{opacity}$ to constrain the opacity distribution and a normal loss $L_{normal}$ to constrain surface normals [S3, unofficial].

**Pitfalls of removal:**
*   Removing $L_{opacity}$ causes smoke-like artifacts around cheeks and makes faces too flat/unatural [S3, unofficial].
*   Removing $L_{normal}$ causes specular highlights, hollows, and messy checkerboard-like normal maps [S3, unofficial].

### Surface Fitting Loss (LFit)
A loss term improving surface smoothness by fitting surface points to a plane determined by depth and normal supervision, weighted by gradient-normal alignment confidence [S4]. It is chosen when depth and normal ground truth are available to guide the SDF geometry [S4].

The loss pushes points onto a plane defined by $\alpha x + \beta y + \gamma z + \mu = 0$ [S4]. The plane parameters $[\alpha, \beta, \gamma, \mu]$ are solved using Ground Truth (GT) depth $D_{kGT}(r)$ and GT normal $N_{kGT}(r)$ of the ray $r$ [S4].

The loss is computed as [S4]:

$$
L_{Fit} = \sum_{p'_j \in s} w_j \times \eta_j \times ||\alpha x_j + \beta y_j + \gamma z_j + \mu||^2
$$

where:
*   $p'_j$ are surface points in a patch $s$ [S4].
*   $w_j$ is a mask from the Depth Consistency loss (Eq. 10 in [S4]), ruling out points whose projections are out of view range or have a depth error $|D_{kGT}(p'_j) - Z_k(p'_j)| > 15$mm [S4].
*   $\eta_j$ is a confidence weight based on the cosine distance between the SDF gradient and the GT normal: $\eta_j = \cos(\nabla f_\theta(p'_j, e(p'_j)), N_{kGT}(r))$ [S4].
*   The term $||\alpha x_j + \beta y_j + \gamma z_j + \mu||^2$ measures the squared distance of point $p'_j=(x_j, y_j, z_j)$ from the plane [S4].

This formulation pushes points onto the plane more strongly if their gradient is well aligned with the GT normal [S4]. The gradient $\nabla f_\theta(p'_j, e(p'_j))$ involves second-order derivatives [S4].

### Gradient Smoothness and SDF Total Variation
Regularization terms applied during fine training to reduce surface noise [S1, unofficial].

1.  **Gradient Smoothness Loss:** Applied during the fine training stage to enforce smoothness in the gradient field [S1, unofficial].
2.  **SDF Total Variation (TV) Loss:** Applied on the SDF voxel grid to regularize the signed distance field values [S1, unofficial].

**Ablation Results:**
*   Removing the Gaussian kernel during coarse training increases Chamfer Distance (CD) from 0.72 to 0.74 [S1, unofficial].
*   Removing the Gaussian kernel during post-processing reduces CD from 0.720 to 0.715 [S1, unofficial].
*   The effectiveness of the gradient smoothness loss is verified via quantitative comparisons in ablation studies [S1, unofficial].

### Surface Tightness Regularization
A regularization term to ensure the surface is tight by minimizing the rendered variance image $B$, which is computed by replacing radiance $c_i$ with variance $\beta_i$ in the volume rendering integral [S5].

The loss is computed as $L_{surface} = \sum_{hw} (B_{hw} - B_{target})^2$, where $B_{target}$ is a target value [S5].

### SDF Sharpness Regularization
A regularization term on the sigmoid sharpness parameter $s$ to ensure convergence of the SDF representation in complex urban scenes where $s$ might not naturally approach zero [S6].

The regularization loss is $L_s = 1/(s + \epsilon)$, where $s$ is the sharpness parameter and $\epsilon$ is a small constant [S6]. Unlike StreetSurf which manually fixes $s$ to increment linearly, ViiNeuS uses a learnable $s$ with regularization [S6].

### Smoothness Loss
Applies a Laplacian loss in world coordinates to encourage smoothness [S7, unofficial]. A smoothness term applied to the reconstruction [S7, unofficial].

Laplacian approximation: $\nabla^2 f(p'_i) \approx \frac{f(x'_i + 1, y'_i, z'_i) - 2f(p'_i) + f(x'_i - 1, y'_i, z'_i)}{h_L^2} + \partial_{yy} f + \partial_{zz} f$ [S7, unofficial].

Final Smoothness loss: $L_{smooth} = \frac{1}{N_{samples}} \sum_{i=1}^{N_{samples}} \nabla^2 f(p'_i)^2$ [S7, unofficial].

Base weight $\lambda_s = 10^{-10}$. Decays by a factor of 0.25 every 2,000 iterations during coarse levels [S7, unofficial]. After resolution level 29 (iterations $\tau = 6,000$ to $8,000$), stop decaying and keep $\lambda_s$ fixed while applying the loss in the parent–child hierarchy [S7, unofficial]. For Tanks-and-Temples, $L_{smooth}$ is kept active until $\tau = 10,000$ without further decay [S7, unofficial].

## Background
Voxurf introduced 3D convolution with a Gaussian kernel for coarse training and gradient smoothness/SDF TV losses for fine training to boost reconstruction smoothness [S1, unofficial]. SUDS adapted spatial and temporal smoothness priors from NSFF to regularize scene flow in scalable urban dynamic scenes [S2]. Generative Occupancy Fields (GOF) applied opacity and normal regularization to prevent artifacts in 3D surface-aware image synthesis [S3, unofficial]. Sensing Surface Patches introduced the Surface Fitting Loss (LFit) to infer SDFs by fitting points to planes defined by depth and normal supervision [S4]. What You See is What You GAN introduced surface tightness regularization by minimizing rendered variance [S5]. ViiNeuS introduced SDF sharpness regularization to handle learnable sharpness parameters in urban scenes [S6]. SVRecon applied a Laplacian smoothness loss with a decaying weight schedule for sparse voxel rasterization [S7, unofficial].

## Sources
[S1] Voxurf: Voxel-based Efficient and Accurate Neural Surface Reconstruction, https://doi.org/10.48550/arxiv.2208.12697, fetched 2026-10-26 (unofficial)
[S2] SUDS: Scalable Urban Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.01191, fetched 2026-10-26
[S3] Generative Occupancy Fields for 3D Surface-Aware Image Synthesis, https://doi.org/10.48550/arxiv.2111.00969, fetched 2026-10-29 (unofficial)
[S4] Sensing Surface Patches in Volume Rendering for Inferring Signed Distance Functions, https://doi.org/10.1609/aaai.v39i4.32426, fetched 2026-10-26
[S5] What You See is What You GAN: Rendering Every Pixel for High-Fidelity Geometry in 3D GANs, https://doi.org/10.1109/cvpr52733.2024.02148, fetched 2026-10-26
[S6] ViiNeuS: Volumetric Initialization for Implicit Neural Surface reconstruction of urban scenes with limited image overlap, https://hal.science/hal-05104965v1, fetched 2026-10-26
[S7] SVRecon: Sparse Voxel Rasterization for Surface Reconstruction, https://doi.org/10.48550/arxiv.2511.17364, fetched 2026-10-26 (unofficial)
