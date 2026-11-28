# Data Augmentation

Also called: Random Rotation Augmentation

## Overview
Data augmentation generates diverse training samples from synthetic sensor data to improve model robustness and generalization [S1, unofficial][S2, S3]. It addresses limited or biased training distributions by applying geometric transformations and inserting synthetic objects [S1, unofficial][S3]. It builds on [Synthetic Dataset Generation](9337-synthetic-dataset-generation.md) by adding stochastic variation to poses, signs, and scene content [S1, unofficial][S2, S3].

## How it works
The method consists of two primary strategies: random rotation of existing data and dataset-specific sign balancing [S1, unofficial][S2].

### Inputs and Outputs
*   **Inputs:** Point cloud data $P$, camera poses $T$, sensor measurements $Z_t = \{v_i, r_{t,i}\}$, and optional CAD models [S1, unofficial][S2, S3].
*   **Outputs:** Augmented dataset $D$ containing samples $(p_j, v_j, f^*_j, i^*_j, s^*_j)$ [S2].

### Data Structures
The dataset sample structure is defined as:
*   $p_j \in \mathbb{R}^3$: Ray origin [S2].
*   $v_j \in S^2$: Ray direction [S2].
*   $f^*_j \in \mathbb{R}$: SDDF measurement [S2].
*   $i^*_j \in \{-1, 1\}$: Intersection indicator [S2].
*   $s^*_j \in \{-1, 1\}$: Sign indicator [S2].

### Steps

#### 1. Random Rotation Augmentation
To obtain diverse training samples, random rotations are applied to both point cloud data and camera poses [S1, unofficial].

1.  Generate random rotation angles $\theta_x, \theta_y, \theta_z$ [S1, unofficial].
2.  Construct rotation matrices $R_x(\theta_x)$, $R_y(\theta_y)$, $R_z(\theta_z)$ [G].
3.  Compute total rotation $R_{total} = R_z(\theta_z) R_y(\theta_y) R_x(\theta_x)$ [G].
4.  Apply $R_{total}$ to all points in the point cloud [S1, unofficial].
5.  Apply $R_{total}$ to the rotation component of all camera poses [S1, unofficial].

**Parameters:**
*   $\theta_x \in [-\pi/64, \pi/64]$ [S1, unofficial].
*   $\theta_y \in [-\pi/64, \pi/64]$ [S1, unofficial].
*   $\theta_z \in [-\pi, \pi]$ [S1, unofficial].

#### 2. Dataset Generation and Sign Balancing (SDDF)
For Signed Directional Distance Function (SDDF) models, the initial dataset is unbalanced in sign indicators [S2].

1.  **Generate Positive Samples:**
    For each ray measurement $(v_i, r_{t,i})$ from sensor position $p_t$ and orientation $R_t$:
    *   $p_j = p_t$
    *   $v_j = R_t v_i$
    *   $f^*_j = r_{t,i}$
    *   $i^*_j = 1$
    *   $s^*_j = 1$
2.  **Generate Negative Samples:**
    For each positive sample $(p_j, v_j, f^*_j, i^*_j, s^*_j)$, generate a corresponding negative sample:
    *   $p' = p_j + (f^*_j + \epsilon) v_j$
    *   $v'_j = v_j$
    *   $f'^*_j = -\epsilon$
    *   $i'^*_j = 1$
    *   $s'^*_j = -1$
    *   Where $\epsilon > 0$ is a small offset [S2].

**Pitfalls:**
*   The initial dataset has all $s^*_j = 1$ and $f^*_j \ge 0$, causing sign imbalance [S2].
*   Intersection indicator augmentation ($i^*_j = -1$) is not performed because generating non-intersecting rays is inefficient and does not improve reconstruction results [S2].

#### 3. Object Insertion Augmentation
For downstream tasks, synthetic objects are inserted into scenes to enhance diversity [S3].

