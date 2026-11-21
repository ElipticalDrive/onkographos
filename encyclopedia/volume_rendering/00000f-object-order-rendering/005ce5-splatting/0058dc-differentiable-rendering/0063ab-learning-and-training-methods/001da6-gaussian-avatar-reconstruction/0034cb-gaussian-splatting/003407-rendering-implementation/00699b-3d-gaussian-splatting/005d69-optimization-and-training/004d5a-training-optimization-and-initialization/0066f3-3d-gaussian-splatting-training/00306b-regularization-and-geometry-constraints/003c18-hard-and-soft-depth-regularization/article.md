# Hard and Soft Depth Regularization


## Overview
Hard and Soft Depth Regularization is a training strategy for 3D Gaussian Splatting that prevents detail loss and blurred appearance by constraining geometry updates to spatial reshaping (movement of centers) rather than deforming Gaussian shapes [S1]. It solves the conflict where forcing anisotropic Gaussians to fit smooth monocular depth estimates causes the primitive geometry to conform to the depth rather than the complex color appearance [S1]. It sets itself apart from standard depth regularization by freezing scale and rotation parameters and rendering two distinct depth maps to independently update center positions and opacities [S1].

## How it works
**Inputs:**
*   Current set of 3D Gaussians $\mathcal{G}$, each defined by position $\mu$, scale $s$, rotation quaternion $q$, and opacity $\alpha$.
*   A monocular depth estimate $D_{mono}$ for the current view.
*   Camera parameters for the current view.

**Outputs:**
*   Updated Gaussian parameters (specifically $\mu$ and $\alpha$) after one regularization step.

**Data Structures:**
*   **Gaussian State:** A list or array of Gaussian objects. Each object stores:
    *   $\mu \in \mathbb{R}^3$: Center position.
    *   $s \in \mathbb{R}^3$: Scale (log-space or linear, depending on implementation).
    *   $q \in \mathbb{R}^4$: Rotation quaternion.
    *   $\alpha \in [0, 1]$: Opacity.
*   **Depth Maps:** Two rendered depth maps, $D_{hard}$ and $D_{soft}$, generated from the current Gaussian state.

**Method Steps:**

1.  **Freeze Shape Parameters:**
    During the depth regularization step, the scale parameters $s$ and rotation quaternions $q$ of all Gaussians are frozen. They are not updated by the gradients computed in this step [S1].

2.  **Render Hard Depth Map ($D_{hard}$):**
    Render a depth map using the standard alpha-blending accumulation, but weighted by opacity to represent the "hard" surface closest to the camera. This map is used to compute gradients for position updates.
    *   *Note:* The exact definition of "hard" vs "soft" in the source is implied by the gradient targets: $D_{hard}$ targets position $\mu$, $D_{soft}$ targets opacity $\alpha$ [S1]. In standard differentiable rendering, $D_{hard}$ is often the expected depth or the depth of the first hit, while $D_{soft}$ might be the alpha-weighted average depth. The source states: "Use the first depth map to compute gradients that update only the center positions $\mu$. Use the second depth map to compute gradients that update only the opacities $\alpha$" [S1].

3.  **Render Soft Depth Map ($D_{soft}$):**
    Render a second depth map. This map is used to compute gradients for opacity updates. The source implies this map helps balance coarse depth fitting with fine color appearance [S1].

4.  **Compute Position Loss ($\mathcal{L}_{pos}$):**
    Compare $D_{hard}$ with the monocular depth estimate $D_{mono}$. Compute a loss (e.g., L1 or L2) between the rendered hard depth and the monocular depth.
    $$ \mathcal{L}_{pos} = \text{Loss}(D_{hard}, D_{mono}) $$

5.  **Compute Opacity Loss ($\mathcal{L}_{alpha}$):**
    Compare $D_{soft}$ with the monocular depth estimate $D_{mono}$. Compute a loss between the rendered soft depth and the monocular depth.
    $$ \mathcal{L}_{alpha} = \text{Loss}(D_{soft}, D_{mono}) $$

6.  **Compute Gradients:**
    *   Compute gradients of $\mathcal{L}_{pos}$ with respect to the Gaussian positions $\mu$.
    *   Compute gradients of $\mathcal{L}_{alpha}$ with respect to the Gaussian opacities $\alpha$.
    *   Gradients for $s$ and $q$ are explicitly ignored or zeroed out [S1].

