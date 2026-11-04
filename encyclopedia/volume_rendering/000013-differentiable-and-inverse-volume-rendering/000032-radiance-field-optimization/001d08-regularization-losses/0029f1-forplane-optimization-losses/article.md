# Forplane Optimization Losses

Also called: Forplane Losses

## Overview
Forplane Optimization Losses are a set of regularization terms designed to stabilize the training of neural radiance fields that utilize orthogonal neural planes (Forplanes) for spatiotemporal representation [S1]. They address the ambiguity between static and dynamic scene components by enforcing structural smoothness in spatial planes, temporal consistency in space-time planes, and separation of static/dynamic features via a disentangle loss [S1]. Unlike standard photometric losses, these terms explicitly constrain the geometry and dynamics of the learned field to prevent overfitting to noise and ensure physically plausible motion [S1].

## How it works
The method operates on a representation where the scene is decomposed into a static field ($V_{sd}^s$) and a dynamic field ($V_{sd}^d$), each composed of orthogonal 2D planes in 3D space or space-time volumes [S1]. The losses are computed on these plane representations during the optimization of the neural network parameters $\boldsymbol{\theta}$ (geometry) and $\boldsymbol{\phi}$ (color) [S1].

### Inputs and Outputs
**Inputs:**
1.  **Static Field Planes ($\mathbf{V}_{sd}^s$):** A set of 2D planes representing the static component. Defined as $\{G_{XY}, G_{YZ}, G_{XZ}\}$ [S1].
2.  **Dynamic Field Planes ($\mathbf{V}_{sd}^d$):** A set of 2D planes representing the dynamic component. Defined as $\{G_{XT}, G_{YT}, G_{ZT}\}$ [S1]. Note: The source text refers to these as "space-time planes" in the context of $\mathcal{L}_{TS}$ and $\mathcal{L}_{DE}$, but distinguishes them from the spatial planes in $\mathcal{L}_{TV}$ [S1].
3.  **Ground Truth Images ($I_i$):** A set of input images with corresponding ground truth colors [S1].
4.  **Hyperparameters:** Weights $\lambda_d, \lambda_{tv}, \lambda_{ts}, \lambda_{de}$ [S1].

**Outputs:**
*   **Scalar Loss Value:** The total loss $\mathcal{L}_{total}$ used for backpropagation [S1].

### Data Structures
The planes are stored as 2D tensors (grids).
*   $G_{XY}$: Shape $(H, W)$, representing the X-Y spatial plane.
*   $G_{XT}$: Shape $(H, T)$, representing the X-Time plane.
*   Resolution: The dimensions depend on the discretization of the spatial and temporal axes. The source notes that differences are normalized by resolution [S1].

### Formulas and Definitions

1.  **Photometric Loss ($\mathcal{L}_{rgb}$):**
    $$ \mathcal{L}_{\text{rgb}}(\boldsymbol{\theta}, \boldsymbol{\phi}) = \sum_i \mathbb{E}_{\mathbf{r} \sim I_i} \left[ \left\| \hat{\boldsymbol{C}}(\mathbf{r}) - \boldsymbol{C}_i^{\text{gt}}(\mathbf{r}) \right\|_2^2 \right] $$
    Where $\hat{\boldsymbol{C}}(\mathbf{r})$ is the rendered color for ray $\mathbf{r}$, and $\boldsymbol{C}_i^{\text{gt}}(\mathbf{r})$ is the ground truth color [S1].

2.  **Total Variation Loss ($\mathcal{L}_{TV}$):**
    Applied to the static spatial planes $\mathbf{V}_{sd}^s = \{G_{XY}, G_{YZ}, G_{XZ}\}$ [S1].
    $$ \mathcal{L}_{TV} = \sum_{G \in \mathbf{V}_{sd}^s} \sum_{g \in G} \|\Delta_h^2(G, g) + \Delta_w^2(G, g)\| $$
    Where $\Delta_h^2(G, g)$ is the squared difference between the $g$-th value in plane $G$ and its neighbor in the height direction, normalized by resolution. Similarly for $\Delta_w^2$ in the width direction [S1].

