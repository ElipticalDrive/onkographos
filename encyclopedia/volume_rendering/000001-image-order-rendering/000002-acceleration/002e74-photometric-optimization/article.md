# Photometric Optimization

Also called: Albedo/Emission/Extinction optimization, EgoNeRF Training, Property reconstruction, Volume Optimization

## Overview
Photometric optimization is an inverse rendering technique that recovers latent scene parameters—such as 3D geometry, pose, texture, and lighting—by minimizing the photometric difference between a target image and a rendered image [S1][S2, S3]. It formulates the problem as an unconstrained optimization task, leveraging gradients estimated via differentiable rendering pipelines to update parameters using gradient descent [S1][S2, S3]. Unlike methods that rely on discrete candidate light sources or explicit shadow detection, this approach performs continuous optimization of scene properties, often utilizing regularizers to constrain the solution space [S1][S2, S3].

## How it works
The method minimizes a photometric loss function to update scene parameters [S1][S2]. The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [G], building on [Radiance-field optimization](44-radiance-field-optimization.md):

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance.

### Inputs and Outputs
*   **Inputs:** A set of target images (ground truth colors $C(\mathbf{r})$), initial scene parameters (e.g., voxel grid densities $\sigma$, albedo $\mathbf{a}$, emission $\mathbf{e}$, or neural network weights), and camera poses [S1][S2].
*   **Outputs:** Optimized scene parameters that minimize the difference between rendered and target images [S1][S2].

### Data Structures
*   **Voxel Grid:** Stores volume properties such as extinction, albedo, and emission [S1].
*   **Feature Grids:** In methods like EgoNeRF [S2], balanced spherical grids $G_\sigma, G_a$ store density and appearance features.
*   **Ray Batch:** A set of randomly sampled rays $\mathcal{R}$ for each optimization step [S2].

### Numbered Steps
1.  **Ray Sampling:** Sample a batch of rays $\mathcal{R}$ from the target images [S2].
2.  **Volume Rendering:** Compute the rendered color $\hat{C}(\mathbf{r})$ for each ray $\mathbf{r} \in \mathcal{R}$ using the current scene parameters and the volume rendering equation [S1][S2].
3.  **Loss Computation:** Calculate the photometric loss $\mathcal{L}$ between the rendered color $\hat{C}(\mathbf{r})$ and the ground-truth color $C(\mathbf{r})$ [S2]:
    $$
    \mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \|_2^2
    $$
4.  **Regularization (Optional):** Apply regularization terms to improve stability and reduce artifacts [S1][S2].
    *   **Laplacian Regularizer:** In [S1], a weak Laplacian regularizer is applied to introduce diffusion between neighboring voxels, reducing stripe and edge artifacts without smoothing intricate detail.
    *   **Total Variation (TV) Loss:** In [S2], TV loss is applied to feature grids to reduce noise, especially for real-world datasets with imperfect camera poses.
5.  **Gradient Computation:** Compute gradients of the total loss with respect to the scene parameters [S1][S2].
6.  **Parameter Update:** Update the scene parameters using an optimizer (e.g., Conjugate Gradient for [S1], or automatic differentiation/Adam for [S2]) [S1][S2].
7.  **Iteration:** Repeat steps 1–6 until convergence [S1].

### Formulas
*   **Photometric Loss:** $\mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \|_2^2$ [S2].
*   **Histogram Matching Loss (for cloud synthesis):** $\arg \min_{L_0, L_a} \sum_{n=1}^{n_L} [h(I_{ph}, n) - h(I(L_0, L_a), n)]^2$, where $n_L$ is the number of intensity levels, $h(I, n)$ is the normalized $n$-th histogram of image $I$, and $I(L_0, L_a)$ is the rendered image using parameters $L_0$ and $L_a$ [S4].

### Parameters
*   **Ray Marching Steps:** In [S1], twice the number of ray marching steps for the length of the diagonal as the number of voxels along an axis is a good choice.
*   **Convergence:** The process converges after three to five conjugate gradient steps [S1].
*   **Emission Constraint:** Emission can only lighten the appearance [S1].
*   **Albedo Constraint:** Albedo can only darken the appearance [S1].

