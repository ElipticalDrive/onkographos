# Direct Matching Alignment Loss

Also called: Direct Alignment

## Overview
Direct Matching Alignment Loss ($L_{align}$) is a regularization term used during NeRF training to align a reference image with a set of surrogate views, thereby reducing overfitting and improving pose estimation accuracy [S1, unofficial]. Unlike standard rendering losses that only compare rendered pixels against ground truth, this method enforces geometric consistency between different camera viewpoints by projecting 3D points from a reference view into surrogate views and comparing their appearance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Reference image $I_r$ with pixel domain $\Omega(I_r)$, camera intrinsics $K$ ($3 \times 3$), rotation $R_r$ ($3 \times 3$), translation $\mathbf{t}_r$ ($3 \times 1$); set of surrogate views $\mathcal{N}_r$ with poses $(K, R_s, \mathbf{t}_s)$; NeRF model parameters $\Theta$ for depth estimation; depth threshold $\delta_d$; Huber parameter $\epsilon$ [S1, unofficial].
*   **Outputs:** Scalar loss value $L_{align}$ [S1, unofficial].

**Data Structures:**
*   Pixel coordinates $\mathbf{u} = [u, v]^T$ [S1, unofficial].
*   Estimated depth $\bar{d}(\mathbf{u})$ (scalar) [S1, unofficial].
*   Image intensities $\mathcal{I}_r(\mathbf{u})$, $\mathcal{I}_s(\mathbf{u}_s)$ (RGB vectors) [S1, unofficial].

**Numbered Steps:**
1.  **Sample Reference Frame and Points:** Randomly select a reference frame $r$ from the training set and pixel coordinates $\mathbf{u} \in \Omega(I_r)$ [S1, unofficial]. Unless specified, the entire training set serves as $\mathcal{N}_r$ [S1, unofficial].
2.  **Estimate Reference Depth:** Compute $\bar{d}(\mathbf{u})$ as the weighted sum of sampled depths $d_{sample}$ along the ray, weighted by the ray termination distribution $h$ [S1, unofficial]:
    $$ \bar{d}(\mathbf{u}) = \sum _{d_{sample}} h(\mathbf{u},d_{sample}) d_{sample} $$
3.  **Project to Surrogate View:** For each surrogate view $s \in \mathcal{N}_r$, compute the corresponding pixel coordinate $\mathbf{u}_s$ by back-projecting $\mathbf{u}$ to 3D using $\bar{d}(\mathbf{u})$, transforming to the surrogate camera frame, and projecting back to 2D [S1, unofficial]:
    $$ \mathbf{u}_s = \Pi \left( K R_s \left( R_{r}^{-1} K^{-1} [\mathbf{u}, 1]^T \bar {d}(\mathbf{u}) - \mathbf{t}_r \right) + \mathbf{t}_s \right) $$
    where $\Pi ([X,Y,Z]^T) = [X/Z, Y/Z]^T$ [S1, unofficial].
4.  **Check Occlusion (Visibility Indicator):** Determine visibility using $\mathcal{V}(\mathbf{u}_s,\mathbf{u})$ [S1, unofficial]:
    $$ \mathcal{V}(\mathbf{u}_s,\mathbf{u}) = \mathbbm {1}[ \bar {d}(\mathbf{u}_s) < \mathcal {T}_s (\mathbf{u}, \bar {d}(\mathbf{u})) - \delta _d ] $$
    where $\mathcal {T}_s (\mathbf{u}, \bar {d}(\mathbf{u}))$ is the depth of the projected 3D point in the surrogate camera frame, calculated as the Z-component of the transformed point [S1, unofficial]:
    $$ \mathcal {T}_s (\mathbf{u}, \bar {d}(\mathbf{u})) = \omega \left( K \left( R_s \left( R_{r}^{-1} K^{-1} [\mathbf{u}, 1]^T \bar {d}(\mathbf{u}) - \mathbf{t}_r \right) + \mathbf{t}_s \right) \right) $$
    and $\omega ([X,Y,Z]^T) = Z$ [S1, unofficial].
