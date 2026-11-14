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
*   Gradients $\nabla_\eta \mathbf{I}$: Automatically computed by the framework [S1].

**Data Structures:**
*   $\mathbf{P} \in \mathbb{R}^{H \times W \times 3}$: Tensor of 3D coordinates for each pixel on the detector plane.
*   $\mathbf{A} \in \mathbb{R}^{H \times W \times (n_x + n_y + n_z)}$: Tensor containing intersection parameters $\alpha$ for all x, y, and z planes for all pixels [S1].
*   $\alpha_{\min}, \alpha_{\max} \in \mathbb{R}^{H \times W}$: Entry and exit bounds for each ray.

**Steps:**

1.  **Define Ray Geometry:**
    The ray $\mathbf{R}(\alpha)$ originates from the source $\mathbf{s}$ and passes through pixel $\mathbf{p}$ on the detector:
    $$ \mathbf{R}(\alpha) = \mathbf{s} + \alpha(\mathbf{p} - \mathbf{s}) $$
    where $\alpha=0$ at the source and $\alpha=1$ at the detector [S1].

2.  **Compute Intersection Parameters:**
    Calculate the intersection parameters $\alpha$ for all pixels with all orthogonal planes of the volume in parallel using tensor operations [S1]:
    $$ \mathbf{A} = \begin{pmatrix} b_x \\ b_y \\ b_z \end{pmatrix} + \begin{pmatrix} i \\ j \\ k \end{pmatrix} \otimes \begin{pmatrix} \Delta X \\ \Delta Y \\ \Delta Z \end{pmatrix} - \mathbf{s} \oslash (\mathbf{P} - \mathbf{s}) $$
    where $(i, j, k)$ are voxel indices, $\otimes$ is the Hadamard (element-wise) product, and $\oslash$ denotes element-wise division [S1]. In standard Siddon's, $\alpha = (x_{\text{plane}} - x_s) / (x_p - x_s)$ [G].

3.  **Determine Entry and Exit Bounds:**
    Calculate the minimum and maximum $\alpha$ values corresponding to the ray entering and exiting the volume bounds for each pixel [S1]:
    $$ \alpha_{\min} = \max \{ \min\{\alpha_x(0), \alpha_x(n_x)\}, \min\{\alpha_y(0), \alpha_y(n_y)\}, \min\{\alpha_z(0), \alpha_z(n_z)\} \} $$
    $$ \alpha_{\max} = \min \{ \max\{\alpha_x(0), \alpha_x(n_x)\}, \max\{\alpha_y(0), \alpha_y(n_y)\}, \max\{\alpha_z(0), \alpha_z(n_z)\} \} $$
    where $\alpha_x(0), \alpha_x(n_x)$ etc. are the $\alpha$ values for the planes at index 0 and $n_x$ (and similarly for y, z) [S1].

4.  **Filter and Sort Intersections:**
    Filter tensor $\mathbf{A}$ to include only values in the range $[\alpha_{\min}, \alpha_{\max}]$ [S1]. Sort each row $\mathbf{A}(h, w, \cdot)$ for $h \in \{1, \dots, H\}, w \in \{1, \dots, W\}$ to obtain the ordered sequence of plane intersections [S1].

