# QuickSplat


## Overview
QuickSplat is a method for fast 3D surface reconstruction of large indoor scenes from multi-view images that combines three learned data prior networks (initializer, densifier, optimizer) in an iterative densification-optimization loop to accelerate convergence and improve geometry in under-observed regions [S1]. It differs from standard 3D Gaussian Splatting (3DGS) and its variants like [2DGS](8442-2d-gaussian-splatting.md) or [SuGaR](8442-2d-gaussian-splatting.md) by replacing heuristic-based densification and gradient descent with learned networks that jointly predict new Gaussians and update vectors, thereby removing heuristics and handling textureless regions better [S1]. Unlike methods relying solely on photometric errors, QuickSplat leverages learned geometry priors to reconstruct flat wall structures and object details in large-scale indoor environments [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Posed multi-view images and a Structure-from-Motion (SfM) point cloud [S1].
*   **Output:** A set of 2D Gaussian primitives $G$ representing the scene, from which a mesh surface is extracted via TSDF fusion of rendered depth maps [S1].

**Data Structures:**
The scene is discretized into a sparse set of 3D voxels of size $v_d$ [S1]. Each voxel is assigned a latent feature vector $f \in \mathbb{R}^{64}$ [S1].
*   **Voxel Grid:** Sparse 3D grid [S1].
*   **Latent Feature:** $\mathbb{R}^{64}$ per voxel [S1].
*   **Gaussian Primitives:** Decoded from voxel features using a small MLP, with $v_g$ Gaussians predicted per voxel [S1].

**Parameters:**
*   $v_d = 4\text{cm}$: Voxel dimension size [S1].
*   $v_g = 2$: Number of Gaussians predicted per voxel [S1].
*   $s = 20\text{K}$: Scaling factor for the number of voxels added in importance sampling [S1].
*   $T = 5$: Number of timesteps for the densification-optimization loop [S1].
*   Learning rate: $10^{-4}$ [S1].
*   Gradient accumulation: Gradients $\nabla G$ are accumulated from 100 training images in each iteration [S1].

**Decoding Gaussian Attributes:**
Voxel latent features are decoded into Gaussian primitives using a small MLP. Different activation functions are used for each attribute [S1]:
1.  **Position ($g_c$):** Relative to voxel center $v_c$:
    $$g_c = v_c + R (2 \sigma(x) - 1)$$
    where $R=4v_d$ is the radius around the voxel, $\sigma$ is the sigmoid function, and $x$ is the MLP output for position [S1].
2.  **Opacity ($g_o$):** Defined as the occupancy output after the last upsampling layer of the network. This allows rendering losses to backpropagate to upsampling layers [S1].
3.  **Scale:** Converted using the softplus function [S1].
4.  **Color:** Converted using the sigmoid function [S1].
5.  **Rotation:** Normalized quaternion vector [S1].

**Network Architectures:**
1.  **Initializer Network ($\theta_I$):** Takes SfM points as input. Uses sparse 3D convolutions in an encoder-decoder architecture. Dense CNN blocks are used in the bottleneck followed by upsampling layers to increase voxel density. An occupancy head acts as a threshold to determine if a voxel should be allocated in the next higher resolution [S1].
2.  **Densifier Network ($\theta_D$):** Predicts additional voxel features in free space. Uses an encoder-decoder architecture similar to the initializer but with **no** dense blocks in the bottleneck to constrain growth to neighbors [S1]. Inputs include current Gaussians $G_t$, gradients $\nabla G_t$, and timestep $t$ [S1].
3.  **Optimizer Network ($\theta_O$):** Uses a sparse 3D UNet architecture. Inputs are latent voxel features $G_t$ and their gradients $\nabla G_t$. It predicts an update vector $\Delta G_t$ normalized to $[-1, 1]$ [S1].

**Algorithm Steps:**

1.  **Initialization:**
    *   Discretize scene into sparse 3D voxels of size $v_d$ [S1].
    *   Assign latent feature $f \in \mathbb{R}^{64}$ to each voxel [S1].
    *   Run Initializer Network $\theta_I$ on SfM points to predict initial voxel features [S1].
    *   Decode voxel features into initial Gaussians $G_0$ [S1].

2.  **Iterative Densification-Optimization Loop (for $t = 0$ to $T-1$):**
    *   **Render and Gradient:** Render current Gaussians $G_t$ using all training images. Compute rendering gradients $\nabla G_t$ accumulated from 100 training images [S1].
    *   **Densification:**
        *   Run Densifier Network $\theta_D$ with inputs $G_t, \nabla G_t, t$ to predict additional voxel features $\hat{G}_t$ [S1].
        *   **Importance Sampling:** Select a subset of new Gaussians for densification. Sample weighted by occupancy predictions of upsampled voxels (interpreted as opacity). The number of voxels $n(t)$ added in iteration $t$ is:
            $$n(t) = s / 2^t$$
            where $s=20\text{K}$ [S1].
        *   Let $\bar{G}_t = G_t \cup \hat{G}_t$ be the concatenated set of existing and novel voxel features [S1].
        *   Initialize the gradient of novel voxels to zeros to obtain $\nabla \bar{G}_t$ [S1].
    *   **Optimization:**
        *   Run Optimizer Network $\theta_O$ with inputs $\bar{G}_t, \nabla \bar{G}_t, t$ to predict update vector $\Delta \bar{G}_t$ [S1].
        *   Update voxel features: $G_{t+1} = \bar{G}_t + \Delta \bar{G}_t$ [S1].

3.  **Post-Training (Optional):**
    *   Run 2000 steps of standard gradient descent without adaptive density control to refine Gaussians [S1].

4.  **Surface Extraction:**
    *   Render depth maps from all training views using the final Gaussians [S1].
    *   Run TSDF fusion on these depth maps to extract the mesh surface [S1].

**Loss Functions:**
*   **Rendering Loss ($\mathcal{L}_c$):**
    $$\mathcal{L}_c(x) = 0.8 || c(x) - C ||_1 + 0.2 (1 - \text{SSIM}(c(x), C))$$
    where $c(x)$ is rendered color, $C$ is observed image color [S1].
*   **Alpha Blending:**
    $$c(x) = \sum_{i \in \mathcal{N}} c_i \alpha_i T_i, \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$$
    where $\alpha_i = o_i g_i^{2D}(u(x))$ is opacity times 2D Gaussian value at intersection $u(x)$ [S1].
*   **Normal Loss ($\mathcal{L}_n$):**
    $$\mathcal{L}_n = 1 - \mathbf{n}_g^T \mathbf{n}_m$$
    where $\mathbf{n}_g$ is per-Gaussian normal and $\mathbf{n}_m$ is mesh normal [S1].
*   **Initializer Total Loss:**
    $$L(\theta_I) = \mathcal{L}_c + \mathcal{L}_d + \mathcal{L}_{occ} + 0.01\mathcal{L}_n + 10\mathcal{L}_{dist}$$
    where $\mathcal{L}_d$ is depth loss, $\mathcal{L}_{occ}$ is occupancy loss (binary cross-entropy on voxel occupancy before upsampling), and $\mathcal{L}_{dist}$ is distortion loss [S1].
*   **Optimizer Total Loss:**
    $$L(\theta_O) = \mathcal{L}_c + \mathcal{L}_d + 10\mathcal{L}_{dist}$$
    Calculated on updated features $G_{t+1}$. Gradients are detached for subsequent timesteps [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants from evidence
V_D = 0.04      # Voxel size in meters (4cm)
V_G = 2         # Gaussians per voxel
S = 20000       # Scaling factor for importance sampling
T = 5           # Number of timesteps
LR = 1e-4       # Learning rate

def discretize_scene(sfm_points, voxel_size):
    """Maps SfM points to a sparse 3D voxel grid."""
    voxels = {}
    for p in sfm_points:
        idx = tuple(int(floor(coord / voxel_size)) for coord in p)
        if idx not in voxels:
            voxels[idx] = {'center': [coord * voxel_size for coord in idx], 'feature': [0.0] * 64}
    return voxels

def assign_latent_features(voxels):
    """Initializes latent features for each voxel."""
    for v in voxels.values():
        v['feature'] = [0.0] * 64
    return voxels

def sort_by_depth(gaussians, rays):
    """Sorts Gaussians by their intersection depth along each ray."""
    sorted_list = []
    for ray in rays:
        ray_gaussians = [g for g in gaussians if intersects(g, ray)]
        ray_gaussians.sort(key=lambda g: g.depth_along(ray))
        sorted_list.extend(ray_gaussians)
    return sorted_list

def initializer_network(voxel_features):
    """Encoder-decoder with sparse 3D convs, dense bottleneck, upsampling, and occupancy head."""
    features = voxel_features
    for layer in encoder_layers:
        features = sparse_conv(features, layer)
    features = dense_bottleneck(features)
    for layer in decoder_layers:
        features = upsample_conv(features, layer)
    return features

def densifier_network(voxel_features, gradients, timestep):
    """Predicts additional voxel features using encoder-decoder without dense bottleneck."""
    features = voxel_features
    for layer in encoder_layers:
        features = sparse_conv(features, layer)
    # No dense bottleneck to constrain growth to neighbors
    for layer in decoder_layers:
        features = upsample_conv(features, layer)
    return features

def optimizer_network(voxel_features, gradients, timestep):
    """Sparse 3D UNet predicting update vector."""
    update = sparse_unet(voxel_features, gradients, timestep)
    return clamp(update, -1.0, 1.0)

def importance_sample(voxel_features, n_t):
    """Selects top n_t voxels weighted by occupancy (opacity)."""
    scores = [v['occupancy'] for v in voxel_features.values()]
    indices = argsort(scores, descending=True)[:n_t]
    return {k: v for i, k in enumerate(voxel_features.keys()) if i in indices}

def render_depth_maps(gaussians, images):
    """Renders depth maps from all training views."""
    depth_maps = []
    for img in images:
        depth = render_depth(gaussians, img.camera)
        depth_maps.append(depth)
    return depth_maps

def tsdf_fusion(depth_maps):
    """Fuses depth maps into a mesh using TSDF."""
    return mesh_from_tsdf(depth_maps)

def quicksplat_optimize(sfm_points, images):
    # 1. Initialization
    voxels = discretize_scene(sfm_points, V_D)
    voxels = assign_latent_features(voxels)
    voxels = initializer_network(voxels)

    # 2. Iterative Densification-Optimization Loop
    for t in range(T):
        gaussians = decode_gaussians(voxels, V_D, V_G)

        # Render and compute gradients accumulated over 100 images
        gradients = accumulate_gradients(gaussians, images[:100])

        # Densification
        new_voxels = densifier_network(voxels, gradients, t)
        n_t = S / (2 ** t)
        selected_new = importance_sample(new_voxels, n_t)
        combined_voxels = merge(voxels, selected_new)
        combined_gradients = merge(gradients, zeros_like(selected_new))

        # Optimization
        update = optimizer_network(combined_voxels, combined_gradients, t)
        voxels = merge(combined_voxels, update)

        # Detach gradients for subsequent timesteps
        detach_gradients()

    # 3. Post-Training (Optional)
    # Run 2000 steps of gradient descent without adaptive density control

    # 4. Surface Extraction
    depth_maps = render_depth_maps(gaussians, images)
    mesh = tsdf_fusion(depth_maps)
    return mesh
```

## Operating characteristics
*   **Time Complexity:** QuickSplat accelerates optimization runtime by 8x compared to baselines (SuGaR, 2DGS, PGSR) [S1]. On ScanNet++ test scenes, runtime is 26s (w/o opt) and 124s (w/ opt), compared to >10h for MonoSDF and 973s-3130s for baselines [S1].
*   **Memory Complexity:** The method uses a sparse voxel grid. The total number of parameters in the networks is ~68M [S1]. Importance sampling controls the number of new voxels added per iteration, with $n(t) = s/2^t$ and $s=20\text{K}$, reducing memory usage over time [S1].
*   **Measured Performance:** On ScanNet++ dataset, QuickSplat achieves a depth error reduction of 48% compared to state-of-the-art methods [S1]. Accuracy (Acc 2cm) is 0.5783 (w/ opt) vs 0.5263 (w/o opt) vs 0.4021 (2DGS) vs 0.4795 (GS2Mesh) vs 0.6027 (PGSR) vs 0.4028 (SuGaR) vs 0.5774 (MonoSDF) [S1]. Chamfer distance is 0.1347 (w/ opt) [S1].
*   **Strengths:** Fast convergence, handles textureless regions better than photometric-only methods, preserves fine details better than monocular methods like MonoSDF, avoids domain gap artifacts of GS2Mesh [S1].
*   **Limitations:** Struggles with mirror reflections (photometric loss encourages reconstructing reflected geometry behind mirrors, leading to noisy artifacts) [S1]. Assumes static environments (cannot reconstruct dynamic scenes) [S1]. Does not reconstruct in real-time [S1]. Holes in surface reconstruction may not be filled by the optimizer alone because it only moves Gaussians around voxel centers and never allocates new voxels [S1]. Rendering loss does not backpropagate to empty voxels, providing no signal for densification in empty regions [S1].

## Background
QuickSplat was introduced by Yueh-Cheng Liu, Lukas Höllein, Matthias Nießner, and Angela Dai from the Technical University of Munich. The paper "QuickSplat: Fast 3D Surface Reconstruction via Learned Gaussian Initialization" was published on arXiv (arXiv:2505.05591v2) on 10 Aug 2025 [S1]. It builds on 2D Gaussian Splatting (2DGS) [26] and integrates learned priors from data, similar to approaches like SGNN [15] and G3R [11], but applies them to Gaussian primitives for surface reconstruction. It is evaluated on the ScanNet++ dataset [55], which contains 902 indoor scenes.

## Sources
[S1] QuickSplat: Fast 3D Surface Reconstruction via Learned Gaussian Initialization, https://doi.org/10.1109/ICCV51701.2025.02585, fetched 2026-10-29
