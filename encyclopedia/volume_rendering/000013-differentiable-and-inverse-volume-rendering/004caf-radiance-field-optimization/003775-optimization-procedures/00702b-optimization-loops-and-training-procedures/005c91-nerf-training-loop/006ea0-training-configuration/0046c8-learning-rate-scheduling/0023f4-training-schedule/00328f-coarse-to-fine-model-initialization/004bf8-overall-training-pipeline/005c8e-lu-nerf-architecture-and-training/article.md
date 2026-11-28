# LU-NeRF Architecture and Training

Also called: Compact LU-NeRF, LU-NeRF Network Architecture and Training, Local Unposed NeRF, Local Unposed NeRF training

## Overview
LU-NeRF estimates camera poses and neural scene representations from unordered, unposed image collections by processing them as local mini-scenes before global synchronization [S1]. It differs from prior unposed NeRF methods by operating in the general SE(3) pose setting without restrictive pose priors, using a compact view-independent 4-layer MLP without positional encoding, and resolving mirror-symmetry ambiguities through a dedicated secondary training stage [S1].

## How it works
**Inputs:** A set of unposed images partitioned into mini-scenes [S1].
**Outputs:** Optimized camera poses (world-to-camera) and a neural radiance field representation for each mini-scene [S1].

**Data Structures:**
*   **Camera Poses:** For a mini-scene with $N$ images, poses are stored as a set of world-to-camera rotation matrices $\{R_i\}_{i=1}^N$ and translation vectors $\{t_i\}_{i=1}^N$ [S1]. Exact memory layout and bit-packing are Not given in the sources.
*   **Neural Field:** A compact 4-layer Multi-Layer Perceptron (MLP) [S1]. It does not use positional encoding and does not condition on view directions (view-independent) [S1]. Exact memory layout and bit-packing are Not given in the sources.
*   **Ray Sampling:** 2048 rays are sampled per mini-batch [S1].