7.  **Update Parameters:**
    *   Update $\mu$ using the gradients from $\mathcal{L}_{pos}$.
    *   Update $\alpha$ using the gradients from $\mathcal{L}_{alpha}$.
    *   Do not update $s$ or $q$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hard_and_soft_depth_regularization(gaussians, camera, monocular_depth, lr_pos, lr_alpha):
    """
    Performs one step of Hard and Soft Depth Regularization.
    
    Args:
        gaussians: List of Gaussian objects with attributes mu, s, q, alpha.
        camera: Camera parameters (intrinsics, extrinsics).
        monocular_depth: 2D array representing the monocular depth estimate.
        lr_pos: Learning rate for position updates.
        lr_alpha: Learning rate for opacity updates.
        
    Returns:
        Updated gaussians list.
    """
    # Step 1: Freeze shape parameters (s, q) by not including them in the computation graph
    # or by zeroing their gradients after backward pass.
    
    # Step 2: Render Hard Depth Map
    # D_hard is typically the expected depth or depth of the first hit.
    # Implementation detail: Render depth weighted by opacity, but prioritizing closer Gaussians.
    D_hard = render_hard_depth(gaussians, camera)
    
    # Step 3: Render Soft Depth Map
    # D_soft is typically the alpha-weighted average depth.
    D_soft = render_soft_depth(gaussians, camera)
    
    # Step 4: Compute Position Loss
    # Example: L1 loss between rendered hard depth and monocular depth
    loss_pos = torch.mean(torch.abs(D_hard - monocular_depth))
    
    # Step 5: Compute Opacity Loss
    # Example: L1 loss between rendered soft depth and monocular depth
    loss_alpha = torch.mean(torch.abs(D_soft - monocular_depth))
    
    # Step 6: Compute Gradients
    # Gradients for mu and alpha
    grad_mu = torch.autograd.grad(loss_pos, [g.mu for g in gaussians], retain_graph=True)
    grad_alpha = torch.autograd.grad(loss_alpha, [g.alpha for g in gaussians])
    
    # Step 7: Update Parameters
    for i, g in enumerate(gaussians):
        # Update position
        g.mu = g.mu - lr_pos * grad_mu[i]
        
        # Update opacity
        g.alpha = g.alpha - lr_alpha * grad_alpha[i]
        
        # Do NOT update s and q
        # g.s = g.s
        # g.q = g.q
        
    return gaussians

def render_hard_depth(gaussians, camera):
    """
    Renders a depth map where depth is determined by the closest Gaussian hit.
    This approximates the 'hard' surface.
    """
    # Standard differentiable rasterization, but depth is stored as the z-coordinate
    # of the hit point, weighted by opacity.
    # To make it 'hard', we might use the depth of the first Gaussian in the sorted list
    # or a specific weighting. The source does not specify the exact rendering equation
    # for D_hard vs D_soft, only that they are two distinct maps.
    # A common implementation for 'hard' depth in 3DGS is the expected depth
    # or the depth of the first hit.
    # For this pseudocode, we assume a standard differentiable renderer that outputs
    # depth. We distinguish 'hard' by using the depth of the first hit (closest).
    pass

def render_soft_depth(gaussians, camera):
    """
    Renders a depth map using alpha-weighted average depth.
    """
    pass
