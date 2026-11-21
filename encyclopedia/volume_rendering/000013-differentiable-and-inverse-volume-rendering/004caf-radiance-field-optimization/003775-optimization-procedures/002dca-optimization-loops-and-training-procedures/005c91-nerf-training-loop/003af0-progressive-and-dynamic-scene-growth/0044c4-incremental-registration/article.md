# Incremental Registration

Also called: Incremental CONfidence (ICON) registration, Incremental Image Registration

## Overview
Incremental Registration is a training strategy for joint pose and radiance field optimization that registers camera frames sequentially following their temporal order [S2]. It solves the problem of non-convex pose optimization and local minima by leveraging the intuition that adjacent camera poses in video sequences are spatially close [S2]. Unlike methods that optimize all poses simultaneously or rely on global initialization, this method incrementally adds frames to the training buffer, initializing new poses from previous ones and freezing older parameters to prevent disruption [S2].

## How it works
The method operates in two primary modes depending on the underlying architecture: the ICON approach for NeRF-based radiance fields [S2] and the ZeroGS approach for 3D Gaussian Splatting [S1, unofficial]. Both follow the shared principles of metric computation, expansion triggers, and re-optimization defined in [Parent](8869-progressive-and-dynamic-scene-growth.md).

### Inputs and Outputs
*   **Inputs:** A sequence of unposed images $\{I_1, I_2, \dots, I_N\}$; a scene regressor (NeRF or Gaussian Splatting); hyperparameters $k$ (iteration interval), $T_{freeze}$ (freeze duration), and $T_{init}$ (initialization iterations).
*   **Outputs:** Optimized radiance field parameters $\Theta$ and camera poses $\{P_1, \dots, P_N\}$.

### Data Structures
*   **Training Buffer:** A set of registered images and their associated poses.
*   **Scene Regressor:** The neural network (NeRF or 3DGS) representing the scene.
*   **Pose Parameters:** A vector or matrix representing the 6-DoF pose $P_i$ for each image $I_i$.

### Steps

#### 1. Initialization
*   **ICON (NeRF):** Jointly optimize NeRF parameters $\Theta$ and poses $\{P_1, P_2\}$ using the first two frames $\{I_1, I_2\}$ [S2].
*   **ZeroGS (3DGS):** Select a seed image $I_{seed}$. Initialize the scene regressor by passing $I_{seed}$ to itself: $(X_i, G_i; X_j, G_j) = f_{SCR}(I_{seed}, I_{seed})$, fixing the camera pose as an identity matrix [S1, unofficial]. Finetune the scene regressor for $T_{init}$ iterations (e.g., 500 iterations) [S1, unofficial].

#### 2. Incremental Registration Loop
Repeat the following until all images are registered:

1.  **Select Batch:** Select a batch of unregistered images $I_{buf} = \{I_k\}$ [S1, unofficial]. In ZeroGS, the reference image is selected from registered images that connects to the most unregistered images [S1, unofficial].
2.  **Pose Initialization:** For each new frame $I_i$, initialize its pose $P_i$ using the previous pose $P_{i-1}$ [S2].
3.  **Add to Buffer:** Add $I_i$ and $P_i$ to the training buffer [S1, unofficial][S2].
4.  **Optimization:**
    *   **ICON:** Freeze the learning rate on poses $\{P_i\}$ and NeRF $\Theta$ for $T_{freeze}$ iterations [S2].
    *   **ZeroGS:** Fix camera poses registered in previous epochs and only optimize poses registered in the current epoch [S1, unofficial].
    *   **Convergence:** Train until convergence or for a fixed number of iterations (e.g., 1,000 for LLFF, 1,500 for Mip-NeRF360) [S1, unofficial].
5.  **Trigger Check:** If using a metric-based trigger (e.g., inlier count $> s_{inlier}$), add the image to the buffer only if the metric is satisfied [S1, unofficial].

