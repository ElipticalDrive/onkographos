# CBARF

Also called: CBARF, Cascaded BA

## Overview
CBARF is a 3D reconstruction method that optimizes imperfect camera poses using a cascaded bundle-adjustment (BA) structure to progressively refine poses and rectify erroneous ones [S1]. It builds on [BARF](6872-barf-bundle-adjusting-neural-radiance-fields.md) by introducing a coarse-to-fine optimization strategy with distinct modulation factors for each stage, allowing it to handle larger initial pose errors (e.g., 30.4° rotation deviation) where standard BARF may overfit [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of images with imperfect camera poses $\hat{P}$, a NeRF backbone (BARF), and scene geometry parameters [S1].
- **Outputs:** Refined camera poses $Q$, a trained NeRF model, and optionally synthetic images via a Voxel Grid Module (DVGO) [S1].

**Data Structures:**
- **Camera Pose:** Represented as $P \in \mathbb{R}^{3 \times 4}$, defined as $P = [R_p \ t_p]$ where $R_p \in SO(3)$ and $t_p \in \mathbb{R}^3$ [S1]. The last row of the standard $4 \times 4$ projection matrix is dropped [S1].
- **Noise/Refinement Parameters:** Represented in the Lie algebra $\mathfrak{se}(3)$ as $n \in \mathbb{R}^6$ [S1].
- **Transform Matrix:** Derived from $\mathfrak{se}(3)$ parameters, represented as $N = [R_n \ t_n]$ [S1].

**Numbered Steps:**

1.  **Simulate Imperfect Poses (Initialization):**
    Generate noise $n \in \mathfrak{se}(3)$ from a 6-dimensional random normal distribution:
    $$ n = 0.35 \mathcal{N}(0, I) \in \mathbb{R}^6 $$
    [S1]
    Transform $n$ into a camera transform matrix $N = [R_n \ t_n]$ [S1]. Compose this with the reference pose $P = [R_p \ t_p]$ to obtain the imperfect pose $\hat{P}$:
    $$ \hat{P} = [R_p R_n \ t_p + t_n] $$
    [S1]
    *Note:* The coefficient 0.35 corresponds to an average deviation of 30.4° in rotation and 0.56 in translation [S1].

2.  **Define Refined Pose Composition:**
    Optimize camera poses by training camera refine parameters $p \in \mathfrak{se}(3)$ [S1]. Convert $p$ into a transform matrix $P_r = [R_r \ t_r]$ [S1]. Compose $P_r$ with $\hat{P}$ to obtain the refined pose $Q$:
    $$ Q = [R_p R_n R_r \ t_p + t_n + t_r] $$
    [S1]

3.  **Cascaded Optimization Structure:**
    Use a coarse-to-fine structure with compact BA modules, using the BARF network as the backbone [S1]. The process consists of three stages: Coarse, Recursive, and Fine [S1].

4.  **Learning Rate Schedule:**
    Calculate the learning rate using exponential interpolation with a modulation factor $\alpha$ to bias learning towards the initial high value [S1]. The modulation factors for the three stages are:
    - Coarse stage: $\alpha = 10.0$
    - Recursive stage: $\alpha = 3.0$
    - Fine stage: $\alpha = 1.0$
    [S1]
    A factor of 1.0 corresponds to the original exponential learning rate used in BARF [S1].

5.  **Handling Incomplete Poses (Optional):**
    For incomplete camera poses, randomly drop 10% of poses to form a test set $F$, assigning an initial pose $P_{ini}$ to $F$ [S1]. The remaining images form the training set $T$ [S1]. Jointly optimize poses for both $T$ and $F$ during training [S1].

6.  **Evaluation Alignment:**
    Use Procrustes analysis to align optimized poses with ground-truth before evaluation to remove global offset [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch
from scipy.spatial.transform import Rotation as R

def se3_exp(v):
    """Convert 6D se(3) vector to 3x4 transform matrix [R t]. [G]"""
    omega, t = v[:3], v[3:]
    return np.hstack([R.from_rotvec(omega).as_matrix(), t.reshape(3, 1)])

def compose_poses(P_ref, N):
    """Compose reference pose P_ref [3x4] with transform N [3x4]. [G]"""
    R_p, t_p = P_ref[:, :3], P_ref[:, 3:]
    R_n, t_n = N[:, :3], N[:, 3:]
    return np.hstack([R_p @ R_n, t_p + t_n])

def procrustes_align(A, B):
    """Align set A to set B using Procrustes analysis. [G]"""
    A_c, B_c = A - np.mean(A, axis=0), B - np.mean(B, axis=0)
    U, _, Vt = np.linalg.svd(A_c.T @ B_c)
    R_opt = Vt.T @ U.T
    if np.linalg.det(R_opt) < 0: Vt[-1, :] *= -1; R_opt = Vt.T @ U.T
    return (B - np.mean(B, axis=0)) @ R_opt.T + np.mean(A, axis=0)

def sample_rays_at_res(images, poses, res):
    """Sample rays and target colors at reduced resolution. [G]"""
    # Standard ray marching implementation omitted for brevity; 
    # returns rays (N, 3) and targets (N, 3) tensors.
    pass

def cbarf_optimize(images, ref_poses, stages_config, incomplete=False):
    """
    images: list of image tensors
    ref_poses: list of 3x4 matrices [Rp tp]
    stages_config: list of dicts {'iters': int, 'modulation': float, 'res': int}
    """
    # 1. Simulate imperfect poses
    noise = 0.35 * np.random.normal(0, 1, 6)
    N = se3_exp(noise)
    imperfect_poses = [compose_poses(P, N) for P in ref_poses]

    # 2. Handle incomplete poses
    if incomplete:
        test_indices = np.random.choice(len(images), size=int(0.1 * len(images)), replace=False)
        train_set_T = [images[i] for i in range(len(images)) if i not in test_indices]
        current_poses = imperfect_poses
    else:
        train_set_T = images
        current_poses = imperfect_poses

    # 3. Cascaded optimization
    nerf = torch.nn.Sequential(torch.nn.Linear(3, 256), torch.nn.ReLU(), torch.nn.Linear(256, 3)) # [G]
    optimizer = torch.optim.Adam(list(nerf.parameters()), lr=1e-3)

    for stage in stages_config:
        iters = stage['iters']
        mod = stage['modulation']
        res = stage['res']
        for step in range(iters):
            rays, targets = sample_rays_at_res(train_set_T, current_poses, res)
            pred = nerf(rays)
            loss = torch.nn.functional.mse_loss(pred, targets)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            # LR schedule with modulation
            current_lr = 1e-3 * mod * np.exp(-step / iters)
            for pg in optimizer.param_groups: pg['lr'] = current_lr
            # Update poses with refined parameters p
            refined_p = optimizer.param_groups[-1]['params'][0].detach().numpy()
            Pr = se3_exp(refined_p)
            current_poses = [compose_poses(P, Pr) for P in current_poses]

    # 4. Evaluation alignment
    aligned_poses = procrustes_align(current_poses, ref_poses)
    return aligned_poses, nerf
```

**Parameters:**
- **Noise Coefficient:** 0.35 (for simulation) [S1].
- **Iterations:** 20k for Coarse and Recursive stages; 200k for Fine stage [S1].
- **Image Resolution:** 400×400 during pose optimization (half of 800×800 training resolution) [S1].
- **Modulation Factors:** 10.0 (Coarse), 3.0 (Recursive), 1.0 (Fine) [S1].

**Edge Cases and Pitfalls:**
- **Overfitting:** Standard BARF may encounter overfitting before completing optimization with large initial errors; CBARF mitigates this via the cascaded structure [S1].
- **Global Offset:** Optimized poses may have a misalignment with the ground-truth coordinate system [S1]. Procrustes analysis is required for evaluation but does not correct the pose during optimization [S1].
- **Incomplete Poses:** Randomly dropping 10% of poses requires joint optimization of training and test sets, using rendering quality as an indirect metric for pose error in the absence of ground-truth for the test set [S1].

## Operating characteristics
- **Total Iterations:** 240k per scene (20k coarse + 20k recursive + 200k fine) [S1].
- **Memory Usage:** Uses the BARF backbone, which stores the NeRF MLP and camera parameters [S1]. Image resolution is reduced to 400×400 during optimization to improve efficiency [S1].
- **Performance:** Successfully optimizes poses initialized with 30.4° average rotation deviation, whereas standard BARF overfits before completion [S1]. Rendering quality is comparable to ground-truth images [S1].
- **Strengths:** Handles large initial pose errors; progressive refinement prevents overfitting.
- **Limitations:** Requires careful tuning of modulation factors; Procrustes alignment is needed for evaluation, implying the absolute pose is not directly optimized to a global frame without post-processing [S1].

## Background
CBARF was proposed in 2024 by authors publishing in the IEEE Transactions on Multimedia [S1]. It addresses the limitations of [BARF](6872-barf-bundle-adjusting-neural-radiance-fields.md) in handling imperfect camera poses by introducing a cascaded BA structure [S1]. The method is used for 3D reconstruction from imperfect camera poses, including scenarios with incomplete pose data [S1].

## Sources
[S1] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