```

**Parameters:**
*   **Learning Rates:** Separate learning rates for position ($\mu$) and opacity ($\alpha$) updates. Typical values are not given in the sources.
*   **Loss Function:** The source does not specify the exact loss function (e.g., L1, L2, Charbonnier) used for $\mathcal{L}_{pos}$ and $\mathcal{L}_{alpha}$.

**Edge Cases and Pitfalls:**
*   **Detail Loss:** Applying depth regularization directly to NeRF-style MLPs or un-frozen Gaussian shapes forces the primitive geometry to conform to smooth monocular depth estimates, which conflicts with complex color appearance and causes detail loss [S1].
*   **Noise:** Gaussian radiance fields are more sensitive to small depth errors than NeRFs, which may result in a noisy distribution of primitives and failures in regions with complex textures [S1].
*   **Scale Invariance:** Existing scale-invariant depth losses often align depth maps to a fixed scale, which can overlook minor depth variations [S1]. This method addresses this by separating position and opacity updates, but the source notes that scale-invariant losses are a general issue in the field [S1].

## Subtopics

### [Global-Local Depth Normalization](9179-global-local-depth-normalization.md)
Global-Local Depth Normalization is a depth regularization technique for 3D Gaussian Splatting that prevents scale-invariant losses from overlooking minor depth variations, thereby improving detailed geometry reshaping in texture-rich regions. It sets itself apart from standard scale-invariant losses by combining global scale preservation with local scale amplification, addressing the issue where existing methods align depth maps to a fixed global scale and ignore small local errors.

### [Depth Reinitialization](9233-depth-reinitialization.md)
Depth Reinitialization is a regularization strategy that alleviates 'overlapping' and 'under-reconstruction' artifacts in 3D Gaussian Splatting by leveraging dense depth information to reorganize the spatial distribution of Gaussians. It sets itself apart from sibling methods by rejecting standard alpha-blended depth maps in favor of geometric mid-point depths to generate clean, artifact-free point clouds for reinitialization.

### [Position Regularization](11049-position-regularization.md)
Position Regularization prevents floating artifacts and dramatic geometry shifts in 3D Gaussian Splatting by constraining Gaussian centers to remain close to their initial mesh positions or a reference surface. It differs from sibling methods like Global-Local Depth Normalization, which focuses on depth consistency, and Depth Reinitialization, which resets positions based on depth errors, by directly penalizing displacement from a known geometric prior (SMPL mesh or initial mesh bounds) rather than enforcing consistency with rendered depth maps [9178-hard-and-soft-depth-regularization.md].

### [Coherent 3D Gaussian Optimization](14045-coherent-3d-gaussian-optimization.md)
Coherent 3D Gaussian Optimization is a regularization strategy for 3D Gaussian Splatting that enforces spatial coherence in Gaussian positions and opacities during training, specifically addressing artifacts in sparse-view novel view synthesis. It sets itself apart from standard 3DGS by replacing independent per-pixel residual updates with a learned implicit decoder that produces smooth, coherent residual fields for depth and opacity, supplemented by multiview total variation and optical flow constraints. This prevents the "semi-transparent" artifacts and detail loss common when optimizing Gaussians against sparse input views.

### Geometric Regularization via Bilateral Filter
Geometric Regularization via Bilateral Filter is a regularization loss that enforces coherent motion by grouping Gaussians in image space using a bilateral filter approach, avoiding the dampening of independent motion caused by spatial KNN [S2]. It is used in scenes with oscillatory motion to prevent the grouping of spatially close but independently moving objects (e.g., leaves of a tree) [S2].

**Steps:**
1.  For a training image $I$ at time $t_I$, sample another time $t'$ [S2].
2.  Compute the 6D screw-axis representation of the transformation between $t_I$ and $t'$ for each Gaussian [S2].
3.  Blend these vectors to form an image the same way color is blended [S2].
4.  For each pixel $p$, compute a weighted sum of the squared distances of the splatted screw-axis representations between $p$ and neighboring pixels $p'$ [S2].

**Formula:**
The geometric loss is:
$$ \mathcal{L}_{\text{geo}} = \sum_{p' \in \mathcal{N}(p)} k_{c,c'} k_{d,d'} k_{p,p'} \|s_p - s_{p'}\|^2 $$
where $k_{c,c'}$ is color similarity, $k_{d,d'}$ is depth similarity, $k_{p,p'}$ is spatial distance weight, and $s_p, s_{p'}$ are screw-axis representations [S2].

**Pitfall:**
Unlike standard bilateral filters, the loss is not divided by the sum of the weights because the goal is comparison, not energy conservation [S2].

### Isotropic Regularization
Isotropic Regularization maintains coherent 3D structure and optimizes newly inserted Gaussians using current and random past keyframes to prevent forgetting [S3, unofficial]. It is used in continuous SLAM to prevent Gaussians from becoming highly elongated along the viewing direction, which creates artifacts [S3, unofficial].

**Steps:**
1.  Use keyframes in current window $W_k$ and two random past keyframes $W_r$ per iteration [S3, unofficial].
2.  Compute the isotropic regularization loss based on scaling parameters [S3, unofficial].

**Formula:**
$$ E_{iso} = \sum_{i=1}^{|\mathcal{G}|} \|\mathbf{s}_i - \tilde{\mathbf{s}}_i \cdot \mathbf{1}\|_1 $$
where $\mathbf{s}_i$ are scaling parameters, $\tilde{\mathbf{s}}_i$ is their mean, and $\mathbf{1}$ is a vector of ones [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Prevents detail loss and blurred appearance caused by forcing anisotropic Gaussians to fit smooth monocular depth [S1]. Balances coarse depth fitting with fine color appearance [S1].
*   **Limitations:** Gaussian radiance fields are more sensitive to small depth errors than NeRFs, which may result in a noisy distribution of primitives [S1]. Existing scale-invariant depth losses may overlook minor depth variations [S1].
*   **When to Use:** When using monocular depth estimates for regularization in 3D Gaussian Splatting, especially in scenes with complex color appearance where shape deformation would cause detail loss [S1].
*   **When Not to Use:** When the scene geometry is simple and monocular depth is accurate, or when computational resources are limited and the overhead of rendering two depth maps is prohibitive.

## Background
Hard and Soft Depth Regularization was proposed in the context of optimizing sparse-view 3D Gaussian Radiance Fields [S1]. It addresses the limitations of existing depth regularization strategies for NeRFs, which commonly employ depth to regularize the entire model, creating a potential geometry conflict in the Gaussian fields [S1]. The method is based on the observation that the basis of scene geometry lies in the position of the Gaussian primitives rather than their shape [S1].

## Sources
[S1] DNGaussian: Optimizing Sparse-View 3D Gaussian Radiance Fields with Global-Local Depth Normalization, https://doi.org/10.1109/cvpr52733.2024.01963, fetched 2026-10-26
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
[S3] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
