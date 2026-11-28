# VF-NeRF

Also called: Vector Field NeRF, Viewshed Fields for Rigid NeRF Registration

## Overview
VF-NeRF is a method for rigid registration between two Neural Radiance Fields (NeRFs) when the original camera positions are unknown. It introduces **Viewshed Fields (VF)**, an implicit function that maps an oriented point (3D location and viewing direction) to a scalar indicating how well that point was covered by the original camera set. By using Normalizing Flows to generate high-scoring oriented points, VF-NeRF creates informative virtual camera viewpoints for photometric alignment, avoiding the noise and poor coverage associated with random sampling. It differs from [NeRF Importance Sampling](1565-nerf-importance-sampling.md) by focusing on registration via novel view generation rather than training efficiency, and from [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md) by its specific application to inter-NeRF alignment.

## How it works
VF-NeRF operates in two distinct modes depending on the source notes: **Registration** (using Viewshed Fields to align two NeRFs) and **Scene Representation** (using Vector Fields to render scenes from posed images). The registration method is the primary algorithmic contribution described in [S1], while the vector field rendering is described in [S2, unofficial].

### Inputs and Outputs
*   **Registration Mode Inputs**: Two pre-trained NeRFs (Source and Target), without camera poses.
*   **Registration Mode Outputs**: A 6-DoF rigid transformation matrix aligning the Source NeRF to the Target NeRF.
*   **Rendering Mode Inputs**: Multi-view posed images and monocular depth maps.
*   **Rendering Mode Outputs**: Rendered RGB images and depth maps of the indoor scene.

### Data Structures
The model consists of the following components [S2, unofficial]:
1.  **`vector_field_network`**: A `VectorFieldNetwork` instance.
2.  **`rendering_network`**: A `RenderingNetwork` instance.
3.  **`density`**: A `LaplaceDensity` function initialized with parameters from `config.density_config.todict()`.
4.  **`fine_vector_field_network`**: Optional `VectorFieldNetwork` instance, used if fine sampling is enabled. Defaults to sharing the `vector_field_network` instance if not explicitly configured.
5.  **`ray_sampler`**: A `UniformSampler` with parameters `n_samples`, `near`, `far`, and `perturb` (inverted boolean).
6.  **`fine_sampler`**: Optional `RangeFineSampler` with parameters `n_importance`, `near`, `far`, `perturb` (inverted), `range` (`config.ray_sampler_config.fine_range`), and `max_samples` (`config.ray_sampler_config.max_samples`).

### Algorithm: Registration via Viewshed Fields (VF)
This method aligns two NeRFs by generating virtual views from the source NeRF using VF and optimizing a rigid transform.

1.  **VF Network Initialization**: Initialize a Normalizing Flows model using Real-NVP [13] with $L=4$ layers and $H=128$ hidden dimensions. The model maps 6D oriented points $(x, d)$ to a 6-dimensional diagonal Gaussian in latent space $z$.
2.  **VF Training**:
    *   During NeRF training, sample oriented points $(x, d)$ around surfaces. $d$ is constant along a ray, $x$ varies.
    *   Map sampled points to latent space $z$.
    *   Optimize the VF network by minimizing the negative log-likelihood of the data distribution $p_X(x)$ using the change of variables formula [S1]:
        $$ \log(p_X(x)) = \log(p_Z(f(x))) + \sum_{i=1}^{K} \log \left| \det \left( \frac{df_i}{df_{i-1}} \right) \right| $$
    *   Use RAdam optimizer with learning rate $5 \times 10^{-5}$.
    *   **Real Scenes**: Train NeRF for 60K iterations; enable VF training on the last 10K iterations.
    *   **Synthetic Scenes**: Train NeRF for 20K iterations; enable VF training on the last 5K iterations. Ignore samples where image alpha $< 128$.
3.  **Point Cloud Generation (PC Initialization)**:
    *   Sample 100,000 points from the learned VF distribution.
    *   Select points with density $> 10$.
    *   Use these points as input for global registration (e.g., FPFH feature extraction + RANSAC) to obtain an initial alignment.
4.  **Virtual Camera Generation**:
    *   Sample from the 6D Gaussian in the latent space to generate high-visibility oriented points $(\tilde{x}, \tilde{d})$.
    *   Use the invertible Normalizing Flows model to recover the oriented point.
    *   Determine the position and orientation of a novel virtual camera using the recovered point and direction.
5.  **Photometric Optimization**:
    *   Render images from the Source NeRF using the generated virtual cameras.
    *   Optimize the 6-DoF rigid transformation parameters to minimize photometric loss between the rendered Source images and the Target NeRF.
    *   **Real Scenes**: 15K iterations, 32K samples/iteration, SGD optimizer, LR $5 \times 10^{-3}$, exponential decay.
    *   **Synthetic Scenes**: 2.5K iterations, 8128 samples/iteration, SGD optimizer, LR $1 \times 10^{-3}$, exponential decay.
    *   **Photometric Initialization**: Perform over 25 random transformations to find a good starting point.
    *   During optimization, use VF to select high-quality rays to improve convergence.

