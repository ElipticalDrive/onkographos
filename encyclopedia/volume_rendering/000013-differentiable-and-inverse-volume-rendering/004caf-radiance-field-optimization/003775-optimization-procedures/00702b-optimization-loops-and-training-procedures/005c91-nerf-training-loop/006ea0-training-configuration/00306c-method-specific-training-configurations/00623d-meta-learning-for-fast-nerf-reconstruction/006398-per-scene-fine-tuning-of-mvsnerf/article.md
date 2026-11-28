# Per-scene fine-tuning of MVSNeRF

Also called: per-scene optimization

## Overview
Per-scene fine-tuning is a post-processing optimization step applied to the neural encoding volume and MLP decoder predicted by MVSNeRF to achieve high-quality final renderings [S1]. It solves the problem of residual reconstruction errors and view-dependent shading artifacts that remain after the initial fast generalizable inference [S1]. Unlike from-scratch optimization methods like NeRF, which require hundreds of thousands of iterations, this method leverages a strong initial reconstruction to converge to comparable or superior quality in approximately 15 minutes [S1].

## How it works

### Inputs and Outputs
**Inputs:**
1. **Neural Encoding Volume ($V$):** A 3D grid of feature vectors predicted by the MVSNeRF 3D CNN for the reference view's frustrum [S1].
2. **MLP Decoder ($f_{\theta}$):** A multi-layer perceptron with weights $\theta$ predicted by the MVSNeRF network [S1].
3. **Reference Image ($I$):** The input image used to initialize the volume [S1].
4. **Target Views:** Camera poses and ground truth images for novel viewpoints [S1].

