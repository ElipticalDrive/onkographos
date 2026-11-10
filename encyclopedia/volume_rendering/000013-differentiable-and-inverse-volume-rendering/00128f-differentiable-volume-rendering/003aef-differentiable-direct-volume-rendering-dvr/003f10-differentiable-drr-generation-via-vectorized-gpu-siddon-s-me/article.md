# Differentiable DRR Generation via Vectorized GPU Siddon's Method

Also called: Auto-Differentiable DRR, Auto-differentiable DRR Generation via Vectorized GPU Siddon's Method, DiffDRR, VGS, Vectorized GPU Siddon's method, Vectorized Siddon's Method for DRR Generation

## Overview
This method generates Digitally Reconstructed Radiographs (DRRs) that are fully differentiable with respect to 7 geometric imaging parameters by reformulating Siddon's ray-tracing algorithm as a series of vectorized tensor operations within PyTorch [S1]. It solves the problem of slow gradient-based optimization in intraoperative imaging by leveraging automatic differentiation, avoiding the high memory costs of standard reverse-mode AD and the complexity of adjoint methods [S1]. Unlike iterative ray-tracing implementations, it computes all plane intersections in parallel using GPU tensor algebra, trading memory efficiency for speed and compatibility with deep learning frameworks [S1].

## How it works
**Inputs:**
*   $\mathbf{V}$: A discrete 3D CT volume (tensor).
*   $\eta = (\rho, \theta, \phi, \gamma, b_x, b_y, b_z)$: Imaging parameters.
    *   $\rho$: Radius (half source-to-detector distance).
    *   $\theta, \phi$: Azimuthal and polar angles of the X-ray source $\mathbf{s}$.
    *   $\gamma$: Rotation of the detector plane about the x-axis.
    *   $b_x, b_y, b_z$: Translation of the source and detector plane relative to the volume center.
*   $H, W$: Desired DRR dimensions.
*   $n_x, n_y, n_z$: Dimensions of the CT volume $\mathbf{V}$.
*   $\Delta X, \Delta Y, \Delta Z$: Voxel sizes of $\mathbf{V}$.

**Outputs:**
*   $\mathbf{I} \in \mathbb{R}^{H \times W}$: The DRR intensity image.
*   Gradients $\nabla_\eta \mathbf{I}$: Automatically computed by the framework.

**Data Structures:**
*   $\mathbf{P} \in \mathbb{R}^{H \times W \times 3}$: Tensor of 3D coordinates for each pixel on the detector plane.
*   $\mathbf{A} \in \mathbb{R}^{H \times W \times (n_x + n_y + n_z)}$: Tensor containing intersection parameters $\alpha$ for all x, y, and z planes for all pixels.
*   $\alpha_{\min}, \alpha_{\max} \in \mathbb{R}^{H \times W}$: Entry and exit bounds for each ray.

**Steps:**

1.  **Define Ray Geometry:**
    The ray $\mathbf{R}(\alpha)$ originates from the source $\mathbf{s}$ and passes through pixel $\mathbf{p}$ on the detector:
    $$ \mathbf{R}(\alpha) = \mathbf{s} + \alpha(\mathbf{p} - \mathbf{s}) $$
    where $\alpha=0$ at the source and $\alpha=1$ at the detector [S1].

2.  **Compute Intersection Parameters:**
    Calculate the intersection parameters $\alpha$ for all pixels with all orthogonal planes of the volume. This is done in parallel using tensor operations [S1]:
    $$ \mathbf{A} = \begin{pmatrix} b_x \\ b_y \\ b_z \end{pmatrix} + \begin{pmatrix} i \\ j \\ k \end{pmatrix} \otimes \begin{pmatrix} \Delta X \\ \Delta Y \\ \Delta Z \end{pmatrix} - \mathbf{s} \otimes (\mathbf{P} - \mathbf{s}) $$
    where $(i, j, k)$ are voxel indices, $\otimes$ is the Hadamard (element-wise) product, and division is implied for the term involving $\mathbf{P}-\mathbf{s}$ in the original derivation context (specifically, the term represents the inverse direction cosine scaling, but the source formula (5) explicitly writes it as a product with the inverse direction or assumes normalized direction components embedded in the structure; strictly following source [S1] Eq 5, the operation is defined as shown, where the division is part of the geometric derivation of $\alpha$ values relative to the source position). *Note: The source formula (5) uses $\otimes$ and division. In standard Siddon's, $\alpha = (x_{plane} - x_s) / (x_p - x_s)$. The source notation simplifies this into a tensor broadcast operation.* [S1].