### Algorithm: Scene Representation via Vector Fields
This method renders indoor scenes using vector fields learned from posed images and depth [S2, unofficial].

1.  **Configuration**: Parse configuration for `vf_net_config`, `rendering_net_config`, `ray_sampler_config`, `density_config`, and `scheduler_config`.
2.  **Network Initialization**:
    *   Initialize `vector_field_network` and `rendering_network`.
    *   If `config.ray_sampler_config.fine_sampling()` is true, initialize `fine_vector_field_network` (defaulting to `self.vector_field_network`).
    *   Initialize `density` as `LaplaceDensity`.
    *   If `config.cos_sim_weights_anneal` is not 'none', initialize `LinearAnnealing` with shape `self.config.cos_sim_weights.shape[0]`, duration `self.config.anneal_end - self.config.anneal_start`, and `soft` mode based on `self.config.cos_sim_weights_anneal == 'soft'`.
3.  **Parallelism**: If `config.cuda_config.num_gpus > 1`, wrap networks in `nn.DataParallel`. Move all networks to `config.cuda_config.device`.
4.  **Ray Sampling**:
    *   Create `ray_sampler` (`UniformSampler`) with `n_samples`, `near`, `far`, and `perturb` (inverted).
    *   If fine sampling is enabled, create `fine_sampler` (`RangeFineSampler`) with `n_importance`, `near`, `far`, `perturb` (inverted), `range`, and `max_samples`.
5.  **Optimization**:
    *   Initialize optimizer as `torch.optim.Adam` with LR `config.scheduler_config.lr` and weight decay `config.scheduler_config.weight_decay`.
    *   Initialize scheduler as `torch.optim.lr_scheduler.ExponentialLR` with decay factor `self.config.scheduler_config.lr_decay_factor ** (1. / self.config.scheduler_config.lr_decay_steps)`.
    *   Train the network to minimize reconstruction loss against posed images and depth maps.

### Parameters
*   **VF Network**: Real-NVP, $L=4$ layers, $H=128$ hidden dims [S1].
*   **VF Optimizer**: RAdam, LR $5 \times 10^{-5}$ [S1].
*   **NeRF Representation**: Nerfacto [S1].
*   **NeRF Training**: 1024 rays/iteration, Adam optimizer, LR $10^{-2}$, exponential decay [S1].
*   **Density**: LaplaceDensity [S2, unofficial].
*   **Annealing**: LinearAnnealing for cosine similarity weights if configured [S2, unofficial].

### Edge Cases and Pitfalls
*   **Transparent Regions**: In synthetic scenes, samples where the image alpha is less than 128 are ignored during VF training [S1].
*   **Random Sampling Failure**: Random sampling on a unit sphere often produces non-informative images with noise; VF-NeRF avoids this by generating informative views [S1].
*   **Initialization**: Photometric optimization requires a good initial guess. VF-NeRF provides this via PC initialization (FPFH+RANSAC) or random transformation initialization [S1].

## Operating characteristics
*   **Time Complexity**: Depends on NeRF training time (60K/20K iterations) plus VF training time (last 10K/5K iterations) and registration optimization (15K/2.5K iterations).
*   **Memory Complexity**: Requires storing the NeRF weights, VF network weights, and latent space samples.
*   **Performance**:
    *   **LLFF Full Overlap (Trex)**: $\Delta t = 0.0205$, $\Delta R = 0.0122$ [S1].
    *   **LLFF No Overlap (Trex)**: $\Delta t = 0.0102$, $\Delta R = 0.0105$ [S1].
    *   **Objaverse**: Results vary by object. E.g., Chair 4b05: $\Delta R = 0.03$, $\Delta t = 2.43$; Piano 0e0d: $\Delta R = 6.89$, $\Delta t = 13.84$ [S1].
*   **Strengths**: Does not require user input (unlike NeRF2NeRF); generates informative views better than random sampling; robust to noise [S1].
*   **Limitations**: Requires pre-trained NeRFs; computationally intensive due to Normalizing Flows and iterative optimization.

## Background
VF-NeRF was introduced to solve the problem of rigid registration between two NeRFs without original camera positions. It builds on [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md) by using NeRF as the scene representation. The method was evaluated on LLFF and Objaverse datasets [S1]. The code implementation is available on GitHub [S2, unofficial].

## Sources
[S1] VF-NeRF: Viewshed Fields for Rigid NeRF Registration, https://doi.org/10.1007/978-3-031-73027-6_10, fetched 2026-10-26
[S2] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
