# Optimal Transport Loss

Also called: OT Loss, Optimal Transport based Loss Function, Sinkhorn Divergences Loss

## Overview
Optimal Transport (OT) Loss captures global and long-range correspondences between a rendered image $I$ and a target image $I_{ref}$ by solving a transportation problem, replacing local pixel-wise losses like L1 or L2 [S1]. It differs from [Data Consistency Loss](2976-data-consistency-loss.md) by optimizing for a global matching structure rather than point-wise residuals, and from [Monocular Depth Consistency Loss](4668-monocular-depth-consistency-loss.md) by operating directly in image space color and position rather than depth constraints.

## How it works
**Inputs:** Rendered image $I$ (resolution $N \times N$), target image $I_{ref}$ (resolution $M \times M$, typically $N=M$), scene parameters $\theta$.
**Outputs:** Scalar loss value $L$, gradients $\frac{\partial L}{\partial \theta}$.

**Data Structures:**
1.  **Cost Matrix $C$**: Size $N \times M$. Element $c(i,j)$ is the unit transportation cost.
2.  **Transportation Matrix $T$**: Size $N \times M$. Element $T_{ij}$ is the amount of mass transported from pixel $i$ in $I$ to pixel $j$ in $I_{ref}$.
3.  **Mass Vectors**: $x \in \mathbb{R}^N$ and $y \in \mathbb{R}^M$. In this application, $x_i = 1$ for all $i$, $y_j = 1$ for all $j$ [S1].

**Parameters:**
*   $\lambda \in [0, 1]$: Balancing weight for color vs. position cost. Typical value: $0.5$ [S1].
*   $\epsilon$: Regularization parameter for Sinkhorn divergence. Typical value: $0.01$ [S1].

**Steps:**
1.  **Compute Cost Matrix**: For each pixel $p_i$ in $I$ with color $c_i = I(p_i)$ and each pixel $p_j$ in $I_{ref}$ with color $c_j = I_{ref}(p_j)$, compute:
    $$c(i,j) = \lambda ||c_i - c_j||_2^2 + (1 - \lambda)||p_i - p_j||_2^2$$
    where $p_i, p_j$ are screen-space coordinates [S1].
2.  **Solve Optimal Transport**: Compute $T$ by minimizing $\sum_{i,j} T_{ij} c(i,j)$ subject to $\sum_j T_{ij} = 1$ and $\sum_i T_{ij} = 1$. Use the Sinkhorn algorithm with parameter $\epsilon$ [S1].
3.  **Compute Gradients w.r.t. Proxy Attributes**: Fix $T$. Compute gradients for color $c_k$ and position $p_k$ of the $k$-th point proxy (corresponding to pixel $k$ in $I$):
    $$\frac{\partial L}{\partial c_k} = \sum_{j} T_{kj} \frac{\partial c(k,j)}{\partial c_k}$$
    $$\frac{\partial L}{\partial p_k} = \sum_{j} T_{kj} \frac{\partial c(k,j)}{\partial p_k}$$
    Note: $\frac{\partial c(k,j)}{\partial c_k} = 2\lambda(c_k - c_j)$ and $\frac{\partial c(k,j)}{\partial p_k} = 2(1-\lambda)(p_k - p_j)$ [S1].
