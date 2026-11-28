# LU-NeRF

Also called: Local Unposed NeRF, Locally Unlocalized NeRF

## Overview
LU-NeRF is a method for few-shot local unposed NeRF that jointly estimates camera poses and scene representation from a small subset of images (mini-scene) without requiring accurate pose initialization or priors, specifically designed to handle mirror symmetry ambiguities [S1, S2]. It employs a local-to-global learning framework that first optimizes overlapping mini-scenes using lightweight models and then synchronizes them into a global reference frame [S1, S2]. Unlike BARF, LU-NeRF does not require coarse pose initialization close to the true poses and does not use positional encoding or view dependency, making it lighter and faster for rough local estimation [S1].

## How it works
**Inputs:** A set of images from a scene [S1].
**Outputs:** Global camera poses and a global radiance field representation [S1, S2].

**Data Structures:**
*   **Mini-scene:** An overlapping subset of images, typically containing 3 to 5 images [S1].
*   **LU-NeRF-1 Model:** A lightweight NeRF model consisting of small Multi-Layer Perceptrons (MLPs) [S1].
    *   **Network Architecture:** Small MLPs [S1]. Positional encoding is eliminated [S1]. View dependency is eliminated [S1].
    *   **Parameters:** $\theta$ represents the model parameters (weights of the MLPs) and camera poses [S1].
*   **Camera Poses:** Represented as relative parameters [S1]. One camera pose in the mini-scene is frozen to the identity matrix to resolve global translation/rotation ambiguity [S1]. All other poses are optimized as relative parameters [S1].
*   **Ray Samples:** $R$ is a set of ray samples [S1].
*   **Depth Rendering:** $d_\theta(r)$ is the depth rendering function for a ray $r$ [S1].

**Steps:**

1.  **Partitioning:** Partition the scene into overlapping subsets of images called mini-scenes, typically containing three to five images [S1].
2.  **Initialization:** For each mini-scene, initialize all camera poses close to the identity matrix, leveraging the assumption that images in a mini-scene are spatially close [S1]. Freeze one camera pose in the mini-scene to the identity matrix [S1].
3.  **Local Optimization:** For each mini-scene, optimize the neural scene representation and camera poses jointly using the lightweight LU-NeRF-1 model [S1].
    *   The optimization minimizes a loss function comprising the photometric $\ell_2$ loss and a total variation regularization term on predicted depths [S1].
    *   The total variation loss is defined as:
        $$ \mathcal{L}_{TV} = \frac{1}{|R|} \sum_{r \in R} \sum_{i=1}^{K} \sum_{j=1}^{K} \left( \| d_\theta(r_{i,j}) - d_\theta(r_{i,j+1}) \|_2^2 + \| d_\theta(r_{i,j}) - d_\theta(r_{i+1,j}) \|_2^2 \right) $$
        where $R$ is the set of ray samples, $d_\theta(r)$ is the depth rendering function for ray $r$, $\theta$ are the model parameters and camera poses, $K$ is the patch size, and $(i, j)$ is the pixel index [S1].
    *   Strong regularization is applied to the radiance field optimization [S1].
