# Coherent 3D Gaussian Optimization


## Overview
Coherent 3D Gaussian Optimization is a regularization strategy for 3D Gaussian Splatting that enforces spatial coherence in Gaussian positions and opacities during training, specifically addressing artifacts in sparse-view novel view synthesis [S1]. It sets itself apart from standard 3DGS by replacing independent per-pixel residual updates with a learned implicit decoder that produces smooth, coherent residual fields for depth and opacity, supplemented by multiview total variation and optical flow constraints [S1]. This prevents the "semi-transparent" artifacts and detail loss common when optimizing Gaussians against sparse input views [S1].

## How it works
The method optimizes the Gaussian parameters ($\Sigma$, $\mathbf{c}$) indirectly by optimizing the parameters of an implicit convolutional decoder ($\phi$) that generates residual corrections for initial depth and opacity estimates [S1].

**Inputs and Outputs:**
*   **Inputs:**
    *   Set of input views $V$ [S1].
    *   Initial depth maps $D^{\text{init}}_n$ for each view $n$, obtained via monocular depth estimation (Yang et al. [60]) [S1].
    *   Initial Gaussian parameters $\Sigma, \mathbf{c}$ [G].
    *   Camera poses for each view [G].
*   **Outputs:**
    *   Optimized Gaussian parameters $\Sigma^*, \mathbf{c}^*$ [S1].
    *   Optimized decoder parameters $\phi^*$ [S1].

**Data Structures:**
*   **Implicit Decoder $f_\phi$:** A convolutional neural network taking the normalized view index $n$ as input [S1].
*   **Segmentation Mask $S$:** A binary mask of shape $(C, H, W)$, where $C=5$ is the number of regions [S1].
*   **Optical Flow Correspondences:** Pairs of pixels $(\mathbf{p}, \mathbf{q})$ between views $i$ and $j$ computed via optical flow (Shi et al. [45]) [S1].
*   **Forward-Backward Consistency Mask $M_{i \rightarrow j}$:** A binary mask indicating reliable pixel correspondences between views $i$ and $j$ [S1].

**Numbered Steps:**

1.  **Initialization:**
    *   Compute initial depth maps $D^{\text{init}}_n$ for all input views using a monocular depth estimator [S1].
    *   Initialize Gaussian parameters $\Sigma, \mathbf{c}$ using standard 3DGS initialization [G].
    *   Initialize decoder parameters $\phi$ randomly [G].
    *   Initialize segmentation masks $S_n$ for each view by dividing the image into $C=5$ regions based on $D^{\text{init}}_n$ [S1].

2.  **Residual Depth Estimation (Single-View Constraint):**
    *   For each view $n$, pass the normalized view index $n$ to the decoder $f_\phi(n)$ [S1].
    *   The decoder outputs a $C$-channel residual depth tensor [S1].
    *   Compute the final residual depth $\Delta D_n$ by applying channelwise summation (`chsum`) of the element-wise product of the segmentation mask $S_n$ and the decoder output:
        $$ \Delta D_n = \text{chsum}(S_n \odot f_\phi(n)) $$
    *   This ensures smooth deformation within regions while allowing discontinuities between regions [S1].

3.  **Residual Opacity Estimation:**
    *   Similarly, estimate residual opacity using a decoder (or shared architecture) producing $C$-channel residual opacity [S1].
    *   Combine with initial opacity to obtain final opacity at each pixel [S1].

4.  **Gaussian Position Update:**
    *   For each pixel $\mathbf{p}$ in view $n$, compute the updated 3D position $\mathbf{x}$ by projecting the residual depth into 3D space:
        $$ \mathbf{x} = g(D^{\text{init}}_n [\mathbf{p}] + \Delta D_n [\mathbf{p}], \mathbf{p}) $$
    *   Here, $g(d, \mathbf{p})$ is the function that projects pixel $\mathbf{p}$ into 3D space according to depth $d$ [S1].
    *   The Gaussian centers are updated to these positions $\mathbf{x}$ [S1].

