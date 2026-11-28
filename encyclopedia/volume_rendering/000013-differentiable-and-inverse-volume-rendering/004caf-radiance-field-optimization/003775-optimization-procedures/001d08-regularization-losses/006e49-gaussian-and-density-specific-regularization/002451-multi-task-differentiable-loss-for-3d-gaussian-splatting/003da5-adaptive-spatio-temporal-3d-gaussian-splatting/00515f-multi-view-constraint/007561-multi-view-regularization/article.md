# Multi-view Regularization

Also called: Cycle Reprojection Regularizer, Geometric Multi-View Term, PGSR Multi-view Regularization

## Overview
Multi-view Regularization is a technique that penalizes cycle reprojection error to enforce multi-view geometric consistency during the optimization of radiance fields [S1, unofficial]. It sets itself apart from its sibling [Multi-View Constraint](14458-multi-view-constraint.md) by explicitly computing a forward-backward reprojection cycle error to handle occlusions and unreliable correspondences, rather than relying solely on patch matching and homography warping for color consistency [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Reference image $I_r$, neighboring image $I_n$, camera intrinsics $\mathbf{K}_r, \mathbf{K}_n$, relative pose $(\mathbf{R}_{rn}, \mathbf{T}_{rn})$ from reference to neighbor, rendered depth map $D_r$, and rendered normal map $\mathbf{N}_r$ for the reference view.
*   **Outputs:** Regularization loss $L_{mv}$ and its gradients with respect to the scene parameters (depth, normals, or Gaussian attributes).

**Data Structures:**
*   **Images:** $I_r, I_n \in \mathbb{R}^{H \times W \times 3}$ (RGB).
*   **Depth:** $D_r \in \mathbb{R}^{H \times W}$ (depth along the ray).
*   **Normals:** $\mathbf{N}_r \in \mathbb{R}^{H \times W \times 3}$ (surface normal in camera space).
*   **Homography:** $\mathbf{H}_{rn} \in \mathbb{R}^{3 \times 3}$ matrix.

**Parameters:**
*   $w_{pc}$: Weight for photometric consistency. Typical value: $0.6$ [S1, unofficial].
*   $w_{gc}$: Weight for geometric consistency. Typical value: $0.02$ [S1, unofficial].
*   Threshold for confidence weight: $1.0$ pixel [S1, unofficial].

**Steps:**

1.  **Back-project to 3D:** For each pixel $\mathbf{u}_r = (u, v)$ in the reference image, retrieve the rendered depth $d_r = D_r(\mathbf{u}_r)$ and normal $\mathbf{n}_r = \mathbf{N}_r(\mathbf{u}_r)$. Compute the 3D point $\mathbf{p}_r$ in the reference camera frame:
    $$ \mathbf{p}_r = d_r \cdot \mathbf{K}_r^{-1} \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} $$
    [S1, unofficial]

2.  **Compute Plane Homography:** Approximate the local surface as a plane defined by $\mathbf{p}_r$ and $\mathbf{n}_r$. Compute the homography $\mathbf{H}_{rn}$ that maps points from the reference view to the neighboring view:
    $$ \mathbf{H}_{rn} = \mathbf{K}_n \left( \mathbf{R}_{rn} + \frac{\mathbf{T}_{rn} \mathbf{n}_r^\top}{\mathbf{p}_r^\top \mathbf{n}_r} \right) \mathbf{K}_r^{-1} $$
    where $\mathbf{R}_{rn}, \mathbf{T}_{rn}$ are the rotation and translation from reference to neighbor. [S1, unofficial]

3.  **Compute Forward-Backward Cycle Error:** To handle occlusions, compute the homography $\mathbf{H}_{rr}$ mapping from the neighboring view back to the reference view (using inverse relative pose and potentially neighbor depth/normal if available, or assuming the same local plane approximation). Calculate the cycle reprojection error $\phi(\mathbf{u}_r)$:
    $$ \phi(\mathbf{u}_r) = \|\mathbf{u}_r - \mathbf{H}_{rr} \mathbf{H}_{rn} \mathbf{u}_r\|_2 $$
    [S1, unofficial]

4.  **Compute Confidence Weight:** Discard pixels with large cycle errors (likely occluded or mismatched):
    $$ w(\mathbf{u}_r) = \begin{cases} \exp(-\phi(\mathbf{u}_r)) & \text{if } \phi(\mathbf{u}_r) < 1 \\ 0 & \text{otherwise} \end{cases} $$
    [S1, unofficial]

