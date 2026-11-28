# Bi-level Fusion

Also called: BNV-Fusion, BNVF, Local- and Global-level Fusion, local + global fusion

## Overview
Bi-level Fusion is an online 3D reconstruction method that represents geometry using implicit neural volumes, combining local-level fusion for efficient depth map integration and global-level fusion framed as neural rendering for consistent reconstruction [S1]. It solves the problem of accumulating errors and outliers in sequential reconstruction by first fusing latent codes via weighted averaging and then optimizing the global volume against depth measurements [S1]. It sets itself apart from its sibling [GRU Volume Fusion](7543-gru-volume-fusion.md) by using explicit weighted averaging of latent codes followed by an optimization step, rather than recurrent neural network updates [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of depth maps $D_t$, camera extrinsic parameters $T_{c_t}^w$, and the current global implicit volume $V_w^{t-1}$ with associated weight volume $W^{t-1}$ [S1].
*   **Outputs:** The updated global implicit volume $V_w^t$ and weight volume $W^t$ [S1].

**Data Structures:**
*   **Implicit Neural Volume ($V_w$):** A 3D grid of latent codes where the voxel resolution is 2 cm, meaning each latent code represents the geometry of a $2\text{cm}^3$ volume around its position [S1].
*   **Weight Volume ($W$):** A 3D grid of weights, where values are set to the number of 3D points associated with a voxel [S1].
*   **Decoder:** An MLP with 4 fully connected layers with sizes [128, 128, 128, 1] that maps a latent code and a 3D position to an SDF value [S1].

**Numbered Steps:**

1.  **Local-level Fusion:**
    1.  Transform the single-view neural volume $V_{c_t}^s$ from camera coordinates to world coordinates using the camera extrinsic parameters: $V_w^s = T_{c_t}^w V_{c_t}^s$ [S1].
    2.  Update the global implicit volume $V_w$ and weight volume $W$ using weighted averaging of the latent codes:
        $$
        V_w^t = \frac{W^{t-1} V_w^{t-1} + W_{c_t}^s V_w^s}{W^{t-1} + W_{c_t}^s}
        $$
        $$
        W^t = W^{t-1} + W_{c_t}^s
        $$
        where $V_w$ and $W$ denote the global implicit volume and its weight, respectively [S1]. The superscript $t-1$ and $t$ means prior update and post update [S1]. The superscript $s$ denotes single-view source [S1].

2.  **Global-level Fusion:**
    1.  Randomly sample a set of pixels in the depth map. The method samples 5000 pixels per image in each iteration [S1].
    2.  Along the ray unprojected from each sampled pixel, sample $N$ 3D points $\{x_0, x_1, ..., x_n\}$ using hierarchical sampling: more points are sampled closer to the measured surface [S1].
        *   Coarse-level sampling: 5 3D points per meter [S1].
        *   Fine-level sampling: 20 3D points per meter [S1].
    3.  Transform the 3D points to world coordinates [S1].
    4.  Extract SDF values $s_i$ from the global volume $V_w$ using trilinear interpolation of decoder outputs [S1].
    5.  Compute the projective TSDF $\bar{s}_i$ for each 3D point given a depth measurement $p$ on the ray:
        $$
        \bar{s}_i = \min(\max(\|p - x_i\|_2, -\delta), \delta)
        $$
        where $\delta$ is the truncation threshold [S1].
    6.  Optimize the global volume $V_w$ to minimize the discrepancy between decoded SDF values $s_i$ and projective TSDF values $\bar{s}_i$:
        $$
        \argmin_{V_w} \sum_{i=0}^{n} \|s_i - \bar{s}_i\|_1
        $$
        where $s_i$ is the SDF value decoded from the neural volume $V_w$ [S1].
    7.  The optimization uses the Adam optimizer for 5 iterations for each image [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def bi_level_fusion(V_w_prev, W_prev, depth_map, camera_extrinsic, depth_encoder, decoder, intrinsics, delta):
    # Step 1: Local-level Fusion
    # Encode depth map to single-view neural volume V_cst
    V_cst = depth_encoder(depth_map)
    # Transform single-view volume to world coordinates
    V_ws = transform_volume_coords(V_cst, camera_extrinsic)
    # Compute source weight volume (number of 3D points per voxel)
    W_cst = compute_weight_volume(V_ws)

    # Weighted averaging of latent codes
    V_w = (W_prev * V_w_prev + W_cst * V_ws) / (W_prev + W_cst)
    W = W_prev + W_cst

    # Step 2: Global-level Fusion
    sampled_pixels = random.sample_pixels(depth_map, num_pixels=5000)

    for iteration in range(5):
        total_loss = 0.0
        param_grads = {}

        for u, v in sampled_pixels:
            p = depth_map[v, u]
            ray_origin, ray_direction = unproject_pixel(u, v, p, intrinsics)

            # Hierarchical sampling: coarse 5 pts/m, fine 20 pts/m
            points_3d = sample_along_ray(ray_origin, ray_direction, p, coarse_density=5, fine_density=20)

            for point in points_3d:
                latent_code = get_latent_code(V_w, point)
                s_i = decoder(latent_code, point)

                # Projective TSDF
                dist = abs(p - sum(r * c for r, c in zip(ray_direction, point)))
                s_bar_i = min(max(dist, -delta), delta)

                total_loss += abs(s_i - s_bar_i)
                # Accumulate gradients for Adam optimizer [G]
                grad = compute_gradient(s_i, s_bar_i, decoder, V_w, point)
                accumulate_grads(param_grads, grad)

        # Update parameters using Adam [G]
        V_w = adam_step(V_w, param_grads, lr=0.001)

    return V_w, W

def transform_volume_coords(V, T_c_w):
    # Apply extrinsic matrix to voxel centers
    return V.apply_transform(T_c_w)

def compute_weight_volume(V_ws):
    # Count number of 3D points associated with each voxel
    return V_ws.count_points()

def random_sample_pixels(depth_map, num_pixels):
    h, w = depth_map.shape
    return [(random.randint(0, w-1), random.randint(0, h-1)) for _ in range(num_pixels)]

def unproject_pixel(u, v, depth, intrinsics):
    fx, fy = intrinsics['fx'], intrinsics['fy']
    cx, cy = intrinsics['cx'], intrinsics['cy']
    x = (u - cx) * depth / fx
    y = (v - cy) * depth / fy
    z = depth
    ray_origin = [0.0, 0.0, 0.0]
    ray_direction = [x, y, z]
    norm_val = math.sqrt(sum(c**2 for c in ray_direction))
    ray_direction = [c / norm_val for c in ray_direction]
    return ray_origin, ray_direction

def sample_along_ray(ray_origin, ray_direction, depth, coarse_density, fine_density):
    points = []
    max_dist = depth
    # Coarse sampling along the ray
    coarse_step = 1.0 / coarse_density
    for d in range(0, int(max_dist * coarse_density)):
        point = [ray_origin[i] + ray_direction[i] * d * coarse_step for i in range(3)]
        points.append(point)
    # Fine sampling near the measured surface
    fine_step = 1.0 / fine_density
    start_fine = max(0, int(max_dist * coarse_density) - 10)
    end_fine = int(max_dist * coarse_density) + 10
    for d in range(start_fine, end_fine):
        point = [ray_origin[i] + ray_direction[i] * d * fine_step for i in range(3)]
        points.append(point)
    return points

def get_latent_code(V_w, point):
    return V_w.interpolate(point)

def decoder(latent_code, point):
    x = latent_code + point
    for size in [128, 128, 128]:
        x = math.tanh(sum(a * b for a, b in zip(x, [1.0]*len(x)))) # Simplified linear+activation [G]
    return x[-1] # Final layer outputs scalar SDF

def compute_gradient(s_i, s_bar_i, decoder, V_w, point):
    # Standard backpropagation gradient computation [G]
    return (s_i - s_bar_i) * 1.0

def accumulate_grads(grads, grad):
    for k in grad:
        grads[k] = grads.get(k, 0.0) + grad[k]

def adam_step(params, grads, lr, beta1=0.9, beta2=0.999, eps=1e-8):
    # Simplified Adam update step [G]
    return {k: params[k] - lr * grads[k] for k in params}
```

**Parameters:**
*   **Voxel Resolution:** 2 cm [S1].
*   **Latent Code Dimension:** 8 [S1].
*   **Decoder Architecture:** MLP with 4 fully connected layers with sizes [128, 128, 128, 1] [S1].
*   **Sampling Density:** Coarse-level: 5 3D points per meter; Fine-level: 20 3D points per meter [S1].
*   **Pixels Sampled:** 5000 pixels per image in each iteration [S1].
*   **Optimization Iterations:** 5 iterations for each image [S1].
*   **Optimizer:** Adam optimizer [S1].
*   **Truncation Threshold ($\delta$):** Defined in the projective TSDF formula, but a specific numerical value is not given in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Depth Noise:** Severe depth noise causes discontinuities between neighboring implicit voxels. Trilinear interpolation among neighboring implicit voxels is used to improve border consistency, but discontinuities may still occur [S1].
*   **Initialization:** Global-level fusion initialized by local-level fusion converges in 5 iterations. Initialized randomly, it takes much longer to converge and remains inferior even after 20 iterations [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** On a 1080Ti GPU, encoding + local-level fusion takes 0.1s total; global-level fusion for 2 iterations takes 0.5s; overall runs at ~2 fps [S1]. DI-Fusion runs at 10 fps on the same device [S1].
*   **Scaling:** Not given in the sources [S1].
*   **Strengths:** Outperforms TSDF-Fusion, iMAP, and DI-Fusion on ICL-NUIM and ScanNet datasets in accuracy, completeness, and F1 score, demonstrating robustness to synthetic noise and better generalization to diverse indoor scenes [S1]. The combination of local- and global-level fusion significantly improves reconstruction quality compared to using either alone [S1].
*   **Limitations:** Slower than traditional volumetric fusion approaches (e.g., InfiniTAM v3 runs at over 30 fps on a mobile device) [S1]. Severe noise can still cause discontinuities despite trilinear interpolation [S1].
*   **When to Use:** Online 3D reconstruction where global consistency is important and depth noise is present [S1].

## Background
Bi-level Fusion was introduced in the paper "BNV-Fusion: Dense 3D Reconstruction using Bi-level Neural Volume Fusion" [S1]. It is an online 3D reconstruction method that uses implicit neural volumes to represent geometry [S1]. The method was evaluated on the augmented ICL-NUIM dataset, 3D Scene Dataset, and ScanNet validation set (312 sequences) [S1]. For ScanNet, every 10th frame was used as input, and ground truth meshes were used as pseudo ground truth [S1]. The depth encoder and shape decoder were trained using object CAD models from two categories (chairs and lamps) in ShapeNet [S1].

## Sources
[S1] BNV-Fusion: Dense 3D Reconstruction using Bi-level Neural Volume Fusion, https://doi.org/10.1109/CVPR52688.2022.00607, fetched 2026-10-30
