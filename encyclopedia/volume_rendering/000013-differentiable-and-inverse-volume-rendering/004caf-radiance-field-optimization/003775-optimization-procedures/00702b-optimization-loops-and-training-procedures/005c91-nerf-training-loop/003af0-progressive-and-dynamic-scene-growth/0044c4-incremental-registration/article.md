# Incremental Registration

Also called: Incremental CONfidence (ICON) registration, Incremental Image Registration

## Overview
Incremental Registration is a training strategy for joint pose and radiance field optimization that registers camera frames sequentially following their temporal order [S2]. It solves the problem of non-convex pose optimization and local minima by leveraging the intuition that adjacent camera poses in video sequences are spatially close [S2]. Unlike methods that optimize all poses simultaneously or rely on global initialization, this method incrementally adds frames to the training buffer, initializing new poses from previous ones and freezing older parameters to prevent disruption [S2]. Builds on [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md) [G].

## How it works
The method adapts to the underlying scene representation architecture, primarily implemented for NeRF-based radiance fields [S2] and 3D Gaussian Splatting [S1, unofficial]. Both variants follow the shared principles of metric computation, expansion triggers, and re-optimization defined in [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md) [S1, unofficial][S2].

### Inputs and Outputs
*   **Inputs:** A sequence of unposed images $\{I_1, I_2, \dots, I_N\}$; a scene regressor (NeRF or 3DGS); hyperparameters $k$ (iteration interval), $T_{freeze}$ (freeze duration), $T_{init}$ (seed finetuning iterations), and $T_{finetune}$ (per-batch finetuning iterations) [S1, unofficial][S2].
*   **Outputs:** Optimized radiance field parameters $\Theta$ and camera poses $\{P_1, \dots, P_N\}$ [S1, unofficial][S2].

### Data Structures
*   **Training Buffer:** A set of registered images and their associated poses [S1, unofficial][S2].
*   **Scene Regressor:** The neural network (NeRF or 3DGS) representing the scene [S1, unofficial][S2].
*   **Pose Parameters:** A $4 \times 4$ matrix representing the 6-DoF pose $P_i$ for each image $I_i$ [S1, unofficial][S2].

### Steps

#### 1. Initialization
*   **ICON (NeRF):** Jointly optimize NeRF parameters $\Theta$ and poses $\{P_1, P_2\}$ using the first two frames $\{I_1, I_2\}$ [S2].
*   **ZeroGS (3DGS):** Select a seed image $I_{seed}$. Initialize the scene regressor by passing $I_{seed}$ to itself: $(X_i, G_i; X_j, G_j) = f_{SCR}(I_{seed}, I_{seed})$, fixing the camera pose as an identity matrix [S1, unofficial]. Finetune the scene regressor for $T_{init}$ iterations (e.g., 500 iterations) [S1, unofficial].

#### 2. Incremental Registration Loop
Repeat the following until all images are registered:

1.  **Train:** Run $k$ training iterations on the current training buffer [S2].
2.  **Select Batch:** Select a batch of unregistered images $I_{buf} = \{I_k\}$ [S1, unofficial]. In ZeroGS, the reference image is selected from registered images that connects to the most unregistered images [S1, unofficial].
3.  **Pose Initialization:** For each new frame $I_i$, initialize its pose $P_i$ using the previous pose $P_{i-1}$ [S2].
4.  **Add to Buffer:** Add $I_i$ and $P_i$ to the training buffer [S1, unofficial][S2].
5.  **Trigger Check (ZeroGS):** Add the target image $I_k$ into the training buffer only when the inlier number is larger than the inlier threshold $s_{inlier}$ [S1, unofficial].
6.  **Optimization:**
    *   **ICON:** Freeze the learning rate on poses $\{P_i\}$ and NeRF $\Theta$ for $T_{freeze}$ iterations [S2].
    *   **ZeroGS:** Fix camera poses registered in previous epochs and only optimize the camera poses registered in the current epoch [S1, unofficial]. Finetune the scene regressor for $T_{finetune}$ iterations [S1, unofficial].
7.  **Convergence:** Train until convergence or for the fixed iteration count [S1, unofficial][S2].