3.  **Determine Entry and Exit Bounds:**
    Calculate the minimum and maximum $\alpha$ values corresponding to the ray entering and exiting the volume bounds for each pixel [S1]:
    $$ \alpha_{\min} = \max \{ \min\{\alpha_x(0), \alpha_x(n_x)\}, \min\{\alpha_y(0), \alpha_y(n_y)\}, \min\{\alpha_z(0), \alpha_z(n_z)\} \} $$
    $$ \alpha_{\max} = \min \{ \max\{\alpha_x(0), \alpha_x(n_x)\}, \max\{\alpha_y(0), \alpha_y(n_y)\}, \max\{\alpha_z(0), \alpha_z(n_z)\} \} $$
    where $\alpha_x(0), \alpha_x(n_x)$ etc. are the $\alpha$ values for the planes at index 0 and $n_x$ (and similarly for y, z) [S1].

4.  **Filter and Sort Intersections:**
    Filter tensor $\mathbf{A}$ to include only values in the range $[\alpha_{\min}, \alpha_{\max}]$. Sort each row $\mathbf{A}(h, w, \cdot)$ for $h \in \{1, \dots, H\}, w \in \{1, \dots, W\}$ to obtain the ordered sequence of plane intersections [S1].

5.  **Evaluate Intensity:**
    Compute the intensity for each pixel using the discrete summation formula [S1]:
    $$ E(R) = \|\mathbf{p} - \mathbf{s}\|_2 \sum_{m=1}^{M-1} (\alpha_{m+1} - \alpha_m) \mathbf{V} \left[ \mathbf{s} + \frac{\alpha_{m+1} + \alpha_m}{2} (\mathbf{p} - \mathbf{s}) \right] $$
    where $\alpha_m$ are the sorted intersection parameters and $M$ is the number of intersections. The term $\mathbf{V}[\dots]$ samples the volume at the midpoint of the ray segment between intersections [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def generate_diff_drr(V, eta, H, W, nx, ny, nz, dX, dY, dZ):
    """
    V: Volume tensor (nx, ny, nz)
    eta: Parameters (rho, theta, phi, gamma, bx, by, bz)
    H, W: Output dimensions
    nx, ny, nz: Volume dimensions
    dX, dY, dZ: Voxel sizes
    """
    # 1. Extract parameters
    rho, theta, phi, gamma, bx, by, bz = eta
    
    # 2. Compute Source Position s
    # s = (rho, theta, phi) in spherical coordinates converted to Cartesian
    # Assuming standard physics convention: x = r sin(theta) cos(phi), etc.
    # Note: The source defines s = (rho, theta, phi) and detector tangent to sphere.
    # We need the Cartesian coordinates of s.
    sx = rho * torch.sin(theta) * torch.cos(phi)
    sy = rho * torch.sin(theta) * torch.sin(phi)
    sz = rho * torch.cos(theta)
    s = torch.stack([sx, sy, sz], dim=-1) # (3,)
    
    # Add translation
    s = s + torch.tensor([bx, by, bz])
    
    # 3. Define Detector Plane Pixels P
    # Create a grid of points on the detector plane.
    # The detector plane is tangent to the sphere at the point opposite s.
    # Center of sphere is origin (0,0,0). Point opposite s is -s/|s| * rho? 
    # Actually, s is at distance rho from origin? "rho is half of the source-to-detector distance"
    # "s = (rho, theta, phi)" implies s is on a sphere of radius rho? 
    # "detector plane is tangent to this implied sphere at the point opposite s"
    # Point opposite s on sphere radius rho is -s.
    # Normal to detector plane is -s (pointing from center to detector? or s to center?).
    # If s is source, ray goes s -> detector. Center is origin.
    # Vector from center to s is s. Vector from center to detector contact is -s.
    # Normal n = -s / |s|.
    
    # Generate base grid in local detector coordinates
    # Detector plane orientation determined by rotation gamma about x-axis.
    # Let's define local detector axes.
    # Normal n = -s / rho.
    # We need two orthogonal vectors u, v in the plane.
    # Rotate standard basis vectors by gamma around x-axis, then align with n.
    # This is complex. Simplified: The source assumes a specific coordinate system.
    # We will assume P is generated such that it lies on the plane tangent to -s.
    
    # For implementation simplicity, we generate P in a canonical frame and transform.
    # Canonical frame: Detector at z = -rho, normal (0,0,-1).
    # Points: x in [-W/2, W/2], y in [-H/2, H/2].
    # Transform to world frame:
    # 1. Rotate by gamma around X.
    # 2. Rotate to align Z-axis with -s.
    
    # Create meshgrid
    y_coords = torch.linspace(-W/2, W/2, W, device=V.device)
    x_coords = torch.linspace(-H/2, H/2, H, device=V.device)
    xx, yy = torch.meshgrid(x_coords, y_coords, indexing='ij')
    
    # Local detector coordinates (z is fixed at -rho relative to center? No, s is at rho)
    # If s is at distance rho from origin, and detector is tangent at -s,
    # the distance from origin to detector is rho.
    # So in local frame where detector is at z=-rho:
    zz = -rho * torch.ones_like(xx)
    
    P_local = torch.stack([xx, yy, zz], dim=-1) # (H, W, 3)
    
    # Rotation matrix for gamma around X
    cos_g = torch.cos(gamma)
    sin_g = torch.sin(gamma)
    R_x = torch.tensor([[1, 0, 0],
                        [0, cos_g, -sin_g],
                        [0, sin_g, cos_g]], device=V.device)
    
    # Rotate P_local
    P_rot = torch.einsum('ij,hwj->hwi', R_x, P_local)
    
    # Now rotate P_rot so that the normal (0,0,-1) aligns with -s/rho.
    # This requires a rotation matrix that maps (0,0,-1) to -s/rho.
    # Let n_target = -s / rho.
    n_target = -s / rho
    # We need a rotation matrix R_align such that R_align @ [0,0,-1]^T = n_target.
    # This is non-trivial to write without SVD or complex axis-angle.
    # However, for the purpose of the article, we assume P is correctly computed.
    # [G] In a real implementation, one would compute the rotation matrix aligning the z-axis to n_target.
    # For this pseudocode, we assume a function `align_to_vector` exists or use a simplified case.
    # To keep it self-contained and valid Python, we'll skip the full rotation matrix derivation 
    # and assume P_world is computed correctly.
    
    # P_world = P_rot transformed to align with s.
    # Let's assume P is already in world coordinates for the sake of the tensor operations.
    P = P_world # Placeholder for correct world coordinates
    
    # 4. Compute Intersection Parameters A
    # Indices i, j, k
    i_idx = torch.arange(nx, device=V.device).float()
    j_idx = torch.arange(ny, device=V.device).float()
    k_idx = torch.arange(nz, device=V.device).float()
    
    # Broadcast to (H, W, nx+ny+nz)
    # This part is tricky in pure tensor ops without explicit loops or advanced indexing.
    # The source formula (5) implies a specific broadcasting.
    # A = [bx, by, bz] + [i, j, k] ⊗ [dX, dY, dZ] - s ⊗ (P - s)
    # Note: The source formula has a division implied in the last term for standard Siddon's,
    # but writes it as product. We follow the source's tensor structure.
    
    # Create tensors for i, j, k
    # i varies along nx, j along ny, k along nz.
    # We need to stack them into a tensor of shape (nx+ny+nz, 3) or similar.
    
    # Let's construct the plane coordinates tensor.
    # x-planes: x = i * dX + bx, y=0, z=0? No, planes are orthogonal.
    # Plane x=i: normal (1,0,0).
    # The formula in source [S1] Eq 5 is:
    # A = [bx, by, bz] + [i, j, k] ⊗ [dX, dY, dZ] - s ⊗ (P - s)
    # This notation is ambiguous without context of how i,j,k are expanded.
    # Standard Siddon's: alpha = (x_plane - x_s) / (x_p - x_s).
    # The source likely means:
    # For x-planes: alpha_x = (i*dX + bx - s_x) / (P_x - s_x)
    # For y-planes: alpha_y = (j*dY + by - s_y) / (P_y - s_y)
    # For z-planes: alpha_z = (k*dZ + bz - s_z) / (P_z - s_z)
    
    # We compute these three tensors and concatenate.
    
    # Denominators
    dx_ray = P[..., 0] - s[0]
    dy_ray = P[..., 1] - s[1]
    dz_ray = P[..., 2] - s[2]
    
    # Avoid division by zero
    dx_ray = torch.where(dx_ray == 0, 1e-9, dx_ray)
    dy_ray = torch.where(dy_ray == 0, 1e-9, dy_ray)
    dz_ray = torch.where(dz_ray == 0, 1e-9, dz_ray)
    
    # X-planes
    alpha_x = (i_idx * dX + bx - s[0]) / dx_ray # Shape: (nx, H, W)
    # Y-planes
    alpha_y = (j_idx * dY + by - s[1]) / dy_ray # Shape: (ny, H, W)
    # Z-planes
    alpha_z = (k_idx * dZ + bz - s[2]) / dz_ray # Shape: (nz, H, W)
    
    # Concatenate along last dimension
    # Reshape to (H, W, N)
    alpha_x = alpha_x.permute(1, 2, 0) # (H, W, nx)
    alpha_y = alpha_y.permute(1, 2, 0) # (H, W, ny)
    alpha_z = alpha_z.permute(1, 2, 0) # (H, W, nz)
    
    A = torch.cat([alpha_x, alpha_y, alpha_z], dim=-1) # (H, W, nx+ny+nz)
    
    # 5. Determine Bounds
    # Min/Max for each axis
    # alpha_x(0) and alpha_x(nx)
    ax_0 = alpha_x[..., 0]
    ax_n = alpha_x[..., -1]
    ay_0 = alpha_y[..., 0]
    ay_n = alpha_y[..., -1]
    az_0 = alpha_z[..., 0]
    az_n = alpha_z[..., -1]
    
    min_x = torch.min(ax_0, ax_n)
    max_x = torch.max(ax_0, ax_n)
    min_y = torch.min(ay_0, ay_n)
    max_y = torch.max(ay_0, ay_n)
    min_z = torch.min(az_0, az_n)
    max_z = torch.max(az_0, az_n)
    
    alpha_min = torch.max(torch.max(min_x, min_y), min_z)
    alpha_max = torch.min(torch.min(max_x, max_y), max_z)
    
    # 6. Filter and Sort
    mask = (A >= alpha_min.unsqueeze(-1)) & (A <= alpha_max.unsqueeze(-1))
    # Mask out invalid values for sorting
    A_filtered = A.clone()
    A_filtered[~mask] = float('inf')
    
    # Sort
    A_sorted, _ = torch.sort(A_filtered, dim=-1)
    
    # 7. Evaluate Intensity
    # Sum over m=1 to M-1
    # We need to handle the variable number of intersections.
    # We can use the sorted tensor and mask out inf values.
    
    # Get unique valid alpha values per pixel? Or just iterate?
    # Vectorized approach:
    # Create a mask of valid intersections
    valid_mask = A_sorted != float('inf')
    
    # Compute segment lengths
    # alpha_{m+1} - alpha_m
    # Pad with 0 at start? No, we sum over segments.
    # We need to sample V at midpoints.
    
    # This step is complex to vectorize fully without loops over M.
    # However, PyTorch allows gathering.
    # Let's assume we extract the valid alphas into a list or padded tensor.
    # For simplicity, we use a loop over the max possible intersections (nx+ny+nz).
    
    intensity = torch.zeros((H, W), device=V.device)
    
    # Precompute ray direction vector
    ray_dir = P - s # (H, W, 3)
    ray_len = torch.norm(ray_dir, dim=-1) # (H, W)
    
    # Iterate over sorted intersections
    # We need pairs (alpha_m, alpha_{m+1})
    # We can shift the tensor
    
    # Replace inf with 0 for calculation, but mask them out later
    A_safe = torch.where(valid_mask, A_sorted, 0.0)
    
    # Shift to get next alpha
    A_next = torch.roll(A_safe, shifts=-1, dims=-1)
    # The last element in A_next will be the first element of A_safe (wrap around)
    # We need to mask the last element of each row if it's not a valid segment end.
    # Actually, we just sum over all m where both m and m+1 are valid.
    
    # Segment length
    d_alpha = A_next - A_safe
    
    # Midpoint alpha
    alpha_mid = (A_next + A_safe) / 2.0
    
    # Ray position at midpoint
    # R(alpha) = s + alpha * (P - s)
    # We need to broadcast s and ray_dir
    # s is (3,), ray_dir is (H, W, 3)
    # alpha_mid is (H, W, N)
    
    # Position = s + alpha_mid * ray_dir
    # s needs to be broadcast to (H, W, N, 3)
    s_expanded = s.unsqueeze(0).unsqueeze(0).unsqueeze(0) # (1, 1, 1, 3)
    ray_dir_expanded = ray_dir.unsqueeze(-1) # (H, W, 1, 3)
    
    pos = s_expanded + alpha_mid.unsqueeze(-1) * ray_dir_expanded # (H, W, N, 3)
    
    # Sample Volume V at pos
    # V is (nx, ny, nz). pos is continuous.
    # We need trilinear interpolation.
    # [G] Using torch.nn.functional.grid_sample or custom interpolation.
    # grid_sample expects normalized coordinates [-1, 1].
    # Convert pos to voxel indices.
    # x_vox = (pos_x - bx) / dX
    # But pos is in world coords.
    # Voxel coords:
    vx = (pos[..., 0] - bx) / dX
    vy = (pos[..., 1] - by) / dY
    vz = (pos[..., 2] - bz) / dZ
    
    # Normalize to [-1, 1] for grid_sample
    # Assuming volume is centered or defined from 0 to nx*dX etc.
    # grid_sample expects input size (N, C, H, W) or (N, C, D, H, W)
    # V is (nx, ny, nz). We treat it as (1, 1, nx, ny, nz).
    
    # Normalize coordinates
    # vx_norm = 2 * vx / nx - 1
    # This assumes volume starts at 0.
    # If volume is centered, adjust accordingly.
    # Source doesn't specify centering. Assume 0-indexed.
    
    vx_norm = 2 * vx / nx - 1
    vy_norm = 2 * vy / ny - 1
    vz_norm = 2 * vz / nz - 1
    
    # grid_sample input shape: (N, C, D, H, W)
    # We need to permute V to (1, 1, nz, ny, nx) for grid_sample?
    # grid_sample expects (N, C, H, W) for 2D or (N, C, D, H, W) for 3D.
    # Coordinates should be (N, H, W, D) or (N, H, W, D, H, W)?
    # For 3D grid_sample, grid shape is (N, H, W, D, 3).
    
    # This is getting very implementation-specific.
    # Let's assume a function `sample_volume(V, vx, vy, vz)` exists.
    
    V_sample = sample_volume(V, vx, vy, vz) # (H, W, N)
    
    # Weighted sum
    # Intensity = ||p-s|| * sum( (alpha_{m+1} - alpha_m) * V_sample )
    
    # Mask out invalid segments
    valid_segment = valid_mask & torch.roll(valid_mask, shifts=1, dims=-1)
    # The first element of valid_mask corresponds to alpha_0.
    # The segment is between alpha_0 and alpha_1.
    # So valid_segment should be true if both m and m+1 are valid.
    # We shifted valid_mask to align with A_next.
    # So valid_segment[m] is true if A_sorted[m] and A_sorted[m+1] are valid.
    
    d_alpha_valid = torch.where(valid_segment, d_alpha, 0.0)
    V_sample_valid = torch.where(valid_segment.unsqueeze(-1), V_sample, 0.0)
    
    intensity = ray_len * torch.sum(d_alpha_valid * V_sample_valid, dim=-1)
    
    return intensity
```

**Edge Cases and Pitfalls:**
*   **Memory Usage:** The method requires storing the full intersection tensor $\mathbf{A}$ of size $H \times W \times (n_x + n_y + n_z)$, leading to high memory requirements, especially for large volumes or high-resolution DRRs [S1]. This impacts performance for small image sizes compared to VGSJ [S1].
*   **Division by Zero:** If a ray is parallel to an axis (e.g., $P_x = s_x$), the denominator in the intersection calculation becomes zero. A guard value (e.g., $10^{-9}$) should be used [G].
*   **Degenerate Input:** Rays that do not intersect the volume should be handled by the filtering step, resulting in zero intensity [S1].

## Operating characteristics
*   **Time Complexity:** The method performs vectorized tensor operations, which are highly parallelized on the GPU. The dominant costs are the computation of intersections, sorting, and volume sampling.
*   **Memory Complexity:** $O(H \cdot W \cdot (n_x + n_y + n_z))$ due to the storage of the intersection tensor $\mathbf{A}$ [S1].
*   **Performance:** On an NVIDIA GeForce RTX 2080 Ti, VGS generates a $200 \times 200$ DRR in $72.7 \pm 10 \mu s$ [S1]. For $H=100$, time is $17.6 \pm 0.05 ms$; for $H=500$, time is $453 \pm 41.2 ms$ [S1].
*   **Accuracy:** Average RMSE of $(8.3 \pm 1.9) \times 10^{-4}$ compared to CPU Plastimatch (CP) for normalized images [S1]. Gradient accuracy is within $0.05 \pm 0.01$ of Forward Finite Differences (FFD) [S1].
*   **Gradient Efficiency:** Gradient generation takes $35.1 ms \pm 73.3 \mu s$, which is an order of magnitude faster than FFD ($400.5 ms \pm 821.4 \mu s$) [S1].
*   **Comparison:** VGS is faster than CPU Plastimatch and faster than Vectorized GPU Siddon-Jacobs (VGSJ) for $H < 500$. At $H=500$, VGS and VGSJ have roughly equivalent run times [S1].
*   **Strengths:** Fast, differentiable, compatible with deep learning frameworks, GPU-accelerated.
*   **Limitations:** High memory usage for large volumes or high-resolution DRRs [S1].

## Background
The method was introduced in the paper "Fast Auto-Differentiable Digitally Reconstructed Radiographs for Solving Inverse Problems in Intraoperative Imaging" [S1]. It addresses the need for differentiable DRR generators in gradient-based optimization tasks, such as slice-to-volume registration in intraoperative imaging [S1]. The method builds on Siddon's ray-tracing algorithm, reformulating it for vectorized tensor computation in PyTorch [S1].

## Sources
[S1] Fast Auto-Differentiable Digitally Reconstructed Radiographs for Solving Inverse Problems in Intraoperative Imaging., https://doi.org/10.1007/978-3-031-23179-7_1, fetched 2026-10-25
