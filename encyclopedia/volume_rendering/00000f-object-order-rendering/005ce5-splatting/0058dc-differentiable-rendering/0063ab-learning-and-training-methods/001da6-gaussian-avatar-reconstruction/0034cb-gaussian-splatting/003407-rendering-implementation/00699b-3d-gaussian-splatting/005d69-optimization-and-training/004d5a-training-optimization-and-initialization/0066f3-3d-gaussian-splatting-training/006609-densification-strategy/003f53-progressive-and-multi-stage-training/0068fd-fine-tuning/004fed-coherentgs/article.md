# CoherentGS

Also called: Coherent 3D Gaussians, Sparse Novel View Synthesis with Coherent 3D Gaussians

## Overview
CoherentGS is a regularized optimization method for sparse novel view synthesis (NVS) that introduces structural coherency to explicit 3D Gaussian representations by constraining Gaussian movement in 2D image space [S1]. It solves the instability and "needle-like" artifacts common in standard 3D Gaussian Splatting (3DGS) when trained on extremely sparse input views (e.g., 3–4 images) by enforcing single-view and multi-view consistency, thereby preventing overfitting and allowing for the identification of occluded regions suitable for inpainting [S1]. Unlike NeRF-based sparse methods that rely on implicit MLPs and produce blurry occluded regions, CoherentGS maintains an explicit representation that can be cleanly segmented for diffusion-based inpainting [S1].

## How it works
**Inputs:**
*   $N$ sparse input images (typically $N=3$ or $4$) with known camera poses [S1].
*   Monocular depth estimates for each input view, predicted using a pre-trained model (e.g., Depth Anything) [S1].
*   Dense flow correspondences between all pairs of input images, predicted using a pre-trained model (e.g., FlowFormer++) [S1].
*   Binary segmentation masks derived from thresholding the monocular depth estimates to preserve sharp discontinuities [S1].

**Outputs:**
*   A set of 3D Gaussians defined by position $\mathbf{x}$, covariance $\Sigma$, opacity $\alpha$, and color $\mathbf{c}$ [S1].

**Data Structures:**
The method assigns exactly one Gaussian to every pixel of every input image, creating a structured representation [S1].
*   Let $\mathcal{G}$ be the set of Gaussians.
*   For each input image $n \in \{1, \dots, N\}$ and each pixel $\mathbf{p}$ in that image, there is a corresponding Gaussian $g_{n,\mathbf{p}} \in \mathcal{G}$ [S1].
*   The position $\mathbf{x}_{n,\mathbf{p}}$ of each Gaussian is constrained to lie on the ray connecting the camera center of view $n$ to pixel $\mathbf{p}$ [S1].
*   Initial positions are determined by the monocular depth estimate $D_{init}(n, \mathbf{p})$ [S1].

**Numbered Steps:**

1.  **Depth-based Initialization:**
    For each input image $n$ and pixel $\mathbf{p}$, initialize the 3D position $\mathbf{x}_{n,\mathbf{p}}$ using the monocular depth estimate $D_{init}(n, \mathbf{p})$ [S1]. The depth is relative, so the initialization places Gaussians in world space consistent with the camera pose [S1].

2.  **Implicit Decoder Setup:**
    Initialize an implicit convolutional decoder (e.g., a small CNN) [S1]. This decoder takes the 2D image coordinate $\mathbf{p}$ and the camera pose of the input view $n$ as input [S1].