5.  **Compute Photometric Consistency Loss ($L_{pc}$):** Warp the neighboring image $I_n$ to the reference frame using $\mathbf{H}_{rn}$ and enforce patch-level photometric consistency via Normalized Cross-Correlation (NCC):
    $$ L_{pc} = \sum_{\mathbf{u}_r} w(\mathbf{u}_r) \left( 1 - \text{NCC}(I_r(\mathbf{u}_r), I_n(\mathbf{H}_{rn}\mathbf{u}_r)) \right) $$
    [S1, unofficial]

6.  **Compute Geometric Consistency Loss ($L_{gc}$):** Penalize the cycle reprojection error directly:
    $$ L_{gc} = \sum_{\mathbf{u}_r} w(\mathbf{u}_r) \phi(\mathbf{u}_r) $$
    [S1, unofficial]

7.  **Combine Losses:** The total multi-view regularization loss is:
    $$ L_{mv} = w_{pc} L_{pc} + w_{gc} L_{gc} $$
    [S1, unofficial]

8.  **Gradient Accumulation:** Accumulate gradients using a custom CUDA kernel for efficiency [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_view_regularization(I_r, I_n, K_r, K_n, R_rn, T_rn, D_r, N_r, w_pc=0.6, w_gc=0.02):
    """
    Computes the multi-view regularization loss.
    
    Args:
        I_r: Reference image (H, W, 3)
        I_n: Neighboring image (H, W, 3)
        K_r: Reference camera intrinsics (3, 3)
        K_n: Neighbor camera intrinsics (3, 3)
        R_rn: Rotation matrix from ref to neighbor (3, 3)
        T_rn: Translation vector from ref to neighbor (3,)
        D_r: Rendered depth map (H, W)
        N_r: Rendered normal map (H, W, 3)
        w_pc: Weight for photometric consistency
        w_gc: Weight for geometric consistency
        
    Returns:
        L_mv: Scalar loss value
    """
    H, W = I_r.shape[:2]
    
    # 1. Back-project to 3D
    # Create grid of pixel coordinates
    u, v = torch.meshgrid(torch.arange(W), torch.arange(H), indexing='xy')
    ones = torch.ones(H, W)
    pixel_coords = torch.stack([u, v, ones], dim=-1).float() # (H, W, 3)
    
    # Unproject to 3D in reference camera frame
    # p_r = d_r * K_r^-1 * [u, v, 1]^T
    K_r_inv = torch.linalg.inv(K_r)
    pixel_coords_hom = pixel_coords.unsqueeze(-1) # (H, W, 3, 1)
    p_r_hom = torch.matmul(K_r_inv, pixel_coords_hom).squeeze(-1) # (H, W, 3)
    p_r = D_r.unsqueeze(-1) * p_r_hom # (H, W, 3)
    
    n_r = N_r # (H, W, 3)
    
    # 2. Compute Plane Homography H_rn
    # H_rn = K_n * (R_rn + T_rn * n_r^T / (p_r^T * n_r)) * K_r^-1
    # Denominator: p_r dot n_r
    denom = torch.sum(p_r * n_r, dim=-1, keepdim=True) # (H, W, 1)
    # Avoid division by zero
    denom = torch.clamp(denom, min=1e-8)
    
    # Outer product T_rn * n_r^T
    T_rn_col = T_rn.unsqueeze(0).unsqueeze(0) # (1, 1, 3)
    n_r_row = n_r.unsqueeze(-1) # (H, W, 3, 1)
    T_n_outer = T_rn_col * n_r_row # (H, W, 3, 3)
    
    # Term inside parenthesis
    inner_term = R_rn + T_n_outer / denom.unsqueeze(-1) # (H, W, 3, 3)
    
    # H_rn = K_n @ inner_term @ K_r_inv
    H_rn = torch.matmul(K_n, torch.matmul(inner_term, K_r_inv)) # (H, W, 3, 3)
    
    # 3. Compute Forward-Backward Cycle Error
    # We need H_rr (neighbor to ref). Assuming symmetric plane approximation or using inverse.
    # For simplicity, if H_rr is not explicitly computed from neighbor depth, 
    # we might approximate it or assume the same local plane holds.
    # However, the formula uses H_rr * H_rn. 
    # Let's assume we compute H_nr (neighbor to ref) similarly using neighbor depth/normal if available.
    # If not, we might use the inverse of H_rn as an approximation, but this is less accurate.
    # The source mentions H_rr maps from neighbor back to ref.
    # Let's assume we have D_n, N_n for the neighbor to compute H_nr accurately.
    # If not, this step is incomplete. Assuming D_n, N_n are available for robustness.
    
    # Re-compute p_n and n_n for neighbor
    # p_n = d_n * K_n^-1 * [u, v, 1]^T
    K_n_inv = torch.linalg.inv(K_n)
    p_n_hom = torch.matmul(K_n_inv, pixel_coords_hom).squeeze(-1)
    p_n = D_n.unsqueeze(-1) * p_n_hom # (H, W, 3)
    n_n = N_n # (H, W, 3)
    
    # H_nr = K_r * (R_nr + T_nr * n_n^T / (p_n^T * n_n)) * K_n^-1
    # R_nr = R_rn^T, T_nr = -R_rn^T * T_rn
    R_nr = R_rn.transpose(-2, -1)
    T_nr = -torch.matmul(R_nr, T_rn)
    
    denom_n = torch.sum(p_n * n_n, dim=-1, keepdim=True)
    denom_n = torch.clamp(denom_n, min=1e-8)
    
    T_nr_col = T_nr.unsqueeze(0).unsqueeze(0)
    n_n_row = n_n.unsqueeze(-1)
    T_n_n_outer = T_nr_col * n_n_row
    
    inner_term_nr = R_nr + T_n_n_outer / denom_n.unsqueeze(-1)
    H_nr = torch.matmul(K_r, torch.matmul(inner_term_nr, K_n_inv))
    
    # Cycle error: u_r - H_nr * H_rn * u_r
    # Warp u_r using H_rn to get u_n
    # Then warp u_n using H_nr to get u_r'
    # u_r' = H_nr * H_rn * u_r
    
    # Vectorized homography warp
    def warp_grid(H, grid):
        # grid: (H, W, 2)
        # H: (H, W, 3, 3)
        # Returns warped grid (H, W, 2)
        # Convert grid to homogeneous (H, W, 3)
        grid_h = torch.cat([grid, torch.ones(H, W, 1)], dim=-1) # (H, W, 3)
        # Apply H
        # grid_h is (H, W, 3), H is (H, W, 3, 3)
        # result = H @ grid_h^T -> (H, W, 3, 3) @ (H, W, 3, 1) -> (H, W, 3, 1)
        warped_h = torch.matmul(H, grid_h.unsqueeze(-1)).squeeze(-1) # (H, W, 3)
        # Normalize
        warped = warped_h[..., :2] / torch.clamp(warped_h[..., 2:], min=1e-8) # (H, W, 2)
        return warped

    grid_ref = torch.stack([u.float(), v.float()], dim=-1) # (H, W, 2)
    grid_neighbor = warp_grid(H_rn, grid_ref) # (H, W, 2)
    grid_cycle = warp_grid(H_nr, grid_neighbor) # (H, W, 2)
    
    phi = torch.norm(grid_ref - grid_cycle, dim=-1) # (H, W)
    
    # 4. Confidence Weight
    mask = phi < 1.0
    w = torch.where(mask, torch.exp(-phi), torch.zeros_like(phi))
    
    # 5. Photometric Loss
    # Warp I_n to reference frame using H_rn
    # I_n_warped = sample(I_n, grid_neighbor)
    # NCC calculation
    # For simplicity, assume NCC is computed over small patches or per-pixel if normalized.
    # The source says "patch-level photometric consistency via NCC".
    # Implementing full patch NCC is complex. We'll approximate with per-pixel if patch size not specified.
    # Or assume a small kernel. Let's assume per-pixel for the formula structure, 
    # noting that in practice a patch is used.
    
    # Normalize images to [0, 1]
    I_r_norm = I_r / 255.0
    I_n_warped = grid_sample(I_n, grid_neighbor, mode='bilinear', padding_mode='zeros')
    I_n_warped = I_n_warped / 255.0
    
    # Per-pixel NCC (simplified)
    # NCC(a, b) = (a - mean(a))(b - mean(b)) / (std(a) * std(b))
    # If patch size is 1, NCC is sign(a*b) if non-zero, which is unstable.
    # The source likely uses a small patch (e.g., 3x3 or 5x5).
    # Since patch size is not given, we note this as a gap [G].
    # We will assume a function `compute_ncc_patch` exists.
    
    ncc = compute_ncc_patch(I_r_norm, I_n_warped, patch_size=3) # (H, W)
    L_pc = torch.sum(w * (1 - ncc))
    
    # 6. Geometric Loss
    L_gc = torch.sum(w * phi)
    
    # 7. Combine
    L_mv = w_pc * L_pc + w_gc * L_gc
    
    return L_mv

def compute_ncc_patch(I_r, I_n, patch_size=3):
    """
    Computes NCC between I_r and I_n over local patches.
    """
    # Pad images
    pad = patch_size // 2
    I_r_pad = F.pad(I_r, (pad, pad, pad, pad, 0, 0), mode='reflect')
    I_n_pad = F.pad(I_n, (pad, pad, pad, pad, 0, 0), mode='reflect')
    
    # Extract patches
    # I_r_patches: (H, W, C, patch_size, patch_size)
    I_r_patches = F.unfold(I_r_pad, kernel_size=patch_size, stride=1) # (H*W, C*P*P)
    I_n_patches = F.unfold(I_n_pad, kernel_size=patch_size, stride=1)
    
    # Reshape to (H, W, C*P*P)
    H, W = I_r.shape[:2]
    I_r_patches = I_r_patches.view(H, W, -1)
    I_n_patches = I_n_patches.view(H, W, -1)
    
    # Flatten patches to vectors
    I_r_flat = I_r_patches.view(H, W, -1)
    I_n_flat = I_n_patches.view(H, W, -1)
    
    # Compute mean and std
    mean_r = torch.mean(I_r_flat, dim=-1, keepdim=True)
    mean_n = torch.mean(I_n_flat, dim=-1, keepdim=True)
    
    std_r = torch.std(I_r_flat, dim=-1, keepdim=True, unbiased=False)
    std_n = torch.std(I_n_flat, dim=-1, keepdim=True, unbiased=False)
    
    # Avoid division by zero
    std_r = torch.clamp(std_r, min=1e-8)
    std_n = torch.clamp(std_n, min=1e-8)
    
    # NCC
    ncc = torch.mean(((I_r_flat - mean_r) * (I_n_flat - mean_n)) / (std_r * std_n), dim=-1)
    
    return ncc
```

**Edge Cases and Pitfalls:**
*   **Occlusions:** The cycle reprojection error $\phi(\mathbf{u}_r)$ becomes large for occluded pixels. The confidence weight $w(\mathbf{u}_r)$ discards these pixels ($\phi \ge 1$), preventing incorrect gradients. [S1, unofficial]
*   **Depth/Normal Accuracy:** The homography $\mathbf{H}_{rn}$ relies on accurate rendered depth and normals. Errors in these estimates lead to incorrect warping and false cycle errors. [S1, unofficial]
*   **Precision:** Floating-point precision in homography multiplication and warping can introduce small errors. The threshold of 1 pixel is used to mitigate this. [S1, unofficial]
*   **Textureless Regions:** While the method helps in textureless regions by enforcing geometric consistency, it may struggle if depth/normal estimates are ambiguous. [S1, unofficial]

## Subtopics

### Photometric Consistency Term ($L_{pc}$)
Enforces color similarity between warped views. Sets itself apart by using NCC for robustness to lighting changes. Choose when lighting is consistent. [S1, unofficial]

### Geometric Consistency Term ($L_{gc}$)
Enforces geometric consistency via cycle reprojection error. Sets itself apart by directly penalizing geometric inconsistency. Choose when geometry is the primary concern. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** Depends on the number of pixels and the cost of homography warping and NCC computation. The custom CUDA kernel improves efficiency [S1, unofficial].
*   **Memory Complexity:** Requires storing intermediate warping grids and patches. [S1, unofficial]
*   **Measured Performance:** On DTU dataset, multi-view regularizers substantially boost accuracy [S1, unofficial]. The method is faster than GeoSVR (15 vs. 53 min) and PGSR (25 vs. 30 min) due to efficient implementation [S1, unofficial].
*   **Strengths:** Improves reconstruction accuracy, particularly on datasets like DTU. Handles occlusions via cycle error. [S1, unofficial]
*   **Limitations:** Marginal gains when combined with depth-rendering formulations that already enforce strong multi-view consistency (e.g., RaDe-GS) [S1, unofficial]. Relies on accurate depth and normal estimates. [S1, unofficial]
*   **When to Use:** When multi-view consistency is critical and depth/normal estimates are reliable. Particularly useful for datasets like DTU. [S1, unofficial]
*   **When Not to Use:** When the underlying depth-rendering formulation already provides strong multi-view consistency, as the additional term may offer only marginal gains. [S1, unofficial]

## Background
Multi-view Regularization, specifically the cycle reprojection variant, was introduced in the context of Gaussian Splatting optimization. It was notably used in PGSR (Photometric and Geometric Regularization for Gaussian Splatting) [S1, unofficial]. The method builds on the idea of enforcing consistency across multiple views to improve geometry, particularly in textureless regions. It has been applied to datasets like DTU and Tanks & Temples [S1, unofficial].

## Sources
[S1] Geometry-Grounded Gaussian Splatting, https://arxiv.org/abs/2601.17835v2, fetched 2026-10-26 (unofficial)