4.  **Chain Rule to Scene Parameters**: Compute final gradient w.r.t. scene parameter $\theta$ using RGBXY derivatives:
    $$\frac{\partial L}{\partial \theta} = \sum_{k=1}^N \left( \frac{\partial L}{\partial c_k} \frac{D c_k}{D \theta} + \frac{\partial L}{\partial p_k} \frac{\partial p_k}{\partial \theta} \right)$$
    where $\frac{D c_k}{D \theta}$ and $\frac{\partial p_k}{\partial \theta}$ are the standard derivatives of color and position w.r.t. $\theta$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimal_transport_loss(I, I_ref, theta, lambda_weight=0.5, epsilon=0.01):
    """
    I: Rendered image tensor, shape (H, W, 3)
    I_ref: Target image tensor, shape (H, W, 3)
    theta: Scene parameters
    lambda_weight: Balancing weight for color vs position
    epsilon: Sinkhorn regularization
    """
    # 1. Extract pixel attributes
    # H, W are assumed equal for simplicity as per source assumption N=M
    H, W, _ = I.shape
    N = H * W
    
    # Reshape to list of pixels
    # I_pixels: (N, 3) colors
    # I_pos: (N, 2) screen coordinates
    I_pixels = I.reshape(-1, 3)
    I_ref_pixels = I_ref.reshape(-1, 3)
    
    # Generate screen coordinates
    # Assuming pixel centers at (x + 0.5, y + 0.5) or similar standard grid
    # This is a placeholder for coordinate generation logic [G]
    coords = generate_screen_coordinates(H, W) # Shape (N, 2)
    
    # 2. Compute Cost Matrix C (N, N)
    # Color distance: lambda * ||c_i - c_j||^2
    # Position distance: (1-lambda) * ||p_i - p_j||^2
    
    # Efficient computation using broadcasting
    # diff_color: (N, N, 3)
    diff_color = I_pixels[:, None, :] - I_ref_pixels[None, :, :]
    color_dist = torch.sum(diff_color ** 2, dim=2) # (N, N)
    
    # diff_pos: (N, N, 2)
    diff_pos = coords[:, None, :] - coords[None, :, :]
    pos_dist = torch.sum(diff_pos ** 2, dim=2) # (N, N)
    
    C = lambda_weight * color_dist + (1 - lambda_weight) * pos_dist
    
    # 3. Solve Optimal Transport using Sinkhorn
    # Mass vectors are all ones
    u = torch.ones(N)
    v = torch.ones(N)
    
    # Sinkhorn iteration [G]
    # K = exp(-C / epsilon)
    # v = 1 / (K^T @ (1 / (K @ u)))
    # T = diag(u) @ K @ diag(v)
    # Note: Source cites Feydy et al. [2019] GPU implementation. 
    # Standard Sinkhorn loop for clarity:
    K = torch.exp(-C / epsilon)
    for _ in range(100): # Iterations for convergence
        u_new = 1.0 / (K @ v)
        v = 1.0 / (K.T @ u_new)
        u = u_new
    T = torch.diag(u) @ K @ torch.diag(v)
    
    # 4. Compute Gradients w.r.t. Color and Position
    # dL/dc_k = sum_j T_kj * d(c(k,j))/d(c_k)
    # d(c(k,j))/d(c_k) = 2 * lambda * (c_k - c_j)
    # dL/dp_k = sum_j T_kj * d(c(k,j))/d(p_k)
    # d(c(k,j))/d(p_k) = 2 * (1-lambda) * (p_k - p_j)
    
    # Gradient w.r.t. I_pixels (colors)
    # dL/dI_pixels[k] = sum_j T_kj * 2 * lambda * (I_pixels[k] - I_ref_pixels[j])
    # This can be computed as: 2 * lambda * (T @ I_ref_pixels - (sum(T, axis=1)[:, None] * I_pixels))
    # Since sum(T, axis=1) = 1 (mass constraint), this simplifies:
    dL_dI_pixels = 2 * lambda_weight * (T @ I_ref_pixels - I_pixels)
    
    # Gradient w.r.t. coords (positions)
    # dL/dcoords[k] = sum_j T_kj * 2 * (1-lambda) * (coords[k] - coords[j])
    # Similarly simplifies due to mass constraint:
    dL_dcoords = 2 * (1 - lambda_weight) * (T @ coords - coords)
    
    # 5. Chain Rule to Theta
    # dL/dtheta = sum_k ( dL/dc_k * Dc_k/Dtheta + dL/dp_k * dp_k/Dtheta )
    # Dc_k/Dtheta and dp_k/Dtheta are provided by the differentiable renderer
    # For each pixel k, we have gradients w.r.t theta from the renderer
    # Let's assume we have functions to get these derivatives
    
    grad_theta = torch.zeros_like(theta)
    
    for k in range(N):
        # Get derivatives of color and position of pixel k w.r.t theta
        # These depend on the specific rendering setup (e.g., Gaussian Splatting, NeRF)
        # dc_k_dtheta: shape (theta_dim,)
        # dp_k_dtheta: shape (theta_dim,)
        dc_k_dtheta, dp_k_dtheta = get_rgbxy_derivatives(k, theta)
        
        # Add contribution
        grad_theta += dL_dI_pixels[k] @ dc_k_dtheta + dL_dcoords[k] @ dp_k_dtheta
        
    return torch.sum(T * C), grad_theta
```

**Edge Cases and Pitfalls:**
*   **Precision**: Sinkhorn iterations may not converge exactly; $\epsilon$ controls accuracy vs. speed [S1].
*   **Boundaries**: The method assumes $N=M$ pixels for direct mapping [S1]. If resolutions differ, upsampling/downsampling or padding is required (not specified in source) [G].
*   **Degenerate Input**: If images are identical, $T$ approaches identity, gradients vanish.

## Subtopics

### Sinkhorn Divergences
An efficient approximation for Optimal Transport. Chosen for GPU efficiency over exact solvers. [Link](sinkhorn-divergences.md) (Hypothetical sibling link).

### RGBXY Derivatives
The derivative framework used to propagate OT gradients to scene parameters. [Link](rgbxy-derivatives.md) (Hypothetical sibling link).

## Operating characteristics
*   **Time Complexity**: $O(NM)$ for the approximated optimal transport algorithm, where $N, M$ are total pixel amounts [S1].
*   **Measured Performance**: On NVIDIA RTX 3090:
    *   128x128 images: ~0.05s [S1].
    *   256x256 images: 0.43s [S1].
    *   512x512 images: 5.72s [S1].
*   **Strengths**: Captures global and long-range correspondences, avoiding local minima associated with pixel-wise losses [S1].
*   **Limitations**: Computationally expensive for high resolutions compared to L1/L2. Requires differentiable rendering framework to compute RGBXY derivatives [S1].
*   **When to Use**: When global structure preservation is critical and computational budget allows for $O(N^2)$ operations.
*   **When Not to Use**: For real-time applications or very high-resolution images where $O(N^2)$ is prohibitive.

## Background
The method was introduced by Jiankai Xing et al. in "Differentiable Rendering Using RGBXY Derivatives and Optimal Transport" (2022) [S1]. It addresses the limitation of standard pixel-wise losses in differentiable rendering by incorporating global matching. The implementation relies on the GPU Sinkhorn divergence library by Feydy et al. [2019] [S1].

## Sources
[S1] Differentiable Rendering Using RGBXY Derivatives and Optimal Transport, https://doi.org/10.1145/3550454.3555479, fetched 2026-10-27