1.  Select a CAD model (e.g., from a library of 283 car models and 30 construction vehicles) [S3].
2.  Randomize the object's appearance (e.g., paint color from 60 diverse options) [S3].
3.  Apply Gaussian perturbation to the object's orientation with $\sigma = 15^\circ$ [S3].
4.  Insert the object into the scene.
5.  Handle occlusion naively by comparing the depth map of the scene with the depth map of the inserted object, removing the previous occlusion check [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def random_rotation_augmentation(point_cloud, camera_poses, seed=None):
    """Applies random rotations to point cloud and camera poses [S1, unofficial]."""
    if seed is not None:
        np.random.seed(seed)
    theta_x = np.random.uniform(-np.pi / 64, np.pi / 64)
    theta_y = np.random.uniform(-np.pi / 64, np.pi / 64)
    theta_z = np.random.uniform(-np.pi, np.pi)
    Rx = np.array([[1, 0, 0], [0, np.cos(theta_x), -np.sin(theta_x)], [0, np.sin(theta_x), np.cos(theta_x)]])
    Ry = np.array([[np.cos(theta_y), 0, np.sin(theta_y)], [0, 1, 0], [-np.sin(theta_y), 0, np.cos(theta_y)]])
    Rz = np.array([[np.cos(theta_z), -np.sin(theta_z), 0], [np.sin(theta_z), np.cos(theta_z), 0], [0, 0, 1]])
    R_total = Rz @ Ry @ Rx
    augmented_pc = (R_total @ point_cloud.T).T
    augmented_poses = []
    for pose in camera_poses:
        new_rot = R_total @ pose[:3, :3]
        new_pos = R_total @ pose[:3, 3]
        new_pose = np.eye(4)
        new_pose[:3, :3] = new_rot
        new_pose[:3, 3] = new_pos
        augmented_poses.append(new_pose)
    return augmented_pc, augmented_poses

def generate_sddf_dataset(sensor_data, epsilon=1e-3):
    """Generates SDDF dataset with sign balancing [S2].
    sensor_data: list of tuples (pt, Rt, vi, rt_i)"""
    dataset = []
    for pt, Rt, vi, rt_i in sensor_data:
        pj = pt
        vj = Rt @ vi
        f_star = rt_i
        i_star = 1
        s_star = 1
        dataset.append((pj, vj, f_star, i_star, s_star))
        p_prime = pj + (f_star + epsilon) * vj
        f_prime = -epsilon
        s_prime = -1
        dataset.append((p_prime, vj, f_prime, i_star, s_prime))
    return dataset

def insert_object_augmentation(scene_depth, object_vertices, object_color_options, orientation_sigma_deg=15):
    """Inserts an object into the scene with naive occlusion handling [S3]."""
    # Select random color from 60 options [S3]
    color = np.random.choice(object_color_options)
    # Apply Gaussian orientation perturbation with sigma = 15 degrees [S3]
    perturbation = np.random.normal(0, orientation_sigma_deg, 3)
    # Construct rotation from perturbation [G]
    Rx = np.array([[1, 0, 0], [0, np.cos(perturbation[0]), -np.sin(perturbation[0])], [0, np.sin(perturbation[0]), np.cos(perturbation[0])]])
    Ry = np.array([[np.cos(perturbation[1]), 0, np.sin(perturbation[1])], [0, 1, 0], [-np.sin(perturbation[1]), 0, np.cos(perturbation[1])]])
    Rz = np.array([[np.cos(perturbation[2]), -np.sin(perturbation[2]), 0], [np.sin(perturbation[2]), np.cos(perturbation[2]), 0], [0, 0, 1]])
    R_obj = Rz @ Ry @ Rx
    rotated_vertices = (R_obj @ object_vertices.T).T
    # Standard projection to render object depth [G]
    object_depth = np.zeros_like(scene_depth)
    # Naive occlusion handling: compare depth maps [S3]
    mask = object_depth < scene_depth
    composited_image = np.where(mask[:, :, np.newaxis], color, scene_depth)
    return composited_image
```

## Subtopics

### Dataset Generation and Augmentation
A procedure to convert sensor poses and range measurements into a dataset $D$ suitable for training the SDDF model, specifically addressing sign imbalance [S2]. It is chosen when training implicit signed distance functions that require balanced positive and negative samples [S2].

### Object Insertion Data Augmentation
An approach that inserts synthetic CAD models (cars, construction vehicles) into scenes with randomized colors and orientation perturbation ($\sigma = 15^\circ$) to enhance diversity for downstream tasks [S3]. It is chosen when the training data lacks sufficient variety in object types or appearances [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Improves model robustness to pose variations and sign ambiguities [S1, unofficial][S2]. Object insertion enhances generalization to unseen objects [S3].
*   **Limitations:** Intersection indicator augmentation is omitted due to inefficiency and lack of benefit [S2]. Naive occlusion handling may produce artifacts compared to physically accurate occlusion checks [S3].
*   **When to use:** When training data is limited or biased towards specific poses/signs [S1, unofficial][S2]. When the target application involves diverse object appearances or poses [S1, unofficial][S3].

## Background
Data augmentation in this context originated from the need to train robust neural scene representations from synthetic data [S1, unofficial][S2, S3]. Random rotation augmentation was introduced in the context of point cloud unsupervised pre-training using 3D Gaussian Splatting [S1, unofficial]. Sign balancing for SDDF models was developed to address the inherent bias in range sensor data where all samples initially have positive signs [S2]. Object insertion augmentation was proposed for neural light field estimation to enhance diversity in street scene reconstruction [S3].

## Sources
[S1] Point Cloud Unsupervised Pre-training via 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2411.18667, fetched 2026-10-26 (unofficial)
[S2] Learning Scene-Level Signed Directional Distance Function With Ellipsoidal Priors and Neural Residuals., https://doi.org/10.1109/tpami.2026.3688658, fetched 2026-10-26
[S3] Neural Light Field Estimation for Street Scenes with Differentiable Virtual Object Insertion, https://doi.org/10.1007/978-3-031-20086-1_22, fetched 2026-10-30