**Outputs:**
1. **Optimized Neural Encoding Volume ($V'$):** The updated 3D feature grid after fine-tuning [S1].
2. **Optimized MLP Weights ($\theta'$):** The updated decoder weights [S1].
3. **High-Quality Renderings:** Novel views synthesized using $V'$ and $\theta'$ [S1].

### Data Structures
**Neural Encoding Volume ($V$):**
* **Layout:** 3D grid of feature vectors covering the reference view's frustrum [S1].
* **Padding:** For large scenes, boundary voxels are padded to extend beyond the reference frustrum to prevent out-of-frustrum artifacts [S1].
* **Content:** Each voxel stores a feature vector $c$ [S1].

**MLP Decoder ($f_{\theta}$):**
* **Architecture:** Takes the concatenation of Positional Encoding ($PE$) of the 3D position and the 3D feature vector $c$ from the volume [S1].
* **Layers:** Consists of a positional encoding expansion layer ($PE0$), seven LinearRelu layers ($LR0$ to $LR7$), and a positional encoding layer ($PE1$) [S1].
* **Layer Definitions:** $LR$ denotes LinearRelu, $PE$ denotes Positional Encoding [S1]. Kernel size ($k$), stride ($s$), dilation ($d$), and input/output channels ($chns$) are specified in the source architecture table [S1].

### Numbered Steps
1. **Initialization:** Initialize the neural encoding volume $V$ and MLP decoder weights $\theta$ within the frustrum of the reference view using the MVSNeRF inference [S1].
2. **Padding (Optional):** For large scenes, pad the boundary voxels of $V$ to extend beyond the reference frustrum. This prevents out-of-frustrum artifacts in novel views, though padded voxels are not well-reconstructed initially and may require longer fine-tuning [S1].
3. **Ray Sampling:** For each pixel in the target view, cast a ray through the camera center and pixel coordinates [S1]. Do not use two-stage (coarse and fine) ray sampling [S1]. Uniformly sample points along each marching ray with per-step small randomness [S1].
4. **Points per Ray:** Use $N = 128$ points for standard scenes and $N = 256$ points for challenging scenes with high glossiness or specularities [S1].
5. **Volume Rendering:** For each sampled point $x_i$ on the ray, retrieve the 3D feature vector $c_i$ from $V$ via trilinear interpolation [S1]. Compute the positional encoding $PE(x_i)$ [S1]. Pass $[PE(x_i), c_i]$ through the MLP decoder $f_{\theta}$ to predict density $\sigma_i$ and color $c_i^{\text{color}}$ [S1]. Compute the rendered color $C(r)$ for ray $r$ using standard volume rendering integration:
   $$ C(r) = \sum_{i=1}^{N} T_i \alpha_i c_i^{\text{color}} $$
   where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, and $\delta_i$ is the distance between adjacent samples [S1].
6. **Loss Computation:** Compute the photometric loss between the rendered image and the ground truth target view image [S1].
7. **Optimization:** Perform gradient descent to update $V$ and $\theta$ [S1]. Run for 10,000 iterations on a single NVIDIA RTX 2080Ti GPU, taking approximately 15 minutes [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def per_scene_finetuning(volume, mlp_weights, reference_view, target_views, num_iterations=10000, points_per_ray=128):
    # volume: 3D tensor (D, H, W, C_feat)
    # mlp_weights: dict of tensors for LR0-LR7
    # target_views: list of dicts with 'intrinsics', 'extrinsics', 'target_image'
    # num_iterations: 10000 [S1]
    # points_per_ray: 128 or 256 [S1]

    # Helper: Trilinear interpolation [G]
    def trilinear_interpolate(vol, pt):
        return interpolate_3d(vol, pt)

    # Helper: Positional encoding [G]
    def positional_encoding(pt):
        return concat_sin_cos(pt, num_frequencies=6)

    # Helper: Ray casting [G]
    def cast_ray(K, extrinsics, u, v):
        return ray_origin, ray_direction

    # Helper: Volume rendering integration [G]
    def render_ray(points, volume, mlp_weights):
        alpha_acc = 0.0
        color_acc = zeros(3)
        for i in range(len(points)):
            pt = points[i]
            feat = trilinear_interpolate(volume, pt)
            pe = positional_encoding(pt)
            mlp_input = concatenate([pe, feat], axis=-1)
            sigma, color = mlp_forward(mlp_weights, mlp_input)
            delta = norm(points[i] - points[i-1]) if i > 0 else 0.01
            alpha = 1.0 - exp(-sigma * delta)
            weight = alpha * exp(-alpha_acc)
            color_acc += weight * color
            alpha_acc += alpha
        return color_acc

    # Optimizer state [G]
    mlp_grads = {k: zeros_like(w) for k, w in mlp_weights.items()}
    vol_grad = zeros_like(volume)
    lr = 0.001 # [G]

    for iteration in range(num_iterations):
        total_loss = 0.0
        for view in target_views:
            H, W = view['height'], view['width']
            rendered_image = zeros((H, W, 3))
            for y in range(H):
                for x in range(W):
                    ray_origin, ray_direction = cast_ray(view['intrinsics'], view['extrinsics'], x, y)
                    # Uniform sampling with per-step small randomness [S1]
                    t_vals = linspace(0.0, 1.0, points_per_ray) + random_uniform(points_per_ray) * 0.01
                    points = ray_origin + t_vals[:, newaxis] * ray_direction
                    rendered_image[y, x] = render_ray(points, volume, mlp_weights)

            loss = mean(abs(rendered_image - view['target_image'])) # [G]
            total_loss += loss

        # Parameter update [G]
        mlp_grads = compute_gradients(mlp_weights, volume, loss)
        mlp_weights = {k: w - lr * g for k, (w, g) in zip(mlp_weights.items(), mlp_grads.items())}
        volume = volume - lr * vol_grad

    return volume, mlp_weights
```

### Parameters
* **Training Iterations:** 10,000 [S1].
* **Points per Ray:** 128 for standard scenes, 256 for challenging scenes (high gloss/specularity) [S1].
* **Hardware:** Single NVIDIA RTX 2080Ti GPU [S1].
* **Training Duration:** Approximately 15 minutes [S1].

### Edge Cases and Pitfalls
1. **Out-of-Frustum Artifacts:** If the scene is large, parts visible in novel views may lie outside the reference view's frustrum. This causes artifacts because the volume is only initialized within the frustrum [S1].
   * **Solution:** Pad the boundary voxels of the neural encoding volume to extend beyond the frustrum [S1].
   * **Pitfall:** Padded voxels are not well-reconstructed initially and may require longer fine-tuning to achieve high quality [S1].
2. **Limited Viewing Range:** The representation is initialized only for content seen by the reference view. Unseen parts (occluded or outside frustum) are not directly recovered, making 360-degree rendering challenging with a single volume [S1].
   * **Solution:** Combine multiple neural encoding volumes at multiple views for larger viewing ranges [S1].
3. **View-Dependent Shading:** Highly challenging scenes with high glossiness or specularities are hard to recover via network inference alone [S1].
   * **Solution:** Use more points per ray (256) and potentially longer fine-tuning [S1].

## Operating characteristics
* **Time Complexity:** $O(I \cdot N \cdot P)$, where $I$ is the number of iterations (10,000), $N$ is the number of points per ray (128-256), and $P$ is the number of pixels [S1].
* **Memory Complexity:** $O(V \cdot C + M)$, where $V$ is the volume size, $C$ is the feature dimension, and $M$ is the MLP size [S1].
* **Measured Performance:** On the DTU and LLFF datasets, MVSNeRF's 15-minute fine-tuning (10k iterations) achieves quality comparable to or better than NeRF's 10.2-hour optimization (200k iterations) [S1].
* **Scaling:** Performance scales with the number of iterations. The 12k-iteration result (18 min) is already better than NeRF's 48k-iteration result (10.2h) [S1].
* **Strengths:** Significantly faster than from-scratch optimization (15 min vs 10.2 hours) [S1]. High quality results quickly [S1].
* **Limitations:** Limited viewing range with a single volume [S1]. Struggles with highly specular scenes without extended fine-tuning [S1]. Padding voxels require longer convergence [S1].
* **When to Use:** When high-quality novel view synthesis is required for a specific scene and a generalizable initial reconstruction is available. Ideal for scenarios where training time is constrained but quality is critical [S1].
* **When Not to Use:** When 360-degree rendering is required and multiple volumes cannot be combined. When scenes have extreme specularities and very short fine-tuning time is available [S1].

## Background
* **Origin:** MVSNeRF was introduced by Sun et al. in 2021 [S1].
* **Development:** The method builds on the idea of generalizable radiance field reconstruction, using a CNN to predict a neural encoding volume and an MLP decoder [S1]. The per-scene fine-tuning step was introduced to refine the initial prediction, addressing the limitations of pure inference-based reconstruction [S1].
* **Usage:** Used in view synthesis tasks on datasets like DTU and LLFF [S1].

## Sources
[S1] MVSNeRF: Fast Generalizable Radiance Field Reconstruction from Multi-View Stereo, https://doi.org/10.1109/iccv48922.2021.01386, fetched 2026-10-26
