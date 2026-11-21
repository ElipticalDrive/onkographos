# SUDS

Also called: SUDS, Scalable Urban Dynamic Scenes

## Overview
SUDS is a 4D radiance field representation designed for large-scale dynamic urban scenes, enabling free-viewpoint rendering, semantic decomposition, and 3D scene flow estimation across thousands of videos and millions of frames [S1]. It distinguishes itself from prior methods by factorizing the scene into three separate multi-resolution hash table branches (static, dynamic, and far-field) and utilizing unlabeled supervision signals (RGB, sparse LiDAR, optical flow, and DINO descriptors) rather than requiring expensive 3D bounding box or panoptic segmentation annotations [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Ordered RGB images from $N$ videos taken at different times/weather conditions, associated camera poses, aligned sparse LiDAR depth measurements, 2D self-supervised DINO descriptors, and 2D optical flow predictions [S1].
*   **Outputs:** Rendered RGB images, semantic feature maps, 3D scene flow vectors, and density fields for arbitrary poses and time steps [S1].

**Data Structures**
The scene is decomposed into $K$ spatial cells, each trained independently with its own set of hash tables [S1]. Each cell contains three branches:
1.  **Static Branch:** Stores 3D spatial voxels $v_{l,s}$ at $L$ resolution levels.
2.  **Dynamic Branch:** Stores 4D spacetime voxels $v_{l,d}$ at $L$ resolution levels.
3.  **Far-field Branch:** Stores 3D direction-based voxels $v_{l,e}$ at $L$ resolution levels.

Each voxel corner in the hash tables is associated with a feature vector of dimension $F$ [S1]. The hash tables use specific hash functions to map voxel coordinates to indices, handling video IDs to separate interpolating functions for different videos without linear memory growth [S1].

**Numbered Steps**

1.  **Spatial Partitioning:** Split the global scene into $K$ spatial cells with centroids $k \in \mathbb{R}^3$. For each cell, generate a training dataset by including only data points whose camera rays intersect the cell. Prune irrelevant rays using LiDAR depth measurements to ensure visibility [S1].

2.  **Hash Indexing:** For an input coordinate $(x, d, t, \text{vid})$ where $x$ is position, $d$ is viewing direction, $t$ is frame index, and $\text{vid}$ is video ID, compute hash indices for the corners of surrounding voxels at each level $l \in \{1, \dots, L\}$:
    *   Static: $i_{l,s} = \text{static\_hash}(\text{space}(v_{l,s}))$
    *   Dynamic: $i_{l,d} = \text{dynamic\_hash}(\text{space}(v_{l,d}), \text{time}(v_{l,d}), \text{vid})$
    *   Far-field: $i_{l,e} = \text{env\_hash}(\text{dir}(v_{l,e}), \text{vid})$
    Hash collisions are handled via gradient averaging [S1].

3.  **Feature Interpolation:**
    *   Look up feature vectors corresponding to the hashed corners.
    *   Interpolate features based on the relative position of the input coordinate within the voxel. Use linear interpolation for the static branch and quadlinear interpolation for the dynamic 4D branch [S1].
    *   Concatenate the interpolated features from all $L$ levels into a feature vector $f \in \mathbb{R}^{LF}$ [S1].

4.  **Branch Evaluation (MLP):** Pass the concatenated feature vector $f$ and auxiliary inputs into separate MLPs for each branch to obtain outputs:
    *   **Static Branch:** View-dependent color $c_s(x, d, A_{\text{vid}} F(t)) \in \mathbb{R}^3$ and view-independent density $\sigma_s(x) \in \mathbb{R}$. Lighting variations are modeled by conditioning on a latent embedding $A_{\text{vid}} F(t)$, where $A_{\text{vid}}$ is a video-specific matrix and $F(t)$ is a Fourier-encoded time index [S1].
    *   **Dynamic Branch:** Time/video-dependent density $\sigma_d(x, t, \text{vid}) \in \mathbb{R}$, color $c_d(x, t, \text{vid}, d) \in \mathbb{R}^3$, shadow ratio $\rho_d(x, t, \text{vid}) \in [0, 1]$, and 3D scene flow vectors $s_{t0}(x, t, \text{vid}) \in [-1, 1]^3$ for forward/backward flow [S1].
    *   **Far-field Branch:** Environment map color $c_e(d, \text{vid}) \in \mathbb{R}^3$ [S1].
    *   **Semantic Heads:** Each branch also outputs $C$-dimensional semantic feature logits $\Phi_s(x)$, $\Phi_d(x, t, \text{vid})$, and $\Phi_e(d, \text{vid})$ to distill DINO features [S1].

5.  **Density and Radiance Combination:**
    *   Compute total density: $\sigma(x, t, \text{vid}) = \sigma_s(x) + \sigma_d(x, t, \text{vid})$ [S1].
    *   Compute total radiance:
        $$ c(x, t, \text{vid}, d) = \frac{(1 - \rho_d)\sigma_s c_s(x, d, A_{\text{vid}} F(t)) + \sigma_d c_d(x, t, \text{vid}, d)}{\sigma} $$
        The shadow ratio $\rho_d$ scales down the static color to model shadows [S1].

6.  **Rendering:** For a camera ray $r$ at frame $t$ and video $\text{vid}$, render color $\hat{C}$ and features $\hat{F}$ by accumulating transmittance along sampled points $r(t)$:
    $$ \hat{C}(r, t, \text{vid}) = \int_0^{+\infty} T(t) \sigma(r(t), t, \text{vid}) c(r(t), t, \text{vid}, d) dt + T(+\infty) c_e(d, \text{vid}) $$
    where $T(t) = \exp\left(-\int_0^t \sigma(r(s), t, \text{vid}) ds\right)$ [S1].

7.  **Loss Calculation:** Minimize the following losses:
    *   **Reconstruction:** L2 photometric loss $L_c(r) = \|C(r) - \hat{C}(r)\|^2$ and L1 feature loss $L_f(r) = \|F(r) - \hat{F}(r)\|_1$ [S1].
    *   **Depth:** L2 loss between rendered depth $\hat{D}(r)$ and LiDAR depth $D(r)$: $L_d(r) = \|D(r) - \hat{D}(r)\|^2$ [S1].
    *   **Flow:** Predict 3D position $\hat{X}_{t0}(r)$ using scene flow, render to 2D, and minimize distance from observed optical flow $O$: $L_o(r) = \sum_{t0} \|\hat{X}(o) - \hat{X}_{t0}(r)\|_1$ [S1].
    *   **Warping:** Advect samples using predicted flow to warp density, color, and features. Render warped outputs and minimize L2/L1 losses against current frame inputs, downweighted by ambiguity weight $W_{t0}(r)$: $L_c^w(r) = \sum_{t0} (1 - W_{t0})(r) \|C(r) - \hat{C}_{t0}^w(r)\|^2$ [S1].
    *   **Flow Regularization:** Cycle consistency $L_{cyc}(r)$, spatial/temporal smoothness $L_{sm}$, and static scene prior $L_{slo}$ [S1].
    *   **Static-Dynamic Factorization:** Skewed binary entropy loss $L_e(r)$ with $k=1.75$ and max dynamic ratio penalty $L_d(r)$ [S1].
    *   **Shadow Loss:** Penalize squared magnitude of shadow ratio $L_\rho(r)$ [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def static_hash(space_coords):
    """Hash function for static branch spatial coordinates."""
    # Implementation details of the specific hash function are not provided in the source.
    # Placeholder for the hashing logic described in [S1].
    return hash(tuple(space_coords))

def dynamic_hash(space_coords, time_val, vid):
    """Hash function for dynamic branch spacetime coordinates."""
    # Implementation details of the specific hash function are not provided in the source.
    # Placeholder for the hashing logic described in [S1].
    return hash(tuple(space_coords) + (time_val, vid))

def env_hash(dir_vec, vid):
    """Hash function for far-field branch direction coordinates."""
    # Implementation details of the specific hash function are not provided in the source.
    # Placeholder for the hashing logic described in [S1].
    return hash(tuple(dir_vec) + (vid,))

def quadlinear_interpolation(features, relative_pos):
    """
    Interpolate features from 8 voxel corners using quadlinear interpolation.
    features: list of 8 feature vectors (F-dim)
    relative_pos: 4D vector (x, y, z, t) in [0, 1]
    """
    # Standard quadlinear interpolation logic
    # ...
    return interpolated_feature

def linear_interpolation(features, relative_pos):
    """
    Interpolate features from 8 voxel corners using linear interpolation.
    features: list of 8 feature vectors (F-dim)
    relative_pos: 3D vector (x, y, z) in [0, 1]
    """
    # Standard trilinear interpolation logic
    # ...
    return interpolated_feature

def fourier_encode_time(t, num_freq_bands=10):
    """Fourier encoding of time index."""
    frequencies = 2.0 ** np.linspace(0, num_freq_bands - 1, num_freq_bands)
    scaled_t = t * frequencies * 2 * np.pi
    return np.concatenate([np.sin(scaled_t), np.cos(scaled_t)])

def train_suds(videos, lidar_depths, optical_flows, dino_features, camera_poses):
    """
    Main training loop for SUDS.
    """
    # Hyperparameters
    L = 16  # Number of resolution levels (example value, not explicitly given as fixed constant in notes, but implied hierarchy)
    F = 2   # Feature dimensionality (example value, not explicitly given)
    C = 256 # Semantic feature dimension (example value, not explicitly given)
    num_iterations = 250000
    batch_size = 4098
    lr_init = 5e-3
    lr_final = 5e-4
    
    # Initialize hash tables and MLPs for each spatial cell
    # ...
    
    for iteration in range(num_iterations):
        # Sample batch of rays
        rays = sample_rays(batch_size)
        
        total_loss = 0
        
        for ray in rays:
            x, d, t, vid = ray.position, ray.direction, ray.time, ray.video_id
            
            # 1. Hash Indexing
            indices_s = []
            indices_d = []
            indices_e = []
            
            for l in range(L):
                # Get voxel corners for level l
                voxels_s = get_voxels_static(x, l)
                voxels_d = get_voxels_dynamic(x, t, l)
                voxels_e = get_voxels_env(d, l)
                
                # Compute hash indices
                for v_s in voxels_s:
                    indices_s.append(static_hash(v_s.coords))
                for v_d in voxels_d:
                    indices_d.append(dynamic_hash(v_d.coords, v_d.time, vid))
                for v_e in voxels_e:
                    indices_e.append(env_hash(v_e.dir, vid))
            
            # 2. Feature Interpolation
            features_s = []
            features_d = []
            features_e = []
            
            for idx in indices_s:
                features_s.append(lookup_hash_table(idx))
            for idx in indices_d:
                features_d.append(lookup_hash_table(idx))
            for idx in indices_e:
                features_e.append(lookup_hash_table(idx))
                
            # Interpolate (simplified for pseudocode)
            f_s = linear_interpolation(features_s, get_relative_pos(x))
            f_d = quadlinear_interpolation(features_d, get_relative_pos_4d(x, t))
            f_e = linear_interpolation(features_e, get_relative_pos(d))
            
            # Concatenate levels
            f_s_concat = concatenate(f_s)
            f_d_concat = concatenate(f_d)
            f_e_concat = concatenate(f_e)
            
            # 3. Branch Evaluation
            # Static
            time_enc = fourier_encode_time(t)
            lighting_cond = A_vid @ time_enc # A_vid is video-specific matrix
            c_s, sigma_s, phi_s = static_mlp(f_s_concat, d, lighting_cond)
            
            # Dynamic
            sigma_d, c_d, rho_d, phi_d, s_flow = dynamic_mlp(f_d_concat, t, vid)
            
            # Far-field
            c_e, phi_e = env_mlp(f_e_concat, d, vid)
            
            # 4. Combination
            sigma_total = sigma_s + sigma_d
            c_total = ((1 - rho_d) * sigma_s * c_s + sigma_d * c_d) / sigma_total
            phi_total = (sigma_s * phi_s + sigma_d * phi_d) / sigma_total # Simplified blending
            
            # 5. Rendering
            T = compute_transmittance(sigma_total)
            C_rendered = integrate(T, sigma_total, c_total) + T_final * c_e
            F_rendered = integrate(T, sigma_total, phi_total)
            
            # 6. Loss Calculation
            loss_c = l2_loss(ray.gt_color, C_rendered)
            loss_f = l1_loss(ray.gt_features, F_rendered)
            loss_d = l2_loss(ray.lidar_depth, render_depth(sigma_total))
            
            # Flow losses
            X_pred = integrate_flow(T, sigma_total, s_flow)
            loss_o = l1_loss(ray.optical_flow, project_2d(X_pred))
            
            # Warping losses
            sigma_w, c_w, phi_w = advect_samples(sigma_total, c_total, phi_total, s_flow)
            C_warped = integrate(T, sigma_w, c_w)
            F_warped = integrate(T, sigma_w, phi_w)
            loss_w_c = l2_loss(ray.gt_color, C_warped)
            loss_w_f = l1_loss(ray.gt_features, F_warped)
            
            # Regularization
            loss_cyc = cycle_consistency_loss(s_flow)
            loss_sm = smoothness_loss(s_flow)
            loss_slo = static_prior_loss(s_flow)
            loss_e = skewed_entropy_loss(sigma_s, sigma_d, k=1.75)
            loss_rho = shadow_penalty_loss(rho_d)
            
            total_loss += (loss_c + loss_f + loss_d + loss_o + loss_w_c + loss_w_f + 
                           loss_cyc + loss_sm + loss_slo + loss_e + loss_rho)
        
        # Update parameters
        update_parameters(total_loss, lr=lr_init)
```

**Parameters**
*   $F$: Dimensionality of feature vectors in the multi-resolution hash tables [S1].
*   $L$: Number of resolution levels in the hash table hierarchy, where resolution doubles between levels [S1].
*   $N$: Number of input videos [S1].
*   $\text{vid}$: Video ID identifier used as an auxiliary input to the hash function [S1].
*   $A_{\text{vid}}$: Video-specific matrix used to compute a latent embedding for lighting variations [S1].
*   $F(t)$: Fourier-encoded time index [S1].
*   $k$: Constant set to 1.75 in the skewed binary entropy loss [S1].
*   $C$: Dimensionality of the feature grid output from the DINO teacher model [S1].
*   Training iterations: 250,000 [S1].
*   Rays per batch: 4,098 [S1].
*   Learning rate: Decays from $5 \times 10^{-3}$ to $5 \times 10^{-4}$ using Adam optimizer [S1].

**Edge Cases and Pitfalls**
*   **Hash Collisions:** Handled via gradient averaging [S1].
*   **Ambiguous Regions:** Warping losses are downweighted in regions with ambiguous geometry or occlusions using weight $W_{t0}(r)$ [S1].
*   **Shadow Over-explanation:** Penalized by the squared magnitude of the shadow ratio to prevent the model from using shadows to explain dark regions incorrectly [S1].
*   **Visibility:** Irrelevant rays are pruned using LiDAR depth to ensure only visible pixels are used for training each spatial cell [S1].

## Operating characteristics
*   **Scale:** SUDS scales to tens of thousands of objects across 1.2 million frames from 1,700 videos spanning geospatial footprints of hundreds of kilometers [S1].
*   **Training Speed:** SUDS is 10x quicker to train than state-of-the-art methods that rely on ground truth 3D bounding box annotations [S1].
*   **Memory:** Unlike object-centric methods like NSG which require over 1TB of memory for a 30-second video, SUDS uses multi-resolution hash tables to manage memory efficiently [S1].
*   **Strengths:** Handles ego-motion, multi-video representations, and large-scale dynamic scenes without requiring labeled 3D data (bounding boxes/segmentation) [S1].
*   **Limitations:** Relies on the quality of self-supervised features (DINO) and optical flow; performance may degrade in regions with poor texture or ambiguous motion [S1].

## Background
SUDS (Scalable Urban Dynamic Scenes) was introduced to address the limitations of prior dynamic scene representations that required supervised inputs like panoptic segmentation labels or bounding boxes, which are difficult to acquire for in-the-wild captures [S1]. It builds on the multi-resolution hash table concept from Instant-NGP [S1] and extends it to 4D dynamic scenes. The method was presented at CVPR 2023 [S1].

## Sources
[S1] SUDS: Scalable Urban Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.01191, fetched 2026-10-26