5.  **Compute Alignment Loss:** Sum the Huber penalty between $\mathcal{I}_{r}(\mathbf{u})$ and $\mathcal{I}_s (\mathbf{u}_s)$, weighted by visibility [S1, unofficial]:
    $$ L_{align} = \sum _{r=1}^{F} \sum _{s \in \mathcal {N}_r} \sum _{\mathbf {u} \in \Omega (I_r)} \mathcal {V}(\mathbf {u}_s,\mathbf {u}) \| \mathcal {I}_s (\mathbf {u}_s) - \mathcal {I}_{r}(\mathbf {u}) \|_\epsilon $$
    where $\| \cdot \|_\epsilon$ is the Huber penalty [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_direct_alignment_loss(I_r, K, R_r, t_r, surrogate_views, nerf_model, delta_d, epsilon):
    """Computes Direct Matching Alignment Loss per [S1, unofficial]."""
    total_loss = 0.0
    H, W = I_r.shape[:2]
    # Step 1: Sample pixels u on reference image
    for v in range(H):
        for u in range(W):
            # Step 2: Estimate depth d(u)
            d_samples = nerf_model.get_sample_depths((u, v), (K, R_r, t_r))
            h_values = nerf_model.get_termination_distribution((u, v), (K, R_r, t_r))
            d_est = sum(h * d for h, d in zip(h_values, d_samples))
            
            # Step 3: Project to surrogate view
            u_hom = [u, v, 1.0]
            # P_cam = inv(K) @ u_hom * d_est
            P_cam = vec_scale(mat_vec_mul(inv_3x3(K), u_hom), d_est)
            # P_world = R_r.T @ (P_cam - t_r)
            P_world = vec_add(vec_scale(R_r.T, P_cam), vec_scale(-R_r.T, t_r))
            
            for I_s, K_s, R_s, t_s in surrogate_views:
                # Transform to surrogate camera frame
                P_s_cam = vec_add(vec_scale(R_s, P_world), t_s)
                if P_s_cam[2] <= 0: continue # Behind camera
                
                # Project to u_s
                u_s = [P_s_cam[0] / P_s_cam[2], P_s_cam[1] / P_s_cam[2]]
                if not is_in_bounds(u_s, I_s.shape): continue
                
                # Step 4: Occlusion Reasoning
                # T_s is depth of projected point in surrogate frame
                T_s = P_s_cam[2]
                # Query NeRF at u_s in view s
                d_samples_s = nerf_model.get_sample_depths(u_s, (K_s, R_s, t_s))
                h_values_s = nerf_model.get_termination_distribution(u_s, (K_s, R_s, t_s))
                d_est_s = sum(h * d for h, d in zip(h_values_s, d_samples_s))
                
                vis = 1.0 if d_est_s < (T_s - delta_d) else 0.0
                if vis > 0:
                    # Step 5: Compute Loss
                    I_s_u_s = bilinear_sample(I_s, u_s)
                    I_r_u = I_r[v, u]
                    diff = I_s_u_s - I_r_u
                    total_loss += vis * huber_penalty(diff, epsilon)
    return total_loss

def huber_penalty(diff, epsilon):
    abs_diff = abs(diff)
    return epsilon * (abs_diff - 0.5 * epsilon) if abs_diff > epsilon else 0.5 * diff**2

def bilinear_sample(img, uv):
    u, v = uv
    u0, v0 = int(u), int(v)
    u1, v1 = min(u0 + 1, img.shape[1]-1), min(v0 + 1, img.shape[0]-1)
    w_u, w_v = u - u0, v - v0
    top = img[v0, u0]*(1-w_u) + img[v0, u1]*w_u
    bot = img[v1, u0]*(1-w_u) + img[v1, u1]*w_u
    return top*(1-w_v) + bot*w_v

def is_in_bounds(uv, shape):
    return 0 <= uv[0] < shape[1] and 0 <= uv[1] < shape[0]

def vec_scale(M, v):
    return [sum(M[i][j]*v[j] for j in range(3)) for i in range(3)]

def vec_add(v1, v2):
    return [a+b for a,b in zip(v1, v2)]

def inv_3x3(M):
    # Standard 3x3 matrix inversion
    inv_det = 1.0 / (M[0][0]*(M[1][1]*M[2][2]-M[1][2]*M[2][1]) - M[0][1]*(M[1][0]*M[2][2]-M[1][2]*M[2][0]) + M[0][2]*(M[1][0]*M[2][1]-M[1][1]*M[2][0]))
    return [
        [(M[1][1]*M[2][2]-M[1][2]*M[2][1])*inv_det, (M[0][2]*M[2][1]-M[0][1]*M[2][2])*inv_det, (M[0][1]*M[1][2]-M[0][2]*M[1][1])*inv_det],
        [(M[1][2]*M[2][0]-M[1][0]*M[2][2])*inv_det, (M[0][0]*M[2][2]-M[0][2]*M[2][0])*inv_det, (M[0][2]*M[1][0]-M[0][0]*M[1][2])*inv_det],
        [(M[1][0]*M[2][1]-M[1][1]*M[2][0])*inv_det, (M[0][1]*M[2][0]-M[0][0]*M[2][1])*inv_det, (M[0][0]*M[1][1]-M[0][1]*M[1][0])*inv_det]
    ]

def mat_vec_mul(M, v):
    return [sum(M[i][j]*v[j] for j in range(3)) for i in range(3)]
```

**Parameters:**
*   $\delta_d$: Depth threshold for occlusion reasoning. Value not given in the sources.
*   $\epsilon$: Huber penalty parameter. Value not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The projected pixel $\mathbf{u}_s$ is defined as the projection of the intersection point of the two rays by definition, ensuring geometric consistency for the occlusion check [S1, unofficial]. Floating-point inaccuracies in depth estimation or projection can lead to incorrect occlusion decisions [G].
*   **Boundaries:** Projected pixels $\mathbf{u}_s$ may fall outside the image boundaries of the surrogate view; these should be handled by skipping the pixel or assigning zero visibility [G].
*   **Degenerate Input:** If the estimated depth $\bar{d}(\mathbf{u})$ is zero or negative, the back-projection is invalid; such rays should be skipped [G].

## Subtopics

### Occlusion Reasoning
This variant explicitly models visibility between views using a depth-based indicator [S1, unofficial]. It calculates the depth of the 3D point projected from the reference view into the surrogate view and compares it with the NeRF's estimated depth at the corresponding pixel in the surrogate view [S1, unofficial]. If the surrogate view's surface is closer than the projected point (minus a threshold $\delta_d$), the point is considered occluded and excluded from the loss [S1, unofficial]. This prevents the loss from penalizing the model for correct occlusions where the reference point is hidden in the surrogate view [S1, unofficial].

### Pseudo-Augmented Reality Fusion
This variant aligns the real-time camera viewpoint with the reconstructed model's observation vector to enhance spatial coherence and immersion [S2]. The alignment objective is defined as the squared L2 norm of the difference between perspective projections of the camera viewpoint $V_{cam}$ and the NeRF observation vector $V_{NeRF}$ [S2]:
$$ L_{align} = \| \text{Proj}(V_{cam}) - \text{Proj}(V_{NeRF}) \|^2 $$
where $\text{Proj}(\cdot)$ is the perspective projection function [S2]. $V_{cam}$ is the real-time camera viewpoint from the user device, and $V_{NeRF}$ is the observation vector of the reconstructed model [S2]. This optimization improves spatial coherence between the user viewpoint and the virtual model, particularly for mobile and AR-enabled devices [S2].

## Operating characteristics
*   **Time Complexity:** The loss calculation is $O(F \cdot |\mathcal{N}_r| \cdot |\Omega(I_r)|)$, where $F$ is the number of reference frames, $|\mathcal{N}_r|$ is the number of surrogate views, and $|\Omega(I_r)|$ is the number of pixels sampled [G]. Since the entire training set is often used as surrogate views, this can be computationally expensive [S1, unofficial].
*   **Memory Complexity:** Requires storing the NeRF model and the training images, with no additional large data structures beyond the ray samples [G].
*   **Measured Performance:** On the Fern sequence, using alignment increased PSNR from 23.06 to 24.50 and SSIM from 0.72 to 0.75 [S1, unofficial]. On the Trex sequence, it increased PSNR from 21.82 to 21.90 and SSIM from 0.76 to 0.78 [S1, unofficial].
*   **Strengths:** Reduces overfitting to training data, improves pose estimation, and enforces geometric consistency across views [S1, unofficial].
*   **Limitations:** Computationally intensive due to cross-view projections and depth queries; relies on accurate depth estimation from the NeRF [S1, unofficial].
*   **When to use:** When training NeRFs on datasets with significant overfitting or when pose estimation accuracy is critical [S1, unofficial].
*   **When not to use:** When computational resources are limited or when the dataset is very large, making the $O(N^2)$ cross-view comparison prohibitive [G].

## Background
The Direct Matching Alignment Loss was proposed by R. Garg et al. in the paper "Direct Alignment for Robust NeRF Learning" presented at ACCV 2024 [S1, unofficial]. The work addresses the issue of NeRFs overfitting to training data, even with multiple views, by introducing a direct alignment term that encourages consistency between different camera viewpoints [S1, unofficial]. The method builds on the standard NeRF rendering pipeline and integrates with the training loop using SGD [S1, unofficial].

## Sources
[S1] ACCV 2024 Open Access Repository, https://openaccess.thecvf.com/content/ACCV2024/html/Garg_Direct_Alignment_for_Robust_NeRF_Learning_ACCV_2024_paper.html, fetched 2026-10-29 (unofficial)
[S2] High fidelity three dimensional reconstruction and interactive visualization of public art sculptures using neural radiance fields, https://link.springer.com/article/10.1007/s44163-026-02441-3, fetched 2026-10-26