5.  **Multiview Total Variation Loss ($\mathcal{L}_{\text{multi}}$):**
    *   Render the depth map $R_{\Sigma, \alpha, \mathbf{x}, d}$ for each view using the current Gaussian parameters and updated positions [S1].
    *   Compute the rendered disparity as $1 / (1 + R_{\Sigma, \alpha, \mathbf{x}, d})$ [S1].
    *   Compute the global TV loss:
        $$ \mathcal{L}_{\text{TV}} = \left\| \nabla \left( \frac{1}{1 + R_{\Sigma, \alpha, \mathbf{x}, d}} \right) \right\|_1 $$
    *   Compute the masked TV loss using the segmentation mask $S$:
        $$ \mathcal{L}_{\text{MTV}} = \left\| \nabla \left( S \odot \left( \frac{1}{1 + R_{\Sigma, \alpha, \mathbf{x}, d}} \right) \right) \right\|_1 $$
    *   Combine them with a schedule parameter $\lambda_s$:
        $$ \mathcal{L}_{\text{multi}} = (1 - \lambda_s)\mathcal{L}_{\text{TV}} + \lambda_s\mathcal{L}_{\text{MTV}} $$
    *   $\lambda_s$ is initialized to 0 and gradually increased to 1 by the end of optimization [S1].

6.  **Flow Loss ($\mathcal{L}_{\text{flow}}$):**
    *   For each pair of views $(i, j)$ with corresponding pixels $(\mathbf{p}, \mathbf{q})$ identified by optical flow:
    *   Compute the 3D positions for both pixels using their respective residual depths:
        $$ D_i [\mathbf{p}] = D^{\text{init}}_i [\mathbf{p}] + \Delta D_i [\mathbf{p}] $$
        $$ D_j [\mathbf{q}] = D^{\text{init}}_j [\mathbf{q}] + \Delta D_j [\mathbf{q}] $$
        $$ \mathbf{x}_i = g(D_i [\mathbf{p}], \mathbf{p}), \quad \mathbf{x}_j = g(D_j [\mathbf{q}], \mathbf{q}) $$
    *   Compute the flow loss using the consistency mask $M_{i \rightarrow j}$:
        $$ \mathcal{L}_{\text{flow}} = \sum_{(i, j)} \sum_{\mathbf{p}} \left\| M_{i \rightarrow j} \odot \left( \mathbf{x}_i - \mathbf{x}_j \right) \right\|_1 $$
    *   $p$ and $q$ are corresponding pixels calculated using an existing optical flow method [S1].

7.  **Multisampling:**
    *   During rendering for color loss calculation, evaluate multiple samples within each pixel and average them to obtain the pixel color [S1].
    *   This ensures Gaussians properly cover the pixel area, preventing semi-transparency in novel views [S1].

8.  **Optimization Step:**
    *   Minimize the total objective function with respect to $\Sigma, \phi, \mathbf{c}$:
        $$ \Sigma^*, \phi^*, \mathbf{c}^* = \arg \min_{\Sigma, \phi, \mathbf{c}} \sum_{\mathbf{p} \in \mathcal{P}} \mathcal{L}(R_{\Sigma, \alpha, \mathbf{x}, \mathbf{c}}(\mathbf{p}), R(\mathbf{p})) + \beta_m \mathcal{L}_{\text{multi}} + \beta_f \mathcal{L}_{\text{flow}} $$
    *   Typical weights: $\beta_m = 5$, $\beta_f = 0.1$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def coherent_3dgs_optimization(input_views, initial_depths, initial_gaussians, decoder, num_regions=5, num_epochs=1000):
    """
    Optimizes 3D Gaussians using coherent regularization.
    """
    phi = decoder.parameters()
    Sigma = initial_gaussians.Sigma
    c = initial_gaussians.c
    alpha = initial_gaussians.alpha

    H, W = initial_depths[input_views[0]].shape
    camera_poses = {v: get_camera_pose(v) for v in input_views}  # [G]
    reference_colors = {v: get_reference_image(v) for v in input_views}  # [G]
    optimizer = torch.optim.Adam(list(phi) + [Sigma, c], lr=1e-3)  # [G]

    segmentation_masks = {}
    for view_idx in input_views:
        segmentation_masks[view_idx] = compute_segmentation_mask(initial_depths[view_idx], num_regions)  # [G]

    flow_correspondences = {}
    consistency_masks = {}
    for i, view_i in enumerate(input_views):
        for j, view_j in enumerate(input_views):
            if i >= j: continue
            flow_correspondences[(view_i, view_j)] = compute_optical_flow(view_i, view_j)  # [G]
            consistency_masks[(view_i, view_j)] = compute_consistency_mask(flow_correspondences[(view_i, view_j)])  # [G]

    lambda_s = 0.0
    for epoch in range(num_epochs):
        lambda_s = min(1.0, epoch / num_epochs)
        residual_depths_map = {}

        for view_idx in input_views:
            normalized_view_idx = view_idx / len(input_views)
            residual_depth_tensor = decoder(normalized_view_idx)
            S = segmentation_masks[view_idx]
            residual_depth = np.sum(S * residual_depth_tensor, axis=0)
            residual_depths_map[view_idx] = residual_depth

            D_init = initial_depths[view_idx]
            updated_depth = D_init + residual_depth
            positions = [project_pixel_to_3d(p, updated_depth[p], camera_poses[view_idx]) for p in range(H * W)]

            R_d = render_depth(Sigma, alpha, positions, camera_poses[view_idx])
            R_c = render_color(Sigma, alpha, positions, c, camera_poses[view_idx], multisample=True)

            disparity = 1.0 / (1.0 + R_d)
            grad_disparity = np.gradient(disparity)
            L_TV = np.sum(np.abs(grad_disparity))

            masked_disparity = S * disparity
            grad_masked_disparity = np.gradient(masked_disparity)
            L_MTV = np.sum(np.abs(grad_masked_disparity))

            L_multi = (1 - lambda_s) * L_TV + lambda_s * L_MTV

            L_flow = 0.0
            for (view_i, view_j), flow in flow_correspondences.items():
                if view_i == view_idx or view_j == view_idx:
                    for p, q in enumerate(flow):
                        D_i = initial_depths[view_i][p] + residual_depths_map[view_i][p]
                        D_j = initial_depths[view_j][q] + residual_depths_map[view_j][q]
                        x_i = project_pixel_to_3d(p, D_i, camera_poses[view_i])
                        x_j = project_pixel_to_3d(q, D_j, camera_poses[view_j])
                        M = consistency_masks[(view_i, view_j)][p]
                        L_flow += M * np.linalg.norm(x_i - x_j, ord=1)

            L_color = np.mean(np.abs(R_c - reference_colors[view_idx]))
            total_loss = L_color + 5.0 * L_multi + 0.1 * L_flow

            total_loss.backward()
            optimizer.step()
            optimizer.zero_grad()

    return Sigma, c, phi

