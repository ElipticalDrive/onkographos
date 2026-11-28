# Geometric Prior Guided Geometry Learning

Also called: NISR Pre-training with Geometric Priors, Volume Rendering Optimization with Normal Loss

## Overview
Geometric Prior Guided Geometry Learning is a two-stage method for high-fidelity 3D reconstruction that leverages geometric priors during both the pre-training of a neural implicit representation and the subsequent volume rendering optimization [S1, unofficial]. It addresses the lack of ground-truth surface normals in monocular settings by using a pretrained monocular normal estimator to supervise the optimization stage, while using ground-truth mesh normals during pre-training [S1, unofficial]. It builds on [Monocular Normal Regularization](13425-monocular-normal-regularization.md) by extending the use of external normal predictions to the entire optimization pipeline, rather than just as a regularizer on gradients.

## How it works
The method consists of two distinct phases: NISR (Neural Implicit Surface Reconstruction) pre-training and Volume Rendering Optimization.

### Inputs and Outputs
**Inputs:**
- Ground-truth mesh data (for pre-training).
- RGB images with camera poses (for optimization).
- A sparse feature volume structure (initialized during pre-training).
- A pretrained Omnidata model for monocular normal prediction [S1, unofficial].

**Outputs:**
- Optimized latent vectors for the sparse feature volume.
- Reconstructed 3D geometry (SDF and normals).

### Data Structures
The method operates on a **Sparse Feature Volume**.
- **Structure:** A 3D grid of sparse voxels.
- **Content:** Each voxel contains a feature vector $f_i \in \mathbb{R}^d$ (size $d$ not specified in sources) [S1, unofficial].
- **Sampling:** Points are sampled via ray-voxel intersection followed by hierarchical inside-voxel sampling, converting world-space distances to sparse space coordinates [S1, unofficial].

### Pre-training Stage
The goal is to learn the encoder-decoder parameters of the NISR using geometric priors.

1.  **Sampling:** Sample points inside sparse voxels using ray-voxel intersection and hierarchical inside-voxel sampling [S1, unofficial].
2.  **Normal Estimation:** Compute the gradient of the SDF decoder $\nabla f_\theta(x)$ at sampled points $x$ [S1, unofficial].
3.  **Loss Calculation:** Compute the pre-training total loss $L$ defined below.
4.  **Optimization:** Update NISR parameters to minimize $L$ [S1, unofficial].

**Formulas:**

**Pre-training Normal Loss ($L_n$):**
$$L_n = (1 - \langle \nabla f_\theta(x), \hat{n} \rangle) + \lambda_e ||\nabla f_\theta(x)^2 - 1||^2$$
Where:
- $\hat{n}$ is the normal sampled from the ground-truth mesh surface [S1, unofficial].
- $\nabla f_\theta(x)$ is the gradient of the SDF decoder [S1, unofficial].
- $\lambda_e$ is the weight for the Eikonal regularization term [S1, unofficial].

**Pre-training Total Loss ($L$):**
$$L = L_o + \lambda_s L_s + \lambda_n L_n + \lambda_f L_f$$
Where:
- $L_o = \sum_{l=1}^L \lambda_{lo} \text{BCE}(o_l, \hat{o}_l)$ is the binary cross-entropy loss for occupancy [S1, unofficial].
- $L_s = | \text{clamp}(s, \tau) - \text{clamp}(\hat{s}, \tau) |$ is the clipped L1 loss for SDF values [S1, unofficial].
- $L_f = \sum_{i=1}^M ||f_i||^2$ regularizes scene feature vectors $f_i \in \mathcal{F}_\theta$ [S1, unofficial].
- $\lambda_s, \lambda_n, \lambda_f, \lambda_{lo}$ are hyperparameters [S1, unofficial].
- $\tau$ is the clamping threshold [S1, unofficial].

**Clamping Function:**
$$\text{clamp}(x, \tau) = \frac{1}{2} \min(\tau, \max(-\tau, x))$$
[S1, unofficial]

### Volume Rendering Optimization Stage
The goal is to optimize the latent vectors of the sparse feature volume using rendered colors and externally predicted normals.

1.  **Normal Prediction:** For each input RGB image, predict the normal map $N(r)$ using the pretrained Omnidata model [S1, unofficial].
2.  **Rendering:** Render the color $\hat{C}(r)$ and normal $\hat{N}(r)$ from the current sparse feature volume for rays $r$ [S1, unofficial].
3.  **Loss Calculation:** Compute the optimization total loss $L$ defined below.
4.  **Optimization:** Update the latent vectors of the sparse feature volume to minimize $L$ [S1, unofficial].

**Formulas:**