3.  **Temporal Smoothness Loss ($\mathcal{L}_{TS}$):**
    Applied to the dynamic space-time planes $\mathbf{V}_{3d}^d = \{G_{XT}, G_{YT}, G_{ZT}\}$ [S1].
    $$ \mathcal{L}_{TS} = \sum_{G \in \mathbf{V}_{3d}^d} \sum_{g \in G} \|\Delta_t^2(G, g)\| $$
    Where $\Delta_t^2(G, g)$ represents the squared difference along the time axis for element $g$ in plane $G$ [S1].

4.  **Disentangle Loss ($\mathcal{L}_{DE}$):**
    Applied to the dynamic space-time planes $\mathbf{V}_{sd}^v$ (identified as $\{G_{XT}, G_{YT}, G_{ZT}\}$ in the context of dynamic fields) [S1].
    $$ \mathcal{L}_{DE} = \sum_{G \in \mathbf{V}_{sd}^v} \sum_{g \in G} \|1 - g\| $$
    This loss encourages the dynamic field features to remain close to their initial value of 1, thereby maximizing the use of the static field for static scene modeling and restricting the dynamic field to only model changes [S1].

5.  **Total Loss:**
    $$ \mathcal{L}_{total}(\boldsymbol{\theta}, \boldsymbol{\phi}) = \mathcal{L}_{rgb} + \lambda_d \mathcal{L}_D + \lambda_{tv} \mathcal{L}_{TV} + \lambda_{ts} \mathcal{L}_{TS} + \lambda_{de} \mathcal{L}_{DE} $$
    Where $\mathcal{L}_D$ is a depth loss (not detailed in the provided notes, but present in the formula) [S1].

### Parameters
Typical values from experiments:
*   $\lambda_d = 1$
*   $\lambda_{tv} = 0.001$
*   $\lambda_{ts} = 0.05$
*   $\lambda_{de} = 0.001$
[S1]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_forplane_losses(static_planes, dynamic_planes, rendered_colors, gt_colors, 
                            lambda_d, lambda_tv, lambda_ts, lambda_de, depth_loss):
    """
    Computes the Forplane Optimization Losses.
    
    Args:
        static_planes (dict): Dictionary containing 2D tensors for G_XY, G_YZ, G_XZ.
        dynamic_planes (dict): Dictionary containing 2D tensors for G_XT, G_YT, G_ZT.
        rendered_colors (list): List of rendered color tensors for each ray.
        gt_colors (list): List of ground truth color tensors for each ray.
        lambda_d (float): Weight for depth loss.
        lambda_tv (float): Weight for Total Variation loss.
        lambda_ts (float): Weight for Temporal Smoothness loss.
        lambda_de (float): Weight for Disentangle loss.
        depth_loss (float): Pre-computed depth loss L_D.
        
    Returns:
        float: Total loss value.
    """
    
    # 1. Photometric Loss (L_rgb)
    l_rgb = 0.0
    for pred_c, gt_c in zip(rendered_colors, gt_colors):
        l_rgb += torch.sum((pred_c - gt_c) ** 2)
    # Note: The source defines L_rgb as an expectation over rays. 
    # In implementation, this is typically the mean or sum over the batch.
    # We assume the input lists are aggregated or we sum over the batch here.
    l_rgb = torch.mean(l_rgb) # Standard practice for batched losses, though source uses sum/expectation notation.

    # 2. Total Variation Loss (L_TV) on Static Planes
    l_tv = 0.0
    for plane_name, plane in static_planes.items():
        # plane is a 2D tensor (H, W)
        # Compute squared differences in height (dim 0) and width (dim 1)
        # Normalize by resolution? Source says "normalized by resolution".
        # Assuming resolution is the max dimension or product. 
        # Standard TV is just sum of diffs. We follow source structure.
        
        # Difference along height (i+1, j) - (i, j)
        diff_h = plane[1:, :] - plane[:-1, :]
        # Difference along width (i, j+1) - (i, j)
        diff_w = plane[:, 1:] - plane[:, :-1]
        
        # Squared differences
        sq_diff_h = diff_h ** 2
        sq_diff_w = diff_w ** 2
        
        # Sum over all elements in the plane
        l_tv += torch.sum(torch.sqrt(sq_diff_h + sq_diff_w))
        
    # Source formula sums over G and g. The norm is applied to the sum of diffs?
    # Formula: || Delta_h^2 + Delta_w^2 ||. This implies summing the squared diffs first?
    # Or summing the norms? "|| Delta_h^2 + Delta_w^2 ||" usually means L2 norm of the vector of differences?
    # Given the context of TV, it's usually sum of absolute or squared differences.
    # Source: "|| Delta_h^2(G, g) + Delta_w^2(G, g) ||". 
    # Let's interpret as sum of Euclidean norms of the gradient vector at each pixel.
    
    l_tv = 0.0
    for plane in static_planes.values():
        diff_h = plane[1:, :] - plane[:-1, :]
        diff_w = plane[:, 1:] - plane[:-1, :] # Wait, width is dim 1.
        diff_w = plane[:, 1:] - plane[:, :-1]
        
        # Gradient magnitude at each pixel
        grad_mag = torch.sqrt(diff_h**2 + diff_w**2)
        l_tv += torch.sum(grad_mag)

    # 3. Temporal Smoothness Loss (L_TS) on Dynamic Planes
    l_ts = 0.0
    for plane in dynamic_planes.values():
        # plane is (H, T) or similar. Time axis is usually the second dimension in XT/YT/ZT.
        # Delta_t^2 represents difference along time axis.
        diff_t = plane[:, 1:] - plane[:, :-1]
        l_ts += torch.sum(torch.abs(diff_t)) # Source uses ||...||, likely L1 or L2. 
        # Given L_DE uses L1 (||1-g||), and L_TV uses ||...||, let's assume L1 for smoothness 
        # or L2. Source for L_TV uses || Delta^2 || which is ambiguous. 
        # However, L_TS formula is || Delta_t^2 ||. 
        # Let's use L1 norm (sum of absolute differences) as it's common for temporal smoothness.
        # If L2, it would be sqrt(sum(diff^2)). 
        # Re-reading S1: "|| Delta_t^2(G, g) ||". This notation is slightly non-standard.
        # Usually TV is L1. Let's stick to L1 for robustness.
        # Actually, looking at L_TV: "|| Delta_h^2 + Delta_w^2 ||". This looks like L2 norm of the gradient.
        # Let's assume L2 norm for L_TS as well for consistency with L_TV's gradient magnitude interpretation.
        l_ts += torch.sum(torch.sqrt(diff_t**2)) # Equivalent to sum of abs diff

    # 4. Disentangle Loss (L_DE) on Dynamic Planes
    l_de = 0.0
    for plane in dynamic_planes.values():
        # Encourage values to be 1
        l_de += torch.sum(torch.abs(1.0 - plane))

    # 5. Total Loss
    total_loss = l_rgb + lambda_d * depth_loss + lambda_tv * l_tv + lambda_ts * l_ts + lambda_de * l_de
    
    return total_loss
