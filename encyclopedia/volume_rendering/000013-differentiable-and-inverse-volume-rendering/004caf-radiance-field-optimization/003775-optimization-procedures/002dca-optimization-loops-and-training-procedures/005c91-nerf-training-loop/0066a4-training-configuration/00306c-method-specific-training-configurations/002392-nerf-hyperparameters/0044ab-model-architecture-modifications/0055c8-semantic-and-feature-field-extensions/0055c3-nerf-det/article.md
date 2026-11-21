# NeRF-Det

Also called: NeRF-then-Det

## Overview
NeRF-Det is a method for indoor 3D object detection using posed RGB images that explicitly models scene geometry as an opacity field by jointly training a NeRF branch with a 3D detection pipeline [S1]. Unlike "NeRF-then-Det" approaches (e.g., NeRF-RPN) that treat geometry estimation and detection as separate stages, NeRF-Det is a joint "NeRF-and-Det" paradigm that integrates multi-view geometry constraints from the NeRF branch into 3D detection through a shared geometry MLP [S1]. This allows the method to achieve state-of-the-art accuracy on multi-view indoor RGB-only 3D detection while enabling generalizable novel view synthesis on new scenes without per-scene optimization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $T$ posed RGB frames $I_i \in \mathbb{R}^{H_i \times W_i \times 3}$ with corresponding intrinsic matrix $K \in \mathbb{R}^{3 \times 3}$ and extrinsic matrix $R_i \in \mathbb{R}^{3 \times 4}$ for $i = 1, \dots, T$ [S1].
*   **Output:** 3D bounding boxes for object detection [S1]. Optionally, novel views and depth maps [S1].

**Data Structures:**
1.  **2D Feature Map ($F_i$):** Extracted from $I_i$ using a Feature Pyramid Network (FPN) [S1]. $F_i \in \mathbb{R}^{C \times H/4 \times W/4}$ [S1].
2.  **3D Grid:** A voxel grid of size $N_x \times N_y \times N_z$, where the z-axis denotes height and x/y denote horizontal dimensions [S1].
3.  **3D Volume Features ($V_i(p)$):** For each voxel center $p = (x, y, z)^T$, features are assigned from the 2D feature maps [S1].
4.  **Shared Geometry MLP (G-MLP):** A neural network that predicts the density field $\sigma(p)$ [S1].
5.  **Detection Head:** Consumes geometry-aware volume features to regress 3D bounding boxes [S1].

**Steps:**

1.  **2D Feature Extraction:** Pass each input image $I_i$ through an FPN backbone to extract high-resolution features $F_i \in \mathbb{R}^{C \times H/4 \times W/4}$ [S1].

2.  **3D Feature Volume Construction:**
    For each voxel center $p = (x, y, z)^T$ in the 3D grid:
    a. Project $p$ to view $i$ to obtain 2D coordinates. The projected homogeneous coordinates are computed as:
    $$
    \begin{pmatrix} u'_i, v'_i, d_i \end{pmatrix}^T = K' \times R_i \times \begin{pmatrix} \mathbf{p}, 1 \end{pmatrix}^T
    $$
    where $K'$ is the scaled intrinsic matrix accounting for feature map downsampling [S1].
    b. The pixel coordinates are $(u_i, v_i) = (u'_i/d_i, v'_i/d_i)$ [S1].
    c. Assign 3D features using nearest neighbor interpolation:
    $$
    V_i(p) = \text{interpolate}((u_i, v_i), F_i)
    $$
    d. **Boundary Handling:** If $p$ is projected outside the image boundary or behind the image plane ($d_i \le 0$), discard the feature and set $V_i(p) = 0$ [S1].

3.  **Multi-View Aggregation:**
    Aggregate features from all $M_p$ effective 2D projections (where $M_p$ is the number of views where $V_i(p) \neq 0$) by averaging:
    $$
    V_{\text{avg}}(p) = \frac{1}{M_p} \sum_{i=1}^{M_p} V_i(p)
    $$
    [S1].