**Parameters:**
*   **Initial Poses:** Identity matrix for rotations and zero for translations [S1]. Random initialization around identity yields no significant difference [S1].
*   **Learning Rate:** Adopted from BARF [S1].
*   **Depth Regularization Weight:** 10 times larger than the photometric $\ell_2$ loss [S1].
*   **Regularization Patch Size:** $2 \times 2$ pixels [S1].
*   **LU-NeRF-1 Termination Threshold:** Average camera rotation change less than $0.125^\circ$ [S1].
*   **LU-NeRF-2 Default Iterations:** 50,000 total (5,000 frozen poses + 45,000 joint optimization) [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize camera poses $\{R_i, t_i\}$ to identity [S1]. Initialize the 4-layer MLP weights [S1]. Do not apply positional encoding or view-direction conditioning to the MLP [S1].
2.  **LU-NeRF-1 Training (Local Optimization):** Sample 2048 rays per mini-batch [S1]. Compute photometric $\ell_2$ loss between rendered and ground-truth pixels [S1]. Apply depth regularization on $2 \times 2$ patches [S1]. Jointly optimize camera poses and MLP parameters using the BARF learning rate schedule [S1]. Terminate training if the average change of camera rotations in the mini-scene is less than $0.125^\circ$ within 5,000 iterations [S1].
3.  **Mirror-Symmetry Ambiguity Mitigation (LU-NeRF-2):** Take estimated poses $\{R_i\}$ from LU-NeRF-1 [S1]. Generate reflected cameras $\{R_\pi R_i\}$, where $R_\pi$ is a rotation around the optical axis [S1]. Train two new models from scratch: one initialized with original poses, one with reflected poses [S1]. Train for 5,000 iterations with frozen initial poses [S1]. Jointly optimize camera poses and neural fields for 45,000 iterations [S1]. Resolve ambiguity by selecting the model with the smallest photometric training loss [S1].
4.  **Global Synchronization and Refinement:** Bring local mini-scene poses into a global reference frame using pose synchronization (e.g., Shonan averaging) [S1]. Perform a final global optimization of all camera poses and the neural scene representation using all images from the entire scene [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_lu_nerf_mini_scene(images, mini_scene_indices):
    # 1. Initialization
    # Poses initialized to identity [S1]
    poses = [IdentityPose() for _ in mini_scene_indices]
    # Compact 4-layer MLP without positional encoding or view-direction conditioning [S1]
    mlp = Compact4LayerMLP()
    previous_poses = copy(poses)
    max_iterations = 100000  # Not given in the sources.

    # 2. LU-NeRF-1 Training
    for iteration in range(max_iterations):
        # Sample 2048 rays per mini-batch [S1]
        rays = sample_rays(images, mini_scene_indices, num_rays=2048)
        # Render colors and depths
        rendered_colors, rendered_depths = render_rays(rays, poses, mlp)

        # Compute photometric L2 loss [S1]
        photometric_loss = mean_squared_error(rendered_colors, rays.gt_colors)
        # Apply depth regularization on 2x2 patches [S1]
        depth_reg_loss = apply_depth_regularization(rendered_depths, patch_size=(2, 2))

        # Depth regularization weight is 10x photometric loss [S1]
        total_loss = photometric_loss + 10.0 * depth_reg_loss

        # Optimize poses and MLP with BARF learning rate schedule [S1]
        optimize(poses, mlp, total_loss)

        # Check termination condition [S1]
        avg_rotation_change = compute_avg_rotation_change(poses, previous_poses)
        if avg_rotation_change < 0.125 and iteration >= 5000:
            break
        previous_poses = copy(poses)

    estimated_poses = poses

    # 3. Mirror-Symmetry Ambiguity Mitigation (LU-NeRF-2)
    # Generate reflected cameras {R_pi R_i} [S1]
    reflected_poses = [reflect_pose(p, axis='optical') for p in estimated_poses]

    # Train two models from scratch [S1]
    model_a_poses, model_a_mlp = train_lu_nerf_2_stage(estimated_poses, images, mini_scene_indices)
    model_b_poses, model_b_mlp = train_lu_nerf_2_stage(reflected_poses, images, mini_scene_indices)

    # Select model with smallest photometric training loss [S1]
    loss_a = compute_photometric_loss(model_a_poses, model_a_mlp, images, mini_scene_indices)
    loss_b = compute_photometric_loss(model_b_poses, model_b_mlp, images, mini_scene_indices)

    if loss_a < loss_b:
        final_poses = model_a_poses
        final_mlp = model_a_mlp
    else:
        final_poses = model_b_poses
        final_mlp = model_b_mlp

    return final_poses, final_mlp

def train_lu_nerf_2_stage(initial_poses, images, mini_scene_indices):
    poses = copy(initial_poses)
    mlp = Compact4LayerMLP() # Scene representation started from scratch [S1]

    # Stage 1: 5k iterations with frozen initial poses [S1]
    for iteration in range(5000):
        rays = sample_rays(images, mini_scene_indices, num_rays=2048)
        rendered_colors, rendered_depths = render_rays(rays, poses, mlp)
        photometric_loss = mean_squared_error(rendered_colors, rays.gt_colors)
        depth_reg_loss = apply_depth_regularization(rendered_depths, patch_size=(2, 2))
        total_loss = photometric_loss + 10.0 * depth_reg_loss
        optimize(mlp, total_loss) # Poses frozen

    # Stage 2: 45k iterations joint optimization [S1]
    for iteration in range(45000):
        rays = sample_rays(images, mini_scene_indices, num_rays=2048)
        rendered_colors, rendered_depths = render_rays(rays, poses, mlp)
        photometric_loss = mean_squared_error(rendered_colors, rays.gt_colors)
        depth_reg_loss = apply_depth_regularization(rendered_depths, patch_size=(2, 2))
        total_loss = photometric_loss + 10.0 * depth_reg_loss
        optimize(poses, mlp, total_loss) # Joint optimization

    return poses, mlp

# Helper definitions
def IdentityPose(): return {'R': [[1,0,0],[0,1,0],[0,0,1]], 't': [0,0,0]}
def copy(pose_list): return [{'R': [row[:] for row in p['R']], 't': p['t'][:]} for p in pose_list]
def sample_rays(images, indices, num_rays): return generate_random_pixel_coords(images, indices, num_rays)
def render_rays(rays, poses, mlp): return mlp.forward(rays, poses)
def mean_squared_error(pred, target): return sum((p - t) ** 2 for p, t in zip(pred, target)) / len(pred)
def apply_depth_regularization(depths, patch_size): return compute_patch_smoothness(depths, patch_size)
def optimize(poses, mlp, loss): return gradient_step(poses, mlp, loss)
def compute_avg_rotation_change(current, previous): return sum(rotation_distance(c, p) for c, p in zip(current, previous)) / len(current)
def reflect_pose(pose, axis): return reflect_rotation_around_axis(pose['R'], axis)
def compute_photometric_loss(poses, mlp, images, indices): return mean_squared_error(render_rays(sample_rays(images, indices, 2048), poses, mlp)[0], sample_rays(images, indices, 2048).gt_colors)
```

## Subtopics

### Mirror-symmetry ambiguity mitigation
This variant addresses the degeneracy where distant small objects appear affine, causing ambiguity between a scene and its reflection [S1]. It is chosen when the initial local optimization produces inverted disparity maps and reflected poses [S1]. It resolves the ambiguity by training two additional models initialized with original and reflected camera poses, then selecting the one with the lowest photometric training loss [S1]. See [LU-NeRF Architecture and Training](13114-lu-nerf-architecture-and-training.md) for the full training pipeline.

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** LU-NeRF outperforms prior unposed NeRF attempts (GNeRF, VMRF) on pose estimation and compares favorably to COLMAP on low-texture and low-resolution images [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** The removal of positional encoding and view-dependency, combined with a compact 4-layer MLP, significantly reduces memory cost and speeds up training compared to standard NeRF architectures [S1]. The local-to-global pipeline is designed to be robust, succeeding even when initial mini-scenes contain frequent outliers [S1].
*   **Limitations:** Performance may degrade if the noisy connected graph built with DINO/RGB features is used for pose synchronization; clean graphs built from ground-truth poses yield better results [S1].
*   **When to use:** Few-shot local unposed NeRF tasks requiring general SE(3) pose estimation without restrictive pose priors [S1].
*   **When not to use:** Scenarios requiring view-dependent appearance modeling or high-frequency positional encoding [S1].

## Background
LU-NeRF was introduced in 2023 by researchers addressing the challenges of few-shot local unposed NeRF [S1]. It builds on the concept of neural radiance fields (NeRF) [S1] and pose synchronization techniques [20, 43, 24]. The method was developed to operate in the general SE(3) pose setting without restrictive assumptions on pose priors, distinguishing it from earlier baselines [S1].

## Sources
[S1] LU-NeRF: Scene and Pose Estimation by Synchronizing Local Unposed NeRFs, https://doi.org/10.1109/iccv51070.2023.01679, fetched 2026-10-26