4.  **Global Synchronization:** Bring the poses of these mini-scenes into a global reference frame through a robust pose synchronization step [S1, S2].
5.  **Global Refinement:** Perform fine-grained optimization of the global poses and scenes [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lu_nerf_training(images):
    # Step 1: Partition scene into overlapping mini-scenes (3-5 images each) [S1]
    mini_scenes = [images[i:i+5] for i in range(0, len(images)-4, 2)]

    local_poses = {}
    local_models = {}

    # Step 2 & 3: Local Optimization for each mini-scene
    for mini_scene in mini_scenes:
        # Initialize poses close to identity [S1]
        poses = [IdentityMatrix() for _ in mini_scene]
        # Freeze one pose to identity to resolve ambiguity [S1]
        poses[0] = IdentityMatrix()

        # Initialize lightweight NeRF model (small MLPs, no positional encoding) [S1]
        model = LightweightNeRFMLP()

        # Jointly optimize poses and scene representation [S1]
        # Loss = Photometric L2 + Total Variation of Depth [S1]
        # Note: iteration count not given in sources [G]
        for iteration in range(1000):
            rays = generate_rays(mini_scene, poses)
            colors, depths = model.forward(rays)

            # Photometric L2 loss
            loss_photometric = sum((c - t)**2 for c, t in zip(colors, mini_scene.target_colors)) / len(colors)

            # Total Variation loss on depth [S1]
            loss_tv = 0.0
            count = 0
            for r in rays:
                for i in range(K):
                    for j in range(K):
                        d_curr = depths[r][i, j]
                        d_right = depths[r][i, j+1] if j + 1 < K else d_curr
                        d_down = depths[r][i+1, j] if i + 1 < K else d_curr
                        loss_tv += (d_curr - d_right)**2 + (d_curr - d_down)**2
                        count += 1
            loss_tv /= max(count, 1)

            total_loss = loss_photometric + loss_tv
            model.parameters, poses = compute_gradients_and_update(total_loss, model.parameters, poses)

        local_poses[mini_scene.id] = poses
        local_models[mini_scene.id] = model

    # Step 4: Global Synchronization [S1, S2]
    global_poses = synchronize_poses(local_poses)

    # Step 5: Global Refinement [S2]
    global_model = GlobalNeRFMLP()
    # Note: iteration count not given in sources [G]
    for iteration in range(2000):
        rays = generate_rays(images, global_poses)
        colors, _ = global_model.forward(rays)
        loss = sum((c - t)**2 for c, t in zip(colors, images.target_colors)) / len(colors)
        global_model.parameters, global_poses = compute_gradients_and_update(loss, global_model.parameters, global_poses)

    return global_poses, global_model

# Self-contained helper definitions
def IdentityMatrix():
    return [[1, 0, 0, 0], [0, 1, 0, 0], [0, 0, 1, 0], [0, 0, 0, 1]]

def generate_rays(scene, poses):
    rays = []
    for cam_idx, cam_pose in enumerate(poses):
        for y in range(scene.height):
            for x in range(scene.width):
                rays.append((cam_idx, x, y))
    return rays

def compute_gradients_and_update(loss, params, poses):
    # Standard backpropagation and gradient descent step
    # Returns updated parameters and poses
    return params, poses

def synchronize_poses(local_poses):
    # Robust pose synchronization into a global reference frame [S1, S2]
    # Implementation details not provided in sources [G]
    return {}
```

**Parameters:**
*   **Mini-scene size:** Typically 3 to 5 images [S1].
*   **Patch size ($K$):** Used in the total variation loss calculation [S1]. Not given in the sources.
*   **Regularization:** Strong regularization is applied, specifically photometric $\ell_2$ loss and total variation of predicted depths [S1].

**Edge Cases and Pitfalls:**
*   **Mirror Symmetry Ambiguity:** Two distinct 3D scenes and camera configurations produce similar images under affine projection. The model must reconcile this ambiguity inherent in the local unposed setting [S1].
*   **Initialization:** The method relies on initializing poses close to the identity matrix, assuming images in a mini-scene are spatially close [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources [G].
*   **Measured Performance:** Experimental results show that LU-NeRF outperforms previous techniques on the tasks of camera pose estimation and new view synthesis [S2]. Evaluation on the LLFF dataset shows higher scores in PSNR, SSIM, and LPIPS metrics compared to methods like BARF [S2].
*   **Scaling:** The local-to-global framework allows handling large scenes by breaking them into manageable mini-scenes [S1].
*   **Strengths:** Does not require coarse pose initialization close to the true poses [S1]. Does not use positional encoding or view dependency, making it lighter and faster for rough local estimation [S1]. Succeeds even in the absence of a priori knowledge of the pose distribution [S2].
*   **Limitations:** Jointly optimizing camera poses and scene representation over a full scene is difficult and underconstrained [S1].
*   **When to use:** When accurate pose initialization is not available and computational resources are limited [S1].

## Background
LU-NeRF was introduced in 2023 [S1]. It addresses the challenges of few-shot novel view synthesis and optimizing unknown poses in a combined setting [S1]. The method builds on the idea of breaking up a large scene into mini-scenes to overcome the non-convexity of global pose optimization without accurate initialization [S1]. It is related to methods like BARF, which also optimizes camera poses but requires coarse initialization and uses positional encoding [S1].

## Sources
[S1] LU-NeRF: Scene and Pose Estimation by Synchronizing Local Unposed NeRFs, https://doi.org/10.1109/iccv51070.2023.01679, fetched 2026-10-26
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