### Edge Cases and Pitfalls
*   **Insufficient Views:** May induce stripe patterns into the volume because constraint pixels affect only voxels along their corresponding ray [S1].
*   **Edges Between Regions:** Edges can appear between constrained and unconstrained regions if only parts of the object are specified in target images [S1].
*   **Emission/Albedo Constraints:** If only emission is optimized, the result cannot fully match desired appearance as light is only added. If only albedo is optimized, light is only removed, preventing perfect matching [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Ray:
    def __init__(self, camera_id, origin, direction, u, v):
        self.camera_id = camera_id
        self.origin = origin
        self.direction = direction
        self.u = u
        self.v = v

def photometric_optimization(scene_params, target_images, camera_poses, num_iterations):
    """
    Optimizes scene parameters to match target images using photometric loss.
    """
    for iteration in range(num_iterations):
        # 1. Ray Sampling
        rays = sample_rays(target_images, camera_poses)
        
        # 2. Volume Rendering
        rendered_colors = volume_render(rays, scene_params)
        
        # 3. Loss Computation
        gt_colors = get_ground_truth_colors(rays, target_images)
        loss = mean_squared_error(rendered_colors, gt_colors)
        
        # 4. Regularization
        if scene_params.use_laplacian:
            loss += laplacian_regularization(scene_params)
        if scene_params.use_tv:
            loss += total_variation_loss(scene_params)
            
        # 5. Gradient Computation
        gradients = compute_gradients(loss, scene_params)
        
        # 6. Parameter Update
        scene_params = update_parameters(scene_params, gradients)
        
    return scene_params

def sample_rays(images, poses, batch_size=256):
    """Sample rays uniformly from the image plane for each pose."""
    rays = []
    for _ in range(batch_size):
        pose_idx = random.randint(0, len(poses) - 1)
        u, v = random.uniform(0, 1), random.uniform(0, 1)
        origin = poses[pose_idx].cam_to_world[:3, 3]
        direction = poses[pose_idx].cam_to_world[:3, :3] @ np.array([u - 0.5, v - 0.5, 1.0])
        rays.append(Ray(pose_idx, origin, direction, u, v))
    return rays

def volume_render(rays, params):
    """Computes color using alpha compositing along each ray."""
    colors = []
    for ray in rays:
        T = 1.0
        color = np.zeros(3)
        for step in range(params.num_steps):
            t = step * params.step_size
            pos = ray.origin + ray.direction * t
            sigma, c = params.query(pos)
            alpha = 1.0 - np.exp(-sigma * params.step_size)
            color += T * alpha * c
            T *= (1.0 - alpha)
        colors.append(color)
    return np.array(colors)

def get_ground_truth_colors(rays, images):
    """Retrieve ground truth colors for sampled pixels."""
    return np.array([images[ray.camera_id][ray.u, ray.v] for ray in rays])

def mean_squared_error(rendered, ground_truth):
    return np.mean((rendered - ground_truth) ** 2)

def laplacian_regularization(params):
    """Weak Laplacian regularizer for diffusion between neighboring voxels."""
    grid = params.grid
    diff_x = np.diff(grid, axis=0)
    diff_y = np.diff(grid, axis=1)
    diff_z = np.diff(grid, axis=2)
    return np.sum(diff_x ** 2) + np.sum(diff_y ** 2) + np.sum(diff_z ** 2)

def total_variation_loss(params):
    """Total variation loss to reduce noise in feature grids."""
    grid = params.grid
    diff_x = np.diff(grid, axis=0)
    diff_y = np.diff(grid, axis=1)
    diff_z = np.diff(grid, axis=2)
    return np.sum(np.abs(diff_x)) + np.sum(np.abs(diff_y)) + np.sum(np.abs(diff_z))

def compute_gradients(loss, params):
    """Computes gradients via automatic differentiation or conjugate gradient solver."""
    return params.compute_gradient(loss)

def update_parameters(params, gradients):
    """Updates parameters using conjugate gradient or Adam optimizer."""
    return params.apply_optimizer(gradients)
```

## Subtopics

### [Image-based reconstruction optimization](6677-image-based-reconstruction-optimization.md)
Image-based reconstruction optimization is an inverse rendering technique that recovers latent scene parameters—such as 3D geometry, pose, texture, and lighting—by minimizing the photometric difference between a target image and a rendered image. It formulates the problem as an unconstrained optimization task, leveraging gradients estimated via differentiable rendering pipelines to update parameters using gradient descent. Unlike methods that rely on discrete candidate light sources or explicit shadow detection, this approach performs continuous optimization of scene properties, often utilizing regularizers to constrain the solution space.

### [NeRFLiX](9385-nerflix.md)
NeRFLiX is a post-processing method for novel view synthesis that removes native rendering artifacts from Neural Radiance Fields (NeRFs) by learning degradation-driven multiview blending. It distinguishes itself from standard photometric optimization by explicitly modeling the degradation process via a NeRF-style degradation simulator and a multiview mixer, rather than solely optimizing scene parameters.

### [Joint Optimization of Scene and Renderer](11490-joint-optimization-of-scene-and-renderer.md)
Joint Optimization of Scene and Renderer is a photometric optimization method that simultaneously refines the geometric and appearance attributes of a scene representation alongside the parameters of a neural renderer. It solves the problem of reconstructing high-quality novel views from a sparse set of input images by treating per-pixel attributes (color, depth, normal, uncertainty, and latent features) as learnable parameters. It differs from [Image-based reconstruction optimization](6677-image-based-reconstruction-optimization.md) by explicitly optimizing the scene's underlying point-based representation attributes rather than just camera poses or static geometry, and it differs from [Gradient-Based SE(3) Optimization](7526-gradient-based-se-3-optimization.md) by focusing on per-view attribute refinement rather than global camera pose estimation.

### [Adaptive IOR Optimization](13856-adaptive-ior-optimization.md)
Adaptive IOR Optimization is a training procedure for neural radiance fields that iteratively searches for the optimal Index of Refraction (IOR) to minimize the reconstruction error of transparent foreground objects. It solves the problem of unknown or inaccurate IOR values in refractive scene reconstruction by decoupling background training from foreground IOR selection. It differs from sibling methods like [Image-based reconstruction optimization](6677-image-based-reconstruction-optimization.md) by explicitly treating IOR as a discrete search parameter rather than a continuous gradient-based variable, and from [Gradient-Based SE(3) Optimization](7526-gradient-based-se-3-optimization.md) by focusing on material properties rather than camera pose.

### Exploitation Stage Optimization Vectors
Optimization parameters used during the explicit mesh shading pass to refine geometry and scene elements (materials, lighting) [S3]. Removing the per-vertex deformation vector $\Delta v$ causes floating noisy points in the reconstructed mesh, which degrades lighting quality [S3]. Both $\Delta v$ and $\Delta m$ are crucial for high-quality reconstruction; ablation results show the full model significantly outperforms variants without $\Delta v$ or $\Delta m$ on both NeRF-synthetic and RefNeRF-synthetic datasets [S3].

### Histogram Matching for Parameter Optimization
A technique to determine parameters $L_0$ and $L_a$ by minimizing the difference between the histogram of the input photograph and the histogram of rendered example cloud volumes [S4]. Solves the nonlinear least square problem using a quasi-Newton method [S4]. Formula: $\arg \min_{L_0, L_a} \sum_{n=1}^{n_L} [h(I_{ph}, n) - h(I(L_0, L_a), n)]^2$, where $n_L$ is the number of intensity levels, $h(I, n)$ is the normalized $n$-th histogram of image $I$, and $I(L_0, L_a)$ is the rendered image using $L_0$ and $L_a$ [S4]. Optimized parameters are denoted $L^*_0, L^*_a$ [S4].

### Combined Light and Volume Optimization
Applies volume optimization on the result of light optimization to better match desired appearance using the same target images [S1]. Light optimization is theoretically redundant if emission is allowed, but useful when volume properties must not change or emission is to be avoided [S1]. Light optimization has a more global character as each light source affects the entire volume, connecting appearance to the rest of the scene [S1]. Used for rapid preview with a single view without introducing high-frequency artifacts from low input image counts [S1].

### Noise by example
Methods to find noise parameters from an example image, often by analyzing the Fourier transform or power spectrum [S5]. Approaches set parameters of existing procedural shaders from an example image to make the shader produce an image resembling the example [S5]. Steps include computing the Fourier transform of the example image, thresholding the magnitude to select sine waves, extending the 2D function to 3D, and refining for directional aspects [S5]. Other approaches identify main frequencies in 1D noise profiles, perform histogram matching, and extend to 2D/3D [S5]. Automatically compute weights of band-pass isotropic noise octaves to resemble an example image [S5]. Randomize the phase spectrum of a given texture to obtain homogeneous noise with the same power spectrum [S5]. Search for closest matches in a database of shader-generated images or optimize shader parameters using a genetic algorithm [S5].

### TokenBA: Alternating Optimization
Alternates between camera correction (improving rendering gradient consistency) and token refinement (improving geometry for subsequent camera updates) [S6, unofficial]. Concludes with a short joint photometric refinement of tokens and poses [S6, unofficial]. Uses up to four alternating rounds followed by at most 96 joint refinement steps with an L1 objective [S6, unofficial]. Adds approximately 1.6 s per scene at 16 views on an RTX 4090 [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** Scales with the number of voxels, ray marching steps, and number of views [S1]. For 128^3 voxels, 256 ray marching steps, and 460K constraint pixels (2 views), the result is computed in less than two seconds [S1]. With 2.3M constraint pixels (10 views), computation takes seven seconds [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Optimizing for 16 light coefficients on a 128^3 volume with 2.3M constraint pixels (10 views) takes around three seconds [S1].
*   **Scaling:** Volume optimization can reconstruct higher frequencies than light optimization because it modifies the volume locally [S1]. EgoNeRF does not need a coarse-to-fine reconstruction approach for robust optimization because the balanced feature grid guarantees a nearly uniform ray-grid hitting rate [S2].
*   **Strengths:** Can reconstruct high frequencies locally [S1].
*   **Limitations:** Insufficient views may induce stripe patterns. Edges can appear between constrained and unconstrained regions. Emission-only or albedo-only optimization cannot perfectly match desired appearance [S1].
*   **When to Use:** When high-frequency details need to be reconstructed, or when specific volume properties (like albedo or emission) need to be stylized or corrected [S1].
*   **When Not to Use:** When only global lighting changes are needed, or when volume properties must remain unchanged [S1].

## Background
Photometric optimization for volume rendering was introduced in the context of volume stylization and property manipulation [S1]. The method was proposed by [S1] for static volume stylization using a painting metaphor, allowing users to modify albedo, emission, and extinction to match target images. It has since been applied to various domains, including cloud synthesis [S4], egocentric view synthesis [S2], dynamic scene representation [S5], and inverse rendering [S3]. It is used for adding imprinted logos to smoke volumes, modifying volcano smoke appearance, and reconstructing extinction for refractive volumes [S1].

## Sources
[S1] Property and Lighting Manipulations for Static Volume Stylization Using a Painting Metaphor, https://doi.org/10.1109/TVCG.2014.13, fetched 2026-10-26
[S2] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
[S3] IRCasTRF: Inverse Rendering by Optimizing Cascaded Tensorial Radiance Fields, Lighting, and Materials From Multi-view Images, https://doi.org/10.1145/3581783.3612010, fetched 2026-10-27
[S4] Example-based synthesis of three-dimensional clouds from photographs, https://doi.org/10.1145/3095140.3095168, fetched 2026-10-29
[S5] A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S6] ZipSplat: Fewer Gaussians, Better Splats, https://doi.org/10.48550/arXiv.2606.05102, fetched 2026-10-29 (unofficial)