4.  **Geometry-Aware Feature Modulation (NeRF Branch):**
    a. Feed the voxel coordinates $p$ into the shared G-MLP to predict the density field $\sigma(p)$ [S1].
    b. Transform the density field into an opacity field $\alpha(p)$ [S1]. The source text states: "transform the density field into an opacity field and multiply it with the volume features" [S1]. While the exact formula for $\alpha$ is not explicitly written as $\alpha = 1 - e^{-\sigma \delta}$ in the NeRF-Det section, it follows the standard volume rendering opacity definition implied by the context of NeRF [S1, 12995-feature-fields.md].
    c. Multiply the opacity with the volume features to reduce weights of empty space:
    $$
    V_{\text{geo}}(p) = \alpha(p) \cdot V_{\text{avg}}(p)
    $$
    [S1].

5.  **Detection:**
    Feed the geometry-aware volume features $V_{\text{geo}}(p)$ to the detection head for 3D bounding box regression [S1].

6.  **Inference:**
    Remove the NeRF branch (G-MLP and associated computations) to minimize overhead. The detection head operates on the pre-computed or efficiently computed volume features [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_det_forward(images, poses, intrinsics, voxel_grid, g_mlp, detection_head, fpn):
    """
    images: List of T RGB images I_i
    poses: List of T extrinsic matrices R_i
    intrinsics: Intrinsic matrix K
    voxel_grid: 3D grid of voxel centers p
    g_mlp: Shared Geometry MLP
    detection_head: 3D detection head
    fpn: Feature Pyramid Network backbone
    """
    T = len(images)
    C, H_f, W_f = 256, 64, 64 # Example dimensions for F_i
    
    # Step 1: Extract 2D features
    F = []
    for i in range(T):
        F_i = fpn(images[i]) # F_i in R^{C x H/4 x W/4}
        F.append(F_i)
        
    # Step 2 & 3: Construct 3D Volume and Aggregate
    # V_avg shape: (N_x, N_y, N_z, C)
    V_avg = torch.zeros((N_x, N_y, N_z, C))
    counts = torch.zeros((N_x, N_y, N_z))
    
    for i in range(T):
        R_i = poses[i]
        # Scale K for feature map downsampling (factor of 4)
        K_prime = intrinsics * 0.25 
        
        # Project voxel centers to 2D
        # p is (N_x * N_y * N_z, 3)
        p_homogeneous = torch.cat([voxel_grid, torch.ones_like(voxel_grid[:, :1])], dim=1)
        projected = K_prime @ R_i @ p_homogeneous.T # (3, N_voxels)
        
        u_prime = projected[0, :]
        v_prime = projected[1, :]
        d = projected[2, :]
        
        # Valid pixels: inside image bounds and in front of camera
        valid = (d > 0) & (u_prime >= 0) & (u_prime < W_f) & (v_prime >= 0) & (v_prime < H_f)
        
        u = u_prime[valid] / d[valid]
        v = v_prime[valid] / d[valid]
        
        # Nearest neighbor interpolation
        # Note: In implementation, this requires mapping 2D coords to grid indices
        # and gathering from F_i. Here we assume a gather function.
        V_i_valid = gather_nearest(F[i], u, v)
        
        # Update V_avg
        # This is a simplified representation of the aggregation logic
        # In practice, scatter_add is used
        counts[valid_indices] += 1
        V_avg[valid_indices] += V_i_valid
        
    # Average
    counts = torch.clamp(counts, min=1)
    V_avg = V_avg / counts.unsqueeze(-1)
    
    # Step 4: Geometry-Aware Modulation
    # Predict density for all voxels
    sigma = g_mlp(voxel_grid) # (N_voxels,)
    
    # Convert density to opacity
    # Assuming uniform step size delta or standard NeRF opacity formula
    # alpha = 1 - exp(-sigma * delta)
    # Since delta is not explicitly given, we assume standard volume rendering opacity
    # or that the G-MLP output is directly used as a weight if delta is normalized.
    # The paper says "transform the density field into an opacity field".
    # We use the standard definition [12995-feature-fields.md]
    delta = 0.1 # Placeholder for voxel size, or derived from grid bounds
    alpha = 1.0 - torch.exp(-sigma * delta)
    
    # Modulate features
    V_geo = alpha.unsqueeze(-1) * V_avg
    
    # Step 5: Detection
    boxes = detection_head(V_geo)
    
    return boxes

def nerf_det_inference(images, poses, intrinsics, voxel_grid, detection_head, fpn):
    """
    Inference removes the NeRF branch.
    """
    # Steps 1-3 are identical
    # Step 4 is skipped (no G-MLP, no alpha modulation)
    # V_geo is effectively V_avg or a simpler aggregation
    # However, the paper implies the volume features are pre-computed or 
    # the detection head is trained to work with the geometry-aware features.
    # During inference, the NeRF branch is removed.
    # This implies the detection head might rely on the volume structure 
    # learned during joint training, but without the dynamic alpha modulation 
    # at inference time? 
    # The paper states: "during inference, the NeRF branch is removed, which minimizes the additional overhead".
    # It does not explicitly say V_geo is replaced by V_avg at inference.
    # It likely means the G-MLP is not called, but the volume features V_avg 
    # are still used, possibly with a fixed opacity or just V_avg.
    # Given the ablation "disabling the detection branch brings 0.43dB improvement",
    # the NeRF branch is the G-MLP.
    # We will assume V_geo = V_avg at inference if G-MLP is removed, 
    # or that the detection head was trained to expect V_geo but 
    # the "removal" refers to not computing sigma/alpha dynamically.
    # However, standard practice for "removing a branch" implies using the 
    # remaining features. Let's assume V_geo = V_avg for inference simplicity 
    # unless specified otherwise. 
    # Actually, the paper says "multiply it with the volume features... 
    # Then, the geometry-aware volume features are fed to the detection head".
    # If the NeRF branch is removed, we don't have alpha. 
    # So we likely use V_avg.
    
    # Re-run Steps 1-3
    # ... (same as forward)
    
    # Use V_avg directly
    boxes = detection_head(V_avg)
    return boxes
```

**Parameters:**
*   **Density Activation:** ReLU [S1].
*   **Color Activation:** Sigmoid [S1].
*   **Feature Interpolation:** Nearest neighbor [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Projection:** Voxels projected outside the image boundary or behind the image plane must have their features set to 0 [S1].
*   **Depth Estimation Errors:** Using predicted depth maps from external geometry reconstruction (e.g., NeuralRecon) significantly degrades detection performance (by 2.3 mAP@.50 compared to plain ImVoxelNet) because depth estimation errors propagate inaccuracy through the detection pipeline [S1].
*   **Detection Branch Impact:** The detection branch negatively impacts the NeRF branch's novel view synthesis quality because it is prone to erasing low-level details required for NeRF rendering [S1]. Disabling the detection branch brings a 0.43dB improvement in PSNR [S1].

## Subtopics

### Shared G-MLP
A shared geometry MLP that enables the constraint of multi-view consistency to be propagated from the NeRF branch to the detection branch [S1]. Without the shared G-MLP, performance drops from 50.1 to 48.1 mAP@.25, as the influence of multi-view consistency is only propagated into the image backbone [S1].

## Operating characteristics
*   **Time Complexity:** The joint training is end-to-end. Inference latency is significantly lower than NeRF-then-Det methods. NeRF-Det-R50 has a latency of 0.554s on one V100, compared to NeRF-RPN's ~846.8s [S1].
*   **Memory Complexity:** Requires storing 3D feature volumes and the G-MLP parameters. The 3D grid size ($N_x \times N_y \times N_z$) determines memory usage for the volume features [S1].
*   **Performance:** Improves state-of-the-art by 3.9 mAP on ScanNet and 3.1 mAP on ARKITScenes benchmarks [S1]. On ScanNet, NeRF-Det-R50 achieves 35.13 AP50 [S1].
*   **Novel View Synthesis:** Achieves an average PSNR of 20+ for novel view synthesis and an RMSE of 0.756 for depth estimation on 312 scenes in the validation set, without per-scene training [S1].
*   **Strengths:** Generalizable to unseen scenes without per-scene optimization; explicitly models geometry; faster than NeRF-then-Det [S1].
*   **Limitations:** The detection branch can degrade the NeRF branch's rendering quality by erasing low-level details [S1].

## Background
NeRF-Det was introduced by Chenfeng et al. (Meta) in 2023 [S1]. It addresses the limitations of existing indoor 3D detection methods that struggle to model scene geometry and the high latency of per-scene NeRF optimization [S1]. It builds on the principles of [Feature Fields](12995-feature-fields.md) by using a shared geometry MLP to connect the NeRF and detection branches [S1].

## Sources
[S1] NeRF-Det: Learning Geometry-Aware Volumetric Representation for Multi-View 3D Object Detection, https://doi.org/10.1109/iccv51070.2023.02131, fetched 2026-10-30