# Helper functions
def compute_segmentation_mask(D_init, num_regions):
    # Divide image into C regions based on D_init [G: Wang et al. strategy]
    pass  # Not given in the sources.

def compute_optical_flow(view_i, view_j):
    # Compute optical flow between view_i and view_j [G: Shi et al.]
    pass  # Not given in the sources.

def compute_consistency_mask(flow):
    # Compute forward-backward consistency mask [G]
    pass  # Not given in the sources.

def project_pixel_to_3d(p, depth, camera_pose):
    # Standard pinhole projection inverse [G]
    pass  # Not given in the sources.

def render_depth(Sigma, alpha, positions, camera_pose):
    # Standard 3DGS depth rendering [G]
    pass  # Not given in the sources.

def render_color(Sigma, alpha, positions, c, camera_pose, multisample):
    # Standard 3DGS color rendering with multisampling [G]
    pass  # Not given in the sources.
```

**Parameters:**
*   $C=5$: Number of segmentation regions [S1].
*   $\beta_m = 5$: Weight for multiview TV loss [S1].
*   $\beta_f = 0.1$: Weight for flow loss [S1].
*   $\lambda_s$: Schedule parameter for TV loss, starting at 0 and ending at 1 [S1].

**Edge Cases and Pitfalls:**
*   **Sparse Views:** Standard center-sampling leads to semi-transparency [S1].
*   **Sharp Discontinuities:** Smooth decoder alone fails [S1].
*   **Optical Flow Errors:** Unreliable correspondences must be masked out using forward-backward consistency [S1].

## Operating characteristics
*   **Time Complexity:** Higher than standard 3DGS due to the additional decoder forward pass, segmentation mask computation, and flow loss calculation [S1]. The decoder parameters $\phi$ are optimized alongside Gaussians [S1].
*   **Memory Complexity:** Requires storing segmentation masks and optical flow correspondences [S1].
*   **Performance:** Improves novel view synthesis quality in sparse-view settings by reducing artifacts and improving geometry coherence [S1].
*   **Strengths:** Handles sparse inputs better than standard 3DGS; enforces smooth geometry; prevents semi-transparency [S1].
*   **Limitations:** Requires monocular depth estimation for initialization; relies on optical flow which can be noisy; computationally more expensive [S1].
*   **When to use:** Sparse-view novel view synthesis where standard 3DGS produces blurry or transparent artifacts [S1].
*   **When not to use:** Dense-view settings where standard 3DGS performs well; real-time applications where the additional computational cost is prohibitive [G].

## Background
Coherent 3D Gaussian Optimization was introduced in "CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians" by A. Paliwal et al. [S1]. It builds on the 3D Gaussian Splatting framework [G] and addresses its limitations in sparse-view scenarios [S1].

## Sources
[S1] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27