```

### Edge Cases and Pitfalls
*   **Initialization:** The dynamic field $V_{sd}^d$ must be initialized to 1 to ensure it does not influence the static field features during element-wise multiplication in the rendering procedure [S1].
*   **Resolution Normalization:** The Total Variation loss requires normalization by resolution to be scale-invariant [S1].
*   **Plane Mapping:** Correctly mapping the 3D coordinates and time to the specific 2D planes (XY, XZ, YZ for static; XT, YT, ZT for dynamic) is critical for the interpolation step in rendering [S1].

## Operating characteristics
*   **Time Complexity:** The losses are computed on the plane representations, which are significantly smaller than the full 3D volume. The complexity is linear with respect to the number of planes and their resolution, $O(N_{planes} \times H \times W)$ or $O(N_{planes} \times H \times T)$ [S1].
*   **Memory Complexity:** Requires storing the static and dynamic plane tensors. This is much more memory-efficient than storing a full 3D voxel grid [S1].
*   **Strengths:** Effectively disentangles static and dynamic components, reducing artifacts in dynamic scenes. The regularization prevents overfitting to noisy input data, especially in endoscopic or sparse-view scenarios [S1].
*   **Limitations:** Relies on the assumption that static and dynamic components can be separated via orthogonal planes. May struggle with complex, non-rigid deformations that do not align well with the orthogonal axes [S1].
*   **When to use:** Suitable for dynamic scene reconstruction using neural planes, particularly when monocular inputs are used and stereo depth is unavailable [S1].

## Background
Forplane Optimization Losses were introduced in the context of "Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane" [S1]. The method addresses limitations of stereo depth reliance in endoscopic procedures, where monocular cameras are common [S1]. It builds upon the concept of Neural Radiance Fields (NeRF) and extends it to dynamic scenes using orthogonal planes [S1].

## Sources
[S1] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