**Optimization Total Loss ($L$):**
$$L = L_{rgb} + \lambda L_{normal}$$
Where:
- $L_{rgb} = \frac{1}{M} \sum |\hat{C}(r) - C(r)|$ is the color error between rendered color $\hat{C}(r)$ and predicted color $C(r)$ [S1, unofficial].
- $L_{normal} = \frac{1}{M} \sum \{ |\hat{N}(r) - N(r)| + (1 - \langle \hat{N}(r), N(r) \rangle) \}$ is the normal error [S1, unofficial].
- $M$ is the number of rendered pixels in a mini-batch [S1, unofficial].
- $\lambda$ is the weight for the normal loss [S1, unofficial].
- $N(r)$ is the normal predicted by the Omnidata model [S1, unofficial].
- $\hat{N}(r)$ is the rendered normal from the volume [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pretrain_nisr(mesh_data, n_samples, lambda_e, lambda_s, lambda_n, lambda_f, lambda_lo, tau):
    """
    Pre-train NISR parameters using ground-truth mesh normals.
    """
    for epoch in range(max_epochs):
        total_loss = 0.0
        for batch in mesh_data:
            # 1. Sample points inside sparse voxels
            points = sample_points_inside_voxels(batch.rays, batch.voxels)
            
            # 2. Get ground-truth normals at these points
            gt_normals = get_mesh_normals(points, batch.mesh)
            
            # 3. Compute SDF gradient
            sdf_values, sdf_grads = nisd_decoder(points)
            
            # 4. Compute Normal Loss
            # L_n = (1 - <grad, n_hat>) + lambda_e * ||grad^2 - 1||^2
            dot_product = sum(dot(g, n) for g, n in zip(sdf_grads, gt_normals))
            eikonal_term = sum((g.norm()**2 - 1)**2 for g in sdf_grads)
            L_n = (1 - dot_product / len(points)) + lambda_e * eikonal_term
            
            # 5. Compute other losses (simplified placeholders for structure)
            # L_o: Binary Cross Entropy for occupancy
            # L_s: Clipped L1 for SDF
            # L_f: L2 regularization for feature vectors
            L_o = compute_bce_loss(nisd_decoder.occupancy, batch.gt_occupancy)
            L_s = compute_clipped_l1_loss(nisd_decoder.sdf, batch.gt_sdf, tau)
            L_f = compute_l2_loss(nisd_decoder.features)
            
            # 6. Total Loss
            L = L_o + lambda_s * L_s + lambda_n * L_n + lambda_f * L_f
            
            # 7. Update NISR parameters
            nisd_decoder.backward(L)
            nisd_decoder.step()

def optimize_volume(rgb_images, poses, omnidata_model, lambda_normal, batch_size):
    """
    Optimize sparse feature volume latent vectors using RGB and predicted normals.
    """
    for epoch in range(max_epochs):
        total_loss = 0.0
        for batch in rgb_images:
            # 1. Predict normals from RGB images using Omnidata
            predicted_normals = []
            for img in batch.images:
                # Omnidata predicts normal map for the image
                n_map = omnidata_model.predict_normal(img)
                predicted_normals.append(n_map)
            
            # 2. Render from current volume
            # Render color C_hat and normal N_hat for rays defined by poses
            rendered_colors, rendered_normals = volume_renderer.render(batch.images, batch.poses, feature_volume)
            
            # 3. Compute Color Loss
            # L_rgb = (1/M) * sum(|C_hat - C|)
            color_error = sum(abs(r_c - p_c) for r_c, p_c in zip(rendered_colors, batch.images))
            L_rgb = color_error / batch_size
            
            # 4. Compute Normal Loss
            # L_normal = (1/M) * sum(|N_hat - N| + (1 - <N_hat, N>))
            normal_error = 0.0
            for r_n, p_n in zip(rendered_normals, predicted_normals):
                # r_n: rendered normal, p_n: predicted normal from Omnidata
                l1_dist = abs(r_n - p_n)
                cos_angle = 1 - dot(r_n, p_n)
                normal_error += l1_dist + cos_angle
            L_normal = normal_error / batch_size
            
            # 5. Total Loss
            L = L_rgb + lambda_normal * L_normal
            
            # 6. Update Feature Volume latent vectors
            feature_volume.backward(L)
            feature_volume.step()
```

### Parameters
- $\lambda_e$: Weight for Eikonal regularization in pre-training. Value not given in the sources.
- $\lambda_s$: Weight for clipped L1 loss in pre-training. Value not given in the sources.
- $\lambda_n$: Weight for normal loss in pre-training. Value not given in the sources.
- $\lambda_f$: Weight for feature vector regularization in pre-training. Value not given in the sources.
- $\lambda_{lo}$: Weight for BCE loss in pre-training. Value not given in the sources.
- $\tau$: Clamping threshold for SDF values. Value not given in the sources.
- $\lambda$: Weight for normal loss in optimization. Value not given in the sources.
- $M$: Number of rendered pixels in a mini-batch. Value not given in the sources.

### Edge Cases and Pitfalls
- **Normal Prediction Accuracy:** The optimization stage relies entirely on the quality of the Omnidata model's normal predictions. If the monocular estimator fails (e.g., on textureless surfaces or ambiguous geometry), the normal loss may introduce errors [S1, unofficial].
- **Precision:** The use of dot products and norms in loss calculations requires careful handling of floating-point precision, especially for the Eikonal term $||\nabla f_\theta(x)^2 - 1||^2$ [S1, unofficial].
- **Boundary Conditions:** The clamping function $\text{clamp}(x, \tau)$ handles SDF values near the surface, but the choice of $\tau$ affects the smoothness of the gradient [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Leverages strong geometric priors from both ground-truth meshes (pre-training) and monocular estimators (optimization) to improve surface detail [S1, unofficial].
- **Limitations:** Requires a pretrained Omnidata model for the optimization stage [S1, unofficial].
- **When to use:** High-fidelity reconstruction from monocular video where geometric detail is critical [S1, unofficial].
- **When not to use:** When computational resources are limited for running the Omnidata model or when ground-truth mesh data is unavailable for pre-training [S1, unofficial].

## Background
The method was proposed in "MonoNeuralFusion: Online Monocular Neural 3D Reconstruction with Geometric Priors" [S1, unofficial]. It builds on the idea of using geometric priors to regularize neural implicit representations, extending it to a two-stage process that combines pre-training with mesh data and optimization with monocular normal predictions [S1, unofficial].

## Sources
[S1] MonoNeuralFusion: Online Monocular Neural 3D Reconstruction with Geometric Priors, https://doi.org/10.48550/arxiv.2209.15153, fetched 2026-10-29 (unofficial)