#### 3. Finalization
*   **ICON:** After all frames are registered, apply a learning rate decay schedule [S2].
*   **ZeroGS:** Optimize all camera poses using the reprojection error loss:
    $$ \arg \min_{X_i, C_k} \sum_{i,k} \rho(\|d_{i,k} \cdot \nu_{i,k} - (X_i - C_k)\|_2) $$
    where $C_k$ is the camera center, $d_{i,k}$ is the scaling factor, and $\nu_{i,k}$ is the ray direction [S1, unofficial]. Fix the seed image pose and optimize all others [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def incremental_registration(images, regressor, k, T_freeze, T_init, T_finetune, s_inlier):
    registered = []
    poses = {}
    i = 2 if regressor.type == "NeRF" else 1

    # 1. Initialization
    if regressor.type == "NeRF":
        optimize_jointly(regressor, images[0], images[1])
        registered = [images[0], images[1]]
        poses[0] = regressor.get_pose(images[0])
        poses[1] = regressor.get_pose(images[1])
    else:
        seed = images[0]
        for _ in range(T_init):
            regressor.finetune(seed)
        registered = [seed]
        poses[0] = identity_matrix()
        regressor.set_pose(0, poses[0])

    while i < len(images):
        # 2. Train for k iterations
        for _ in range(k):
            train_step(regressor, registered)

        # 3. Pose Initialization & Buffer Add
        frame = images[i]
        poses[i] = poses[i-1]
        regressor.set_pose(i, poses[i])

        if regressor.type == "3DGS":
            inliers = count_inliers(regressor, frame, poses[i])
            if inliers < s_inlier:
                i += 1
                continue
        registered.append(frame)

        # 4. Optimization Phase
        if regressor.type == "NeRF":
            freeze_learning_rate(regressor, ["poses", "nerf"])
            for _ in range(T_freeze):
                train_step(regressor, registered)
            unfreeze_learning_rate(regressor)
        else:
            freeze_learning_rate(regressor, ["old_poses"])
            for _ in range(T_finetune):
                optimize_current_poses(regressor, frame)
            unfreeze_learning_rate(regressor)

        i += 1

    # 5. Finalization
    if regressor.type == "NeRF":
        apply_lr_decay(regressor)
    else:
        global_pose_optimization(regressor, registered, poses)

# Helper definitions (standard operations) [G]
def optimize_jointly(regressor, f1, f2):
    regressor.train()
    for f in [f1, f2]:
        regressor.forward(f)
        regressor.backward()
        regressor.step()

def train_step(regressor, buffer):
    regressor.train()
    for f in buffer:
        regressor.forward(f)
        regressor.backward()
        regressor.step()

def count_inliers(regressor, frame, pose):
    # Count pixels with reprojection error below threshold [G]
    threshold = 6.0
    return sum(1 for p in frame.points if reprojection_error(p, pose) < threshold)

def optimize_current_poses(regressor, frame):
    regressor.optimize_pose(frame)
    regressor.step()

def global_pose_optimization(regressor, registered, poses):
    regressor.freeze_parameters("seed_pose")
    for _ in range(1000):
        regressor.forward_all(registered)
        regressor.backward()
        regressor.step()
```

### Parameters
*   **$k$ (Iteration Interval):** Number of iterations between adding new frames. Value: 1,000 iterations for CO3D/HO3D; 500 iterations for LLFF [S2].
*   **$T_{freeze}$ (Freeze Duration):** Iterations to freeze learning rates. Value: 100,000 iterations for HO3D/CO3D; 30,000 iterations for LLFF [S2].
*   **$T_{init}$ (Initialization Iterations):** Finetuning iterations for seed. Value: 500 iterations [S1, unofficial].
*   **$T_{finetune}$ (Per-Batch Finetuning):** Finetuning iterations during incremental training. Value: 1,000 iterations for LLFF; 1,500 iterations for Mip-NeRF360 [S1, unofficial].
*   **$s_{inlier}$ (Inlier Threshold):** Minimum inlier count for registration. Value: 5,000 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Local Minima:** Optimizing poses and NeRF using photometric losses is highly non-convex [S2]. Incorrectly optimized poses can mislead the NeRF, causing re-registration on already registered viewpoints [S2].
*   **Pose Drift:** Errors accumulate during incremental training [S1, unofficial]. Finalization stages are required to refine poses [S1, unofficial].
*   **Occlusion:** Avoid sampling rays from occluded regions [S2].

## Subtopics

### Incremental Joint Learning Optimization
A progressive optimization framework that jointly learns depth, RGB, camera poses, and scene representation by sequentially adding frames to the optimization process [S3]. It initializes with a small subset of frames (e.g., five frames) and progressively introduces new frames in sequential order [S3]. Simultaneously optimizes depth network parameters, RGB radiance field parameters, camera poses $\{R, t\}$, and scene representation $\theta$ using a combined loss function [S3]. The depth rendering function is $\hat{\mathbf{D}}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) d_i$ [S3]. The optical flow loss is $\mathcal{L}_{fa} = \left\| \hat{\mathcal{F}}_{k \rightarrow k+1} - \mathcal{F}_{k \rightarrow k+1} \right\|_1$ [S3]. The final loss is $\mathcal{L} = \lambda_1 \mathcal{L}_p + \lambda_2 \mathcal{L}_D + \lambda_3 \mathcal{L}_{FBA} + \lambda_4 \mathcal{L}_{fa} + \lambda_5 \mathcal{L}_{fb}$ [S3]. This joint optimization leverages 2D photometric consistency and 3D structural information to enhance depth estimation, pose prediction, and scene reconstruction [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Strengths:** Provides good initialization for camera poses, avoiding the breaking point of 20-degree rotation difference observed in global optimization [S2]. Efficient for video sequences with smooth camera motion [S2].
*   **Limitations:** Not suitable for scenes with large, abrupt camera motions or non-sequential image sets [S2]. Requires careful tuning of $k$ and $T_{freeze}$ [S2].

## Background
Incremental Registration was introduced in the ICON (Incremental CONfidence) paper [S2] and adopted in ZeroGS [S1, unofficial]. It addresses the limitations of global pose optimization, which suffers from local minima and requires good initial poses [S2]. The method leverages the temporal coherence of video data [S2].

## Sources
[S1] ZeroGS: Training 3D Gaussian Splatting from Unposed Images, https://doi.org/10.48550/arxiv.2411.15779, fetched 2026-10-26 (unofficial)
[S2] ICON: Incremental CONfidence for Joint Pose and Radiance Field Optimization, https://doi.org/10.1109/cvpr52733.2024.00517, fetched 2026-10-26
[S3] Incremental Joint Learning of Depth, Pose, and Implicit Scene Representation on Monocular Camera in Large-Scale Scenes, https://doi.org/10.1109/tase.2025.3617654, fetched 2026-10-29
