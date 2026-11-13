# SplattingAvatar

Also called: Realistic Real-Time Human Avatars with Mesh-Embedded Gaussian Splatting

## Overview
SplattingAvatar is a real-time human avatar rendering method that embeds 3D Gaussians onto a driving mesh (FLAME) to enable non-ambiguous motion control and high-fidelity rendering of complex geometries like eyes and hair [S2, unofficial]. It avoids the noise associated with nearest-triangle sampling in canonical space by using trainable embeddings for motion control [S2, unofficial].

## How it works
**Inputs:** Multi-angle video of a human head; pre-fitted FLAME mesh parameters (pose, expression, shape) [S1, unofficial][S2, unofficial].
**Outputs:** A set of 3D Gaussians embedded in the mesh tangent space, with trainable embeddings for deformation, rendered as real-time video [S2, unofficial].

**Data Structures:**
*   **Gaussian:** Defined by position, covariance, opacity, spherical harmonics coefficients, and a trainable embedding vector [S2, unofficial].
*   **Embedding:** A learnable vector per Gaussian, used to compute pose- and expression-dependent offsets [S2, unofficial].
*   **Mesh:** FLAME head mesh, providing the tangent space basis for initialization [S2, unofficial].

**Steps:**
1.  **Initialization:** Initialize one Gaussian per mesh triangle in the mesh tangent space [S1, unofficial].
2.  **Training Loop:** Optimize Gaussian parameters (position, covariance, opacity, SH, embeddings) using 3D Gaussian Splatting [23] loss [S2, unofficial].
    *   **Loss:** $\mathcal{L} = \lambda_l \mathcal{L}_1 + \lambda_s \mathcal{L}_{SSIM}$ [S2, unofficial].
    *   **Parameters:** $\lambda_l = 0.01$, $\lambda_s = 1.0$ [S2, unofficial].
    *   **Thresholds:** $T_s = 10.0$, $T_r = 0.008$ [S2, unofficial].
3.  **Densification and Pruning:** Starting from iteration 600, densify and prune Gaussians every 100 iterations [S2, unofficial].
4.  **Opacity Reset:** Every 3000 iterations, reset all Gaussian opacities to zero to remove redundant Gaussians [S2, unofficial].
5.  **Walking on Triangle:** After densifying and pruning, enforce Gaussians to stay on the mesh triangles [S2, unofficial].
6.  **Termination:** Stop densify, prune, and opacity-reset at iteration 15,000 [S2, unofficial]. Total iterations: 30,000 [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
gaussians = []
for triangle in mesh.triangles:
    # Initialize one Gaussian per triangle in tangent space
    g = Gaussian()
    g.position = triangle.center
    g.covariance = identity_matrix
    g.opacity = 1.0
    g.embedding = random_vector
    gaussians.append(g)

# Training Loop
for iteration in range(1, 30001):
    # Render current frame
    rendered_image = render(gaussians, camera_pose, mesh_pose)
    
    # Compute Loss
    loss = 0.01 * L1_loss(rendered_image, target_image) + 1.0 * SSIM_loss(rendered_image, target_image)
    
    # Backpropagate and optimize
    optimize(gaussians, loss)
    
    # Densify and Prune (every 100 iterations, starting from 600)
    if iteration >= 600 and iteration % 100 == 0:
        densify(gaussians, threshold=10.0)
        prune(gaussians, threshold=0.008)
        # Walking on Triangle step
        walking_on_triangle(gaussians, mesh)
        
    # Opacity Reset (every 3000 iterations)
    if iteration % 3000 == 0:
        for g in gaussians:
            g.opacity = 0.0
            
    # Stop densify/prune/reset after 15000 iterations
    if iteration > 15000:
        # Only render and optimize, no densify/prune/reset
        pass

# Rendering
for frame in frames:
    # Update mesh pose based on driving data
    mesh_pose = get_pose(frame)
    # Render using Unity compute shaders
    render(gaussians, camera_pose, mesh_pose)
```

**Edge Cases and Pitfalls:**
*   **Nearest-Triangle Sampling:** Using nearest-triangle sampling for deformation from posed space to canonical space causes unstable sampling and strong noise in complex geometries like the eye region [S2, unofficial]. SplattingAvatar avoids this by using embeddings-based motion control [S2, unofficial].
*   **Disabling Walking on Triangle:** Disabling the 'walking on triangle' mechanism and clipping UV values causes Gaussians to stick and pile up on the boundaries of mesh triangles, leading to performance drops in PSNR, SSIM, and LPIPS metrics [S2, unofficial].

## Operating characteristics
*   **Performance:** Over 300 FPS on an NVIDIA RTX 3090 GPU [S2, unofficial].
*   **Quality:** On head avatar dataset, Ours+FLAME achieves PSNR 28.19, SSIM 0.931, LPIPS 0.063. Ours+NHA achieves PSNR 28.86, SSIM 0.931, LPIPS 0.060 [S2, unofficial].
*   **Comparison:** Compared to FLARE, SplattingAvatar does not focus on accurate geometry and materials reconstruction but offers significant improvement in photometric quality and rendering efficiency [S2, unofficial].
*   **Limitations:** Metrics can be affected by errors in the pre-processing pipeline, particularly in the neck area due to segmentation and head tracking limitations [S2, unofficial].

## Background
SplattingAvatar was introduced in the paper "SplattingAvatar: Realistic Real-Time Human Avatars with Mesh-Embedded Gaussian Splatting" [S2, unofficial]. It builds on 3D Gaussian Splatting [23] and addresses limitations of previous methods like INSTA [53] and PointAvatar [51] by embedding Gaussians in mesh tangent space [S2, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