#### 3. Finalization
*   **ICON:** After all frames are registered, apply a learning rate decay schedule [S2].
*   **ZeroGS:** Optimize all camera poses using the reprojection error loss:
    $$ \arg \min_{X_i, C_k} \sum_{i,k} \rho(\|d_{i,k} \cdot \nu_{i,k} - (X_i - C_k)\|_2) $$
    where $C_k$ is the camera center, $d_{i,k}$ is the scaling factor, and $\nu_{i,k}$ is the ray direction [S1, unofficial]. Fix the seed image pose and optimize all others [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def incremental_registration(images, scene_regressor, k, T_freeze, T_init):
    # 1. Initialization
    # ICON: Optimize first two frames
    if method == "ICON":
        optimize_jointly(scene_regressor, images[0:2])
        registered = {1: images[0], 2: images[1]}
    else:
        # ZeroGS: Seed initialization
        seed = images[0]
        finetune(scene_regressor, seed, T_init)
        registered = {1: seed}
        scene_regressor.set_pose(1, Identity())

    i = 2
    while i <= len(images):
        # 2. Select Batch
        batch = select_batch(images[i:i+k], registered)
        
        for frame in batch:
            # 3. Pose Initialization
            prev_pose = scene_regressor.get_pose(registered[-1].id)
            scene_regressor.set_pose(frame.id, prev_pose)
            
            # 4. Add to Buffer
            registered.append(frame)
            
            # 5. Optimization
            if method == "ICON":
                freeze_learning_rate(scene_regressor, poses=True, nerf=True)
                train_iterations(T_freeze)
                unfreeze_learning_rate(scene_regressor)
            else:
                # ZeroGS: Fix old poses, optimize new
                fix_old_poses(registered)
                optimize_current_poses(scene_regressor, batch)
                train_iterations(T_init)
        
        i += k

    # 3. Finalization
    if method == "ICON":
        apply_lr_decay()
    else:
        global_pose_optimization(scene_regressor, registered)
```

### Parameters
*   **$k$ (Iteration Interval):** Number of iterations between adding new frames. Value: 1,000 for CO3D/HO3D; 500 for LLFF [S2].
*   **$T_{freeze}$ (Freeze Duration):** Iterations to freeze learning rates. Value: 100,000 for HO3D/CO3D; 30,000 for LLFF [S2].
*   **$T_{init}$ (Initialization Iterations):** Finetuning iterations for scene regressor. Value: 500 for seed; 1,000 for LLFF; 1,500 for Mip-NeRF360 [S1, unofficial].
*   **$s_{inlier}$ (Inlier Threshold):** Minimum inlier count for registration. Value: 5,000 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Local Minima:** Optimizing poses and NeRF using photometric losses is highly non-convex [S2]. Incorrectly optimized poses can mislead the NeRF, causing re-registration on already registered viewpoints [S2].
*   **Pose Drift:** Errors accumulate during incremental training. Finalization stages are required to refine poses [S1, unofficial].
*   **Occlusion:** Avoid sampling rays from occluded regions [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of frames, as each frame is processed once. However, the total training time is significantly higher than global optimization due to the iterative nature and freezing/unfreezing cycles [S2].
*   **Memory Complexity:** Bounded by the number of registered frames and the scene representation size. Freezing older parameters helps manage memory [S2].
*   **Strengths:** Provides good initialization for camera poses, avoiding the breaking point of 20-degree rotation difference observed in global optimization [S2]. Efficient for video sequences with smooth camera motion [S2].
*   **Limitations:** Not suitable for scenes with large, abrupt camera motions or non-sequential image sets [S2]. Requires careful tuning of $k$ and $T_{freeze}$ [S2].

## Background
Incremental Registration was introduced in the ICON (Incremental CONfidence) paper [S2] and adopted in ZeroGS [S1, unofficial]. It addresses the limitations of global pose optimization, which suffers from local minima and requires good initial poses [S2]. The method leverages the temporal coherence of video data [S2].

## Sources
[S1] ZeroGS: Training 3D Gaussian Splatting from Unposed Images, https://doi.org/10.48550/arxiv.2411.15779, fetched 2026-10-26 (unofficial)
[S2] ICON: Incremental CONfidence for Joint Pose and Radiance Field Optimization, https://doi.org/10.1109/cvpr52733.2024.00517, fetched 2026-10-26