3.  **Regularized Optimization Loop:**
    Minimize the optimization objective to update Gaussian parameters $\Sigma, \alpha, \mathbf{x}, \mathbf{c}$ [S1]. The objective function is:
    $$ \Sigma^*, \alpha^*, \mathbf{x}^*, \mathbf{c}^* = \arg\min_{\Sigma, \alpha, \mathbf{x}, \mathbf{c}} \sum_{\mathbf{p} \in \mathcal{P}} \mathcal{L}(R_{\Sigma, \alpha, \mathbf{x}, \mathbf{c}}(\mathbf{p}), R(\mathbf{p})) $$
    where $\mathcal{P}$ contains all pixels of every input image, $R(\mathbf{p})$ is the reference pixel color, and $\mathcal{L}$ is the combination of L1 and SSIM losses [S1].

    During optimization, the rendering equation for a pixel $\mathbf{p}$ is:
    $$ R_{\Sigma, \alpha, \mathbf{x}, \mathbf{c}}(\mathbf{p}) = \sum_{i \in \mathcal{N}(\mathbf{p})} \mathbf{c}_i \gamma_i \prod_{j=1}^{i-1}(1-\gamma_j) $$
    where $\mathcal{N}(\mathbf{p})$ is the number of ordered points overlapping pixel $\mathbf{p}$, and $\gamma_k = f(\Sigma_k, \alpha_k, \mathbf{x}_k, \mathbf{p})$ computes the effective opacity by evaluating the Gaussian at $\mathbf{p}$ and multiplying by global opacity $\alpha$ [S1].

    To enforce coherency, the following constraints are applied to the gradients or directly to the parameters:

    *   **Single-View Coherency:**
        Pass the input view coordinate $\mathbf{p}$ and camera pose to the implicit decoder to predict a residual depth value $\Delta D$ [S1]. The updated depth $D$ is $D_{init} + \Delta D$ [S1]. Gaussians with similar initial depth are forced to move together by the decoder's output, ensuring smooth depth variations within a single view [S1]. Binary segmentation masks are applied to the decoder output to preserve sharp discontinuities [S1].

    *   **Multi-View Coherency:**
        Render the depth map for all Gaussians for each view [S1]. Apply a Total Variation (TV) loss to the rendered depth across all Gaussians to ensure smoothness and consistency across views [S1].

    *   **Flow-based Loss:**
        Use the pre-computed dense flow correspondences between image pairs [S1]. Apply a flow-based loss to constrain optimization by ensuring that corresponding pixels in two images have similar Gaussian positions [S1]. This enforces cross-view consistency [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coherent_gs_optimization(input_images, camera_poses, monocular_depths, flow_correspondences):
    """
    input_images: List of N images
    camera_poses: List of N camera poses
    monocular_depths: List of N depth maps (relative depth)
    flow_correspondences: Dict of flow maps between image pairs
    
    Returns: Optimized set of 3D Gaussians
    """
    N = len(input_images)
    
    # 1. Initialization: Assign one Gaussian per pixel per image
    # Structure: Gaussians[n][p] = {x, sigma, alpha, c}
    # x is initialized using monocular depth
    gaussians = {}
    for n in range(N):
        gaussians[n] = {}
        H, W = input_images[n].shape[:2]
        for y in range(H):
            for x in range(W):
                p = (x, y)
                # Initialize position on the ray using monocular depth
                # ray_origin = camera_poses[n].position
                # ray_direction = unproject(camera_poses[n], p)
                # depth_init = monocular_depths[n][y, x]
                # gaussians[n][p].x = ray_origin + ray_direction * depth_init
                gaussians[n][p].x = initialize_position_on_ray(camera_poses[n], p, monocular_depths[n][y, x])
                gaussians[n][p].sigma = initial_covariance()
                gaussians[n][p].alpha = initial_opacity()
                gaussians[n][p].c = input_images[n][y, x]

    # 2. Initialize Implicit Decoder
    decoder = ImplicitConvolutionalDecoder()
    
    # 3. Optimization Loop
    for iteration in range(num_iterations):
        total_loss = 0
        
        # Render and compute photometric loss
        for n in range(N):
            rendered_image = render_image(gaussians[n], camera_poses[n])
            # L1 + SSIM loss
            photometric_loss = compute_l1_ssim_loss(rendered_image, input_images[n])
            total_loss += photometric_loss
            
        # 4. Single-View Coherency Loss
        for n in range(N):
            # Get segmentation masks from monocular depth to preserve discontinuities
            masks = compute_segmentation_masks(monocular_depths[n])
            
            # Predict residual depth using implicit decoder
            # Input: pixel coordinates and camera pose
            residual_depths = {}
            for y in range(H):
                for x in range(W):
                    p = (x, y)
                    # decoder takes p and camera pose n
                    delta_d = decoder.predict(p, camera_poses[n])
                    residual_depths[p] = delta_d
            
            # Apply masks to residual depths
            masked_residuals = apply_masks(residual_depths, masks)
            
            # Enforce that Gaussians with similar depth move together
            # This is implemented by adding a loss term that penalizes 
            # deviations from the decoder's predicted smooth depth field
            single_view_loss = compute_single_view_coherency_loss(gaussians[n], masked_residuals)
            total_loss += single_view_loss
            
        # 5. Multi-View Coherency Loss (Total Variation on Rendered Depth)
        multi_view_loss = 0
        for n in range(N):
            rendered_depth = render_depth(gaussians[n], camera_poses[n])
            # Total Variation loss on the rendered depth map
            tv_loss = compute_total_variation(rendered_depth)
            multi_view_loss += tv_loss
        total_loss += multi_view_loss
        
        # 6. Flow-based Loss
        flow_loss = 0
        for (n1, n2), flow_map in flow_correspondences.items():
            # For each pixel p in image n1, find corresponding pixel p' in n2
            # Ensure Gaussian positions at p and p' are consistent with flow
            # Specifically, the 3D position of Gaussian at p in n1 and p' in n2
            # should project consistently or have similar depth/position
            flow_loss += compute_flow_consistency_loss(gaussians, n1, n2, flow_map)
        total_loss += flow_loss
        
        # Backpropagate and update Gaussians
        gradients = compute_gradients(total_loss, gaussians)
        update_gaussians(gaussians, gradients)
        
    return gaussians

def initialize_position_on_ray(camera, pixel, depth):
    # [G] Standard unprojection logic
    ray_origin = camera.position
    ray_direction = unproject_ray(camera, pixel)
    return ray_origin + ray_direction * depth

def unproject_ray(camera, pixel):
    # [G] Convert pixel coordinates to normalized device coordinates, 
    # then to world space ray direction using camera intrinsics
    pass

def initial_covariance():
    # [G] Small initial covariance to allow growth during densification if needed,
    # though CoherentGS primarily focuses on optimization of initialized Gaussians
    pass

def initial_opacity():
    # [G] Standard initial opacity for 3DGS
    pass
```

**Parameters:**
*   $N$: Number of sparse input images. Typically 3 or 4 [S1].
*   Learning rate: Not specified in the sources [S1].
*   Number of iterations: Not specified in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Optimization Instability:** Constraining unstructured Gaussians directly in 3D space causes optimization instability [S1]. CoherentGS avoids this by mapping constraints to 2D image space [S1].
*   **Occluded Regions:** Because the constraints prevent Gaussians from moving freely in 3D space, the method does not reconstruct areas that are occluded in all input images [S1]. This is a feature, not a bug, as it allows these regions to be identified and inpainted using diffusion models [S1].
*   **Relative Depth:** Monocular depth estimates are relative and lack consistency across views [S1]. The depth-based initialization positions Gaussians in world space, but the regularized optimization is required to encourage coherent updates [S1].

## Subtopics

### [MVGS (Multi-view-regulated Gaussian Splatting)](15778-mvgs-multi-view-regulated-gaussian-splatting.md)
MVGS is a plug-and-play optimization method that improves Novel View Synthesis (NVS) performance for existing Gaussian-based methods by constraining the optimization of 3D Gaussians with multi-view information. It sets itself apart from its parent [CoherentGS](14072-coherentgs.md), which enforces 2D image-space coherency, by explicitly leveraging cross-view consistency through a cross-intrinsic guidance scheme and multi-view cross-ray densification to enhance detail and stability.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** CoherentGS yields high-quality synthesized views compared to state-of-the-art sparse-view NeRF-based approaches (e.g., FreeNeRF, SparseNeRF) on a variety of scenes [S1].
*   **Scaling:** The method is designed for extremely sparse input views (e.g., 3 images) [S1].
*   **Strengths:** Prevents overfitting to input images; allows identification of fully occluded areas for inpainting; avoids blurry occluded regions common in NeRF-based methods; improves texture and geometry reconstruction [S1].
*   **Limitations:** Does not reconstruct areas occluded in all input images (requires inpainting); relies on pre-trained monocular depth and flow models [S1].
*   **When to use:** Sparse novel view synthesis (3-4 input images) in static scenes [S1].
*   **When not to use:** Dense input views (standard 3DGS is more appropriate); scenes where fully automatic reconstruction of all occluded regions without inpainting is required [S1].

## Background
CoherentGS was proposed by A. Paliwal et al. during an internship at Meta Reality Labs [S1]. The work addresses the limitation of 3D Gaussian Splatting (3DGS) in extremely sparse input settings, where the unstructured representation leads to artifacts [S1]. It builds on the explicit representation of 3DGS [23] and integrates concepts from implicit decoders [5, 30, 37] and monocular depth prediction [60] and flow estimation [45] to enforce coherency [S1].

## Sources
[S1] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27
