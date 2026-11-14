# DBARF

Also called: DBARF (Deep Bundle-Adjusting Generalizable Neural Radiance Fields), Deep Bundle-Adjusting Generalizable Neural Radiance Fields

## Overview
DBARF jointly optimizes Generalizable Neural Radiance Fields (GeNeRFs) and relative camera poses in a self-supervised manner without requiring accurate initial poses [S1]. It differs from BARF, which relies on coarse-to-fine training with positional encodings absent in GeNeRFs, by using a RAFT-like recurrent network to implicitly minimize feature consistency errors, bypassing direct gradient descent on the cost function [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of unposed images $\{I_i\}$ for a scene.
*   **Outputs:** Optimized relative camera poses $\{P_{ij}\}$, depth maps $\{D_i\}$, and rendered novel views.

**Data Structures:**
*   **Scene Graph:** A graph where nodes are images and edges connect "nearby" views. Constructed once per scene using SuperPoint keypoints and SuperGlue matches, filtered by epipolar constraints, and sorted by inlier count [S1].
*   **Feature Maps:** Extracted using a ResNet-like backbone with a Feature Pyramid Network (FPN) [S1].
*   **Cost Map:** A 2D map representing feature inconsistency between the target view and nearby views [S1].

**Algorithm Steps:**

1.  **Scene Graph Construction:** For a target view $i$, select nearby views $\mathcal{N}(i)$ from the precomputed scene graph based on the number of inlier matches [S1].
2.  **Feature Extraction:** Extract image features $\mathbf{F}_i$ for the target view and $\mathbf{F}_j$ for each nearby view $j \in \mathcal{N}(i)$ using the FPN backbone [S1].
3.  **3D Point Sampling:** For each target pixel $\mathbf{u}_i$, sample a patch $\mathcal{S}(\mathbf{u}_i)$ from the target view. Compute 3D points $\mathbf{X}_{\mathcal{S}(\mathbf{u}_i)}$ using the predicted depth map $D_i$ for the target view [S1].
4.  **Initialization:** Initialize relative poses $\mathbf{P}_{ij}^0$ and depth $D_i^0$ randomly [S1].
5.  **Cost Map Computation:** Compute the initial cost map $\mathcal{C}^0$ using the feature difference between the warped nearby features and the target features [S1]:
    $$
    \mathcal{C} = \frac{1}{|\mathcal{N}(i)|} \sum_{j \in \mathcal{N}(i)} \| \chi (\mathbf{K}_j \mathbf{P}_{ij} \mathbf{X}_{\mathcal{S}(\mathbf{u}_i)}, \mathbf{F}_j) - \chi (\mathcal{S}(\mathbf{u}_i), \mathbf{F}_i) \|
    $$
    where $\mathbf{K}_j$ is the intrinsics matrix, $\mathbf{P}_{ij} = \mathbf{P}_j \mathbf{P}_i^{-1}$ is the relative pose, $\chi$ is differentiable bilinear interpolation, and $\mathbf{X}_{\mathcal{S}(\mathbf{u}_i)}$ are 3D points derived from $D_i$ [S1].
6.  **Recurrent Pose and Depth Optimization:** Use a GRU (following RAFT-like architecture) to predict corrections $\Delta \mathbf{P}_{ij}$ and $\Delta D_i$ based on the cost map. Update poses and depth for $t$ iterations:
    $$
    \mathbf{P}_{ij}^{t+1} = \mathbf{P}_{ij}^t + \Delta \mathbf{P}_{ij}, \quad \mathbf{D}_i^{t+1} = \mathbf{D}_i^t + \Delta \mathbf{D}_i
    $$
    The cost map is updated after each iteration using the current poses and depth [S1].
7.  **Volume Rendering:** Use the optimized poses and features in a GeNeRF (e.g., IBRNet) to predict volume density and pixel color. Render the target image using volume rendering [S1].
8.  **Joint Training:** The pose optimizer and GeNeRF are jointly learned [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dbarf_training(images, scene_graph, ge_nerf, feature_backbone, pose_estimator, 
                   num_nearby_views, num_recurrent_iters, beta, lr_extractor, lr_ge_nerf, lr_pose):
    """
    DBARF Training Loop.
    images: List of input images.
    scene_graph: Precomputed scene graph defining nearby views for each image.
    ge_nerf: Generalizable NeRF model (e.g., IBRNet).
    feature_backbone: ResNet-like FPN for feature extraction.
    pose_estimator: GRU-based network for pose/depth correction.
    num_nearby_views: Number of nearby views to use (5 for pretraining, 10 for fine-tuning).
    num_recurrent_iters: Number of recurrent iterations for pose optimization (4) [S1].
    beta: Coefficient for loss weighting schedule (-1e5) [S1].
    """
    # Standard gradient descent optimizers [G]
    opt_extractor = Optimizer(feature_backbone.parameters(), lr=lr_extractor)
    opt_ge_nerf = Optimizer(ge_nerf.parameters(), lr=lr_ge_nerf)
    opt_pose = Optimizer(pose_estimator.parameters(), lr=lr_pose)
    
    total_iterations = 200000  # Pretraining iterations [S1]
    for iteration in range(total_iterations):
        for target_img, nearby_indices in scene_graph.batches(images):
            nearby_imgs = [images[idx] for idx in nearby_indices[:num_nearby_views]]
            
            # 1. Feature Extraction [S1]
            F_target = feature_backbone(target_img)
            F_nearby = [feature_backbone(img) for img in nearby_imgs]
            
            # 2. Initialization [S1]
            P_ij = random_uniform_relative_poses(len(nearby_imgs))
            D_target = random_uniform_depth(target_img.shape)
            
            # 3. Recurrent Pose Optimization [S1]
            for t in range(num_recurrent_iters):
                # Compute 3D points from depth [S1]
                X_S = depth_to_3d_points(D_target, target_img.intrinsics)
                
                # Compute Cost Map [S1]
                C = compute_cost_map(F_target, F_nearby, P_ij, X_S, target_img.intrinsics)
                
                # Predict corrections using GRU [S1]
                delta_P, delta_D = pose_estimator(C, F_target, F_nearby, P_ij, D_target)
                
                # Update poses and depth [S1]
                P_ij = P_ij + delta_P
                D_target = D_target + delta_D
            
            # 4. Volume Rendering [S1]
            rendered_img = ge_nerf.render(target_img, P_ij, F_target, F_nearby)
            
            # 5. Loss Computation [S1]
            L_depth = compute_depth_smoothness_loss(D_target, target_img)
            L_photo = compute_photometric_loss(rendered_img, target_img, nearby_imgs, P_ij)
            L_rgb = compute_rgb_loss(rendered_img, target_img)
            
            weight = 2 ** (beta * iteration)
            L_final = weight * (L_depth + L_photo) + (1 - weight) * L_rgb
            
            # 6. Backpropagation [G]
            opt_extractor.zero_grad()
            opt_ge_nerf.zero_grad()
            opt_pose.zero_grad()
            L_final.backward()
            opt_extractor.step()
            opt_ge_nerf.step()
            opt_pose.step()

def compute_cost_map(F_target, F_nearby, P_ij, X_S, K):
    """
    Computes the feature consistency cost map.
    Matches Eq. 11 in [S1].
    """
    cost_sum = 0.0
    for j, F_j in enumerate(F_nearby):
        # Project 3D points to nearby view coordinates [S1]
        proj_coords = K @ (P_ij[j] @ X_S)
        # Sample features from nearby view using bilinear interpolation [S1]
        F_j_warped = bilinear_interpolate(F_j, proj_coords)
        # Sample features from target view patch [S1]
        F_target_patch = bilinear_interpolate(F_target, X_S)
        # Compute L2 norm difference [S1]
        cost_sum += norm(F_j_warped - F_target_patch)
    return cost_sum / len(F_nearby)

def compute_depth_smoothness_loss(D, I):
    """
    Edge-aware depth smoothness loss. Matches Eq. 13 in [S1].
    """
    dD_x, dD_y = gradient(D, axis=0), gradient(D, axis=1)
    dI_x, dI_y = gradient(I, axis=0), gradient(I, axis=1)
    return abs(dD_x) * exp(-abs(dI_x)) + abs(dD_y) * exp(-abs(dI_y))

def compute_photometric_loss(rendered_img, target_img, nearby_imgs, P_ij):
    """
    Warped photometric loss. Matches Eq. 14 in [S1].
    """
    alpha = 0.85
    loss_sum = 0.0
    for j, nearby_img in enumerate(nearby_imgs):
        # Warp nearby image to target view using P_ij [S1]
        warped_img = warp_image(nearby_img, P_ij[j], target_img.intrinsics)
        # SSIM Loss [S1]
        ssim_val = structural_similarity(warped_img, target_img)
        # L1 Loss [S1]
        l1_val = norm(warped_img - target_img)
        loss_sum += alpha * (1 - ssim_val) / 2 + (1 - alpha) * l1_val
    return loss_sum / len(nearby_imgs)

def compute_rgb_loss(rendered_img, target_img):
    """
    Photometric error between rendered and ground truth.
    """
    return norm(rendered_img - target_img)

# Helper definitions [G]
def bilinear_interpolate(feature_map, coords):
    return feature_map[coords]

def gradient(tensor, axis):
    return tensor[1:] - tensor[:-1] if axis == 0 else tensor[:, 1:] - tensor[:, :-1]

def norm(tensor):
    return sqrt(sum(p**2 for p in tensor.flatten()))

def warp_image(img, pose, intrinsics):
    return img

def structural_similarity(a, b):
    return 1.0

def random_uniform_relative_poses(n):
    return [[1,0,0,0],[0,1,0,0],[0,0,1,0],[0,0,0,1]] * n

def random_uniform_depth(shape):
    return [[1.0]*shape[1] for _ in range(shape[0])]

def depth_to_3d_points(depth, intrinsics):
    return depth

class Optimizer:
    def __init__(self, params, lr):
        self.params = params
        self.lr = lr
    def zero_grad(self):
        pass
    def step(self):
        for p in self.params:
            if p.grad is not None:
                p -= self.lr * p.grad
```

**Parameters:**
*   **Learning Rate (Feature Extractor):** $1 \times 10^{-3}$ (pretraining), $5 \times 10^{-4}$ (fine-tuning) [S1].
*   **Learning Rate (GeNeRF):** $5 \times 10^{-4}$ (pretraining), $2 \times 10^{-4}$ (fine-tuning) [S1].
*   **Learning Rate (Pose Optimizer):** $2 \times 10^{-4}$ (pretraining), $1 \times 10^{-5}$ (fine-tuning) [S1].
*   **Nearby Views:** 5 during pretraining, 10 during fine-tuning and evaluation [S1].
*   **Pose Update Iterations:** 4 per batch [S1].
*   **Training Iterations:** 200,000 pretraining, 60,000 fine-tuning [S1].
*   **Beta Coefficient:** $-1 \times 10^{5}$ [S1].

**Edge Cases and Pitfalls:**
*   Direct gradient descent on camera poses with GeNeRFs fails because image features can be outliers (e.g., due to occlusion), causing erroneous gradients [S1].
*   ResNet features create non-smooth cost maps that coarse-to-fine strategies cannot fix due to the lack of positional encodings in GeNeRFs [S1].
*   NeRF randomly samples points during training, losing spatial information of features if the cost function is taken directly as input [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of nearby views, recurrent iterations, and GeNeRF inference. The recurrent optimization adds a fixed overhead per batch (4 iterations) [S1].
*   **Memory Complexity:** Requires storing feature maps for target and nearby views, and intermediate cost maps during recurrent updates [S1].
*   **Performance:** Evaluated on NVIDIA RTX A5000 GPU [S1].
*   **Strengths:** Handles unposed images effectively; bypasses gradient issues associated with direct optimization in GeNeRFs [S1].
*   **Limitations:** Requires a precomputed scene graph; relies on the quality of the feature backbone [S1].

## Background
DBARF was proposed in 2023 by the authors of the paper "DBARF: Deep Bundle-Adjusting Generalizable Neural Radiance Fields" [S1]. It addresses the limitations of BARF, which uses positional encodings for coarse-to-fine training, a technique not applicable to GeNeRFs like IBRNet that operate directly on features [S1]. The method is inspired by RAFT for optical flow and BANet for cost volume processing [S1].

## Sources
[S1] DBARF: Deep Bundle-Adjusting Generalizable Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00011, fetched 2026-10-29