5.  **Evaluate Intensity:**
    Compute the intensity for each pixel using the discrete summation formula [S1]:
    $$ E(R) = \|\mathbf{p} - \mathbf{s}\|_2 \sum_{m=1}^{M-1} (\alpha_{m+1} - \alpha_m) \mathbf{V} \left[ \mathbf{s} + \frac{\alpha_{m+1} + \alpha_m}{2} (\mathbf{p} - \mathbf{s}) \right] $$
    where $\alpha_m$ are the sorted intersection parameters and $M$ is the number of intersections [S1]. The term $\mathbf{V}[\dots]$ samples the volume at the midpoint of the ray segment between intersections [S1].

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

    # 2. Compute Source Position s (Cartesian)
    sx = rho * torch.sin(theta) * torch.cos(phi)
    sy = rho * torch.sin(theta) * torch.sin(phi)
    sz = rho * torch.cos(theta)
    s = torch.tensor([sx, sy, sz], device=V.device) + torch.tensor([bx, by, bz], device=V.device)

    # 3. Define Detector Plane Pixels P
    # Detector is tangent to sphere of radius rho at point -s.
    # Normal n = -s / rho.
    n = -s / rho
    # Create local detector grid at z = -rho, normal (0,0,-1)
    y_coords = torch.linspace(-W/2, W/2, W, device=V.device)
    x_coords = torch.linspace(-H/2, H/2, H, device=V.device)
    xx, yy = torch.meshgrid(x_coords, y_coords, indexing='ij')
    zz = -rho * torch.ones_like(xx)
    P_local = torch.stack([xx, yy, zz], dim=-1) # (H, W, 3)

    # Rotation to align (0,0,-1) to n
    ref = torch.tensor([0.0, 0.0, -1.0], device=V.device)
    u = torch.cross(ref, n)
    if torch.norm(u) < 1e-6:
        u = torch.tensor([1.0, 0.0, 0.0], device=V.device)
    u = u / torch.norm(u)
    v = torch.cross(n, u)

    # Rotation matrix R such that R @ [0,0,-1]^T = n
    R = torch.stack([u, v, n], dim=-1) # (3, 3)
    
    # Apply rotation gamma about x-axis in the detector's local frame
    cos_g = torch.cos(gamma)
    sin_g = torch.sin(gamma)
    R_x = torch.tensor([[1, 0, 0],
                        [0, cos_g, -sin_g],
                        [0, sin_g, cos_g]], device=V.device)
    P_local_rot = torch.einsum('ij,hwj->hwi', R_x, P_local)
    P_world = torch.einsum('ij,hwj->hwi', R, P_local_rot) - s.unsqueeze(0).unsqueeze(0)

    # 4. Compute Intersection Parameters A
    i_idx = torch.arange(nx, device=V.device).float()
    j_idx = torch.arange(ny, device=V.device).float()
    k_idx = torch.arange(nz, device=V.device).float()

    # Denominators (ray direction components)
    dx_ray = P_world[..., 0] - s[0]
    dy_ray = P_world[..., 1] - s[1]
    dz_ray = P_world[..., 2] - s[2]
    # Guard against division by zero
    dx_ray = torch.where(dx_ray.abs() < 1e-9, 1e-9 * torch.sign(dx_ray), dx_ray)
    dy_ray = torch.where(dy_ray.abs() < 1e-9, 1e-9 * torch.sign(dy_ray), dy_ray)
    dz_ray = torch.where(dz_ray.abs() < 1e-9, 1e-9 * torch.sign(dz_ray), dz_ray)

    # Alpha values for planes
    alpha_x = (i_idx * dX + bx - s[0]) / dx_ray.unsqueeze(0)
    alpha_y = (j_idx * dY + by - s[1]) / dy_ray.unsqueeze(0)
    alpha_z = (k_idx * dZ + bz - s[2]) / dz_ray.unsqueeze(0)

    # Permute to (H, W, N) and concatenate
    alpha_x = alpha_x.permute(1, 2, 0)
    alpha_y = alpha_y.permute(1, 2, 0)
    alpha_z = alpha_z.permute(1, 2, 0)
    A = torch.cat([alpha_x, alpha_y, alpha_z], dim=-1)

    # 5. Determine Bounds
    ax_0, ax_n = alpha_x[..., 0], alpha_x[..., -1]
    ay_0, ay_n = alpha_y[..., 0], alpha_y[..., -1]
    az_0, az_n = alpha_z[..., 0], alpha_z[..., -1]

    min_x, max_x = torch.min(ax_0, ax_n), torch.max(ax_0, ax_n)
    min_y, max_y = torch.min(ay_0, ay_n), torch.max(ay_0, ay_n)
    min_z, max_z = torch.min(az_0, az_n), torch.max(az_0, az_n)

    alpha_min = torch.max(torch.max(min_x, min_y), min_z)
    alpha_max = torch.min(torch.min(max_x, max_y), max_z)

    # 6. Filter and Sort
    mask = (A >= alpha_min.unsqueeze(-1)) & (A <= alpha_max.unsqueeze(-1))
    A_sort = A.clone()
    A_sort[~mask] = float('inf')
    A_sorted, _ = torch.sort(A_sort, dim=-1)

    # 7. Evaluate Intensity
    ray_dir = P_world - s # (H, W, 3)
    ray_len = torch.norm(ray_dir, dim=-1) # (H, W)

    # Handle variable number of intersections by masking
    valid = A_sorted != float('inf')
    # Shift to get next alpha
    A_next = torch.roll(A_sorted, shifts=-1, dims=-1)
    A_next[~valid] = 0.0 # Mask invalid
    # Segment validity: both current and next must be valid
    seg_valid = valid & torch.roll(valid, shifts=1, dims=-1)
    # Mask the last column which wraps around
    seg_valid[:, :, -1] = False

    d_alpha = A_next - A_sorted
    alpha_mid = (A_next + A_sorted) / 2.0

    # Sample volume at midpoints
    # Convert world coords to voxel indices
    mid_pos = s[None, None, :] + alpha_mid[:, :, :, None] * ray_dir[:, :, None, :]
    vx = (mid_pos[..., 0] - bx) / dX
    vy = (mid_pos[..., 1] - by) / dY
    vz = (mid_pos[..., 2] - bz) / dZ

    # Trilinear interpolation
    x0, x1 = torch.floor(vx).long(), torch.ceil(vx).long()
    y0, y1 = torch.floor(vy).long(), torch.ceil(vy).long()
    z0, z1 = torch.floor(vz).long(), torch.ceil(vz).long()
    dx, dy, dz = vx - x0.float(), vy - y0.float(), vz - z0.float()

    v000 = V[x0, y0, z0]
    v100 = V[x1, y0, z0]
    v010 = V[x0, y1, z0]
    v110 = V[x1, y1, z0]
    v001 = V[x0, y0, z1]
    v101 = V[x1, y0, z1]
    v011 = V[x0, y1, z1]
    v111 = V[x1, y1, z1]

    v00 = v000 + dx * (v100 - v000)
    v01 = v001 + dx * (v101 - v001)
    v10 = v010 + dx * (v110 - v010)
    v11 = v011 + dx * (v111 - v011)
    v0 = v00 + dy * (v10 - v00)
    v1 = v01 + dy * (v11 - v01)
    V_sample = v0 + dz * (v1 - v0)

    # Weighted sum
    intensity = ray_len * torch.sum(seg_valid * d_alpha * V_sample, dim=-1)
    return intensity
```

**Edge Cases and Pitfalls:**
*   **Memory Usage:** The method requires storing the full intersection tensor $\mathbf{A}$ of size $H \times W \times (n_x + n_y + n_z)$, leading to high memory requirements, especially for large volumes or high-resolution DRRs [S1]. This impacts performance for small image sizes compared to VGSJ [S1].
*   **Division by Zero:** If a ray is parallel to an axis (e.g., $P_x = s_x$), the denominator in the intersection calculation becomes zero. A guard value (e.g., $10^{-9}$) should be used [G].
*   **Degenerate Input:** Rays that do not intersect the volume should be handled by the filtering step, resulting in zero intensity [S1].

## Operating characteristics
*   **Time Complexity:** The method performs vectorized tensor operations, which are highly parallelized on the GPU [S1]. The dominant costs are the computation of intersections, sorting, and volume sampling [G].
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
