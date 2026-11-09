# Back-projection

Also called: Adjoint Backprojection, Adjoint Projection, Cone Beam Adjoint, FDK Backprojection (CPU/PyTorch Path), Radon backprojection (parallel)

## Overview
Back-projection is the adjoint (transpose) of the forward projection operator $\mathcal{P}$, used to map 2D projection data (sinograms) or projection-space gradients back into a 3D volume representation [S1, unofficial][S2, unofficial][S3, unofficial][S4, unofficial]. It solves the inverse problem of tomographic reconstruction by redistributing measured ray integrals along their corresponding geometric paths into the voxel grid [S5, unofficial]. Unlike forward projection, which integrates density along rays to produce a 2D image, back-projection scatters 2D values into 3D space, effectively blurring the data if unfiltered [S5, unofficial]. It is the core differentiable operator required for iterative optimization and gradient-based reconstruction, ensuring stable convergence when paired with forward projection [S6, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sinogram/projection tensor $p$ of shape $[B, P, H, W]$ (spatial) or $[B, P, H, W/2+1]$ (Fourier-space) [S2, unofficial][S3, unofficial], and geometric parameters defining the ray paths (source positions, detector orientations, or ray anchors/directions) [S4, unofficial].
*   **Output:** A 3D volume tensor $x$ of shape $[B, D, H_{rec}, W_{rec}]$ or $[B, D, H_{rec}, W_{rec}/2+1]$ (Fourier-space) [S2, unofficial][S3, unofficial].

**Data Structures:**
*   **Ray Specification:** Defined by ray anchors $\mathbf{t} \in \mathbb{R}^D$ and directions $\mathbf{n} \in \mathbb{R}^D$ [S4, unofficial].
*   **Volume Grid:** Defined by origin $\mathbf{x}_0$, spacing $\mathbf{\Delta}$, and dimensions $\mathbf{Q}$ (knot_spec) [S4, unofficial].
*   **Interpolation State:** For GPU implementations, pending voxel values and indices are stored in registers or local memory to minimize atomic contention [S3, unofficial].

**Numbered Steps:**

1.  **Coordinate Transformation:** For each projection pixel coordinate $\mathbf{k}_{proj} = (k_r, k_c)$, compute the corresponding 3D coordinate in the volume space.
    *   In Fourier-space methods, extend to 3D: $\mathbf{k}_{3D} = (k_c, k_r, 0)$ [S2, unofficial].
    *   In spatial methods, compute the ray path from source to detector element [S3, unofficial].
    *   Apply inverse rotation/translation matrices to map detector coordinates to volume coordinates: $\mathbf{k}_{rec} = \mathbf{R}^{-1} \mathbf{k}_{3D}$ [S2, unofficial].

2.  **Interpolation Kernel Selection:** Determine the interpolation order $order \in \{0, 1, 2\}$ [S4, unofficial].
    *   $order=0$: Voxel-based (nearest neighbor).
    *   $order=1$: Trilinear (3D) or bilinear (2D).
    *   $order=2$: Higher-order box splines or tricubic (Catmull-Rom) [S2, unofficial][S4, unofficial].

3.  **Scatter Accumulation:** Distribute the projection value $p$ into the volume.
    *   **Spatial Domain:** For each ray, iterate through voxels intersected by the ray. For each voxel, compute the interpolation weights based on the fractional distance of the ray within the voxel. Accumulate the weighted projection value into the voxel using atomic addition [S3, unofficial].
        *   Formula: $x_{\mathbf{q}} \leftarrow x_{\mathbf{q}} + w \cdot p$ [S3, unofficial].
    *   **Fourier Domain:** Accumulate the projection value into the 3D Fourier volume at the transformed coordinate $\mathbf{k}_{rec}$ using 3D interpolation kernels in transpose mode [S2, unofficial].
        *   For trilinear, distribute to the $2 \times 2 \times 2$ neighborhood [S2, unofficial].
        *   For tricubic, distribute to the $4 \times 4 \times 4$ neighborhood [S2, unofficial].

4.  **Gradient Computation (Differentiable Back-projection):**
    *   To compute $\frac{\partial \mathcal{L}}{\partial x}$, pass the residual gradient $\frac{\partial \mathcal{L}}{\partial \hat{p}}$ through the back-projection operator [G].
    *   To compute gradients w.r.t. geometry (source/detector positions), apply the chain rule through the interpolation derivatives, scattering gradients back to the camera parameters [S2, unofficial][S8258, unofficial].
    *   Shift gradients are computed using phase modulation derivatives: $\frac{\partial P}{\partial \mathbf{s}} = -i2\pi\mathbf{k}_{proj}P(\mathbf{k}_{proj})$ [S2, unofficial].

**Pseudocode (Spatial Cone-Beam Adjoint):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cone_adjoint(projections, sources, detectors, box, n_samples, volume_shape):
    """
    Computes the adjoint of the cone-beam forward projection.
    
    Args:
        projections: Tensor [B, 1, views, rows, cols]
        sources: Tensor [views, 3]
        detectors: Tensor [views, rows, cols, 3]
        box: Tensor [6] (min_x, max_x, min_y, max_y, min_z, max_z)
        n_samples: int
        volume_shape: tuple (Z, Y, X)
        
    Returns:
        volume: Tensor [B, 1, Z, Y, X]
    """
    batch, _, views, rows, cols = projections.shape
    volume = torch.zeros(batch, 1, *volume_shape, device=projections.device)
    
    # Precompute voxel grid
    z, y, x = volume_shape
    zz, yy, xx = torch.meshgrid(torch.arange(z), torch.arange(y), torch.arange(x), indexing='ij')
    voxels = torch.stack([xx, yy, zz], dim=-1).float() # [Z, Y, X, 3]
    
    for view in range(views):
        src = sources[view] # [3]
        det_plane = detectors[view] # [rows, cols, 3]
        proj_view = projections[:, 0, view] # [B, rows, cols]
        
        # Vectorized ray casting for all voxels
        # Ray direction: det - src
        rays_d = det_plane - src # [rows, cols, 3]
        rays_o = src.expand_as(rays_d) # [rows, cols, 3]
        
        # Normalize directions
        rays_d_norm = rays_d / rays_d.norm(dim=-1, keepdim=True)
        
        # Compute ray-box intersection to determine active rays
        # Simplified: assume all rays are valid for this example
        # In practice, use ray_bbox_intersect to get t_min, t_max
        
        # Sample along rays
        # For each voxel, find the intersection point with the ray passing through the detector pixel
        # This is computationally expensive in pure Python; typically done in CUDA
        
        # Pseudocode for a single ray (detector pixel r, c)
        for r in range(rows):
            for c in range(cols):
                ray_o = rays_o[r, c]
                ray_d = rays_d_norm[r, c]
                value = proj_view[:, r, c] # [B]
                
                # Determine ray path through volume
                # Intersect ray with box
                t_min, t_max = ray_box_intersect(ray_o, ray_d, box)
                
                if t_max <= t_min:
                    continue
                    
                # Sample points along the ray
                t_vals = torch.linspace(t_min, t_max, n_samples)
                points = ray_o + t_vals.unsqueeze(-1) * ray_d # [n_samples, 3]
                
                # Trilinear interpolation into volume
                # Map points to voxel coordinates
                # Accumulate weighted values
                # This step requires a differentiable trilinear interpolation function
                # volume += scatter_add(points, value, weights)
                
                pass # Implementation detail omitted for brevity; see GPU kernels [S3, unofficial]

    return volume
```

**Parameters:**
*   `det_count`: Number of detector pixels. For exact reconstruction of pixels outside the inscribed circle in a 512x512 image, `det_count = ceil(sqrt(2) * 512)` [S1, unofficial].
*   `angles`: Array of sampling angles. For parallel beam, equally spaced in $[0, \pi)$ [S1, unofficial].
*   `order`: Interpolation order (0, 1, 2) [S4, unofficial].
*   `n_samples`: Number of samples per ray [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Atomic Contention:** Using `atomicAdd` for scatter accumulation is a major bottleneck on GPUs. Keeping pending voxels in registers minimizes atomics [S3, unofficial].
*   **Unfiltered Blurring:** Unfiltered back-projection creates a blurred image due to low-frequency noise accumulation ($1/r^2$ blur) [S5, unofficial].
*   **Boundary Conditions:** Volumes often require a 1-voxel zero-padding border to handle interpolation at edges without bounds checks [S3, unofficial].
*   **Precision:** Relative error compared to reference implementations (e.g., Astra Toolbox) is typically $< 10^{-4}$ for forward and $< 10^{-5}$ for back-projection [S1, unofficial].

## Subtopics

### [Forward Projection](4740-forward-projection.md)
Generates 2D projections from a 3D volume by sampling central slices in Fourier space, implementing the Central Slice Theorem. It is the mathematical adjoint to back-projection.

### [GPU Backprojection with Global Memory Buffer](4991-gpu-backprojection-with-global-memory-buffer.md)
Maps projection weights to volume expansion coefficients using a single kernel launch per angle with a 1D NDRange, avoiding complex local memory tiling.

### [GPU Backprojection using OpenCL Buffers](5110-gpu-backprojection-using-opencl-buffers.md)
Performs filtered backprojection using OpenCL global memory buffers, explicitly including the ramp filtering step in the GPU path.

### [GPU Backprojection using OpenCL Image Textures](5111-gpu-backprojection-using-opencl-image-textures.md)
Accelerates backprojection by leveraging OpenCL image objects for hardware-accelerated bilinear interpolation of pre-filtered data.

### [Fourier-space Projection Operations](6356-fourier-space-projection-operations.md)
Computes 2D/3D back-projections exclusively in the Fourier domain using RFFT format, achieving 1–2 orders of magnitude speedup over spatial-domain libraries.

### [Gradient w.r.t Source/Detector Geometry](8258-gradient-w-r-t-source-detector-geometry.md)
Computes gradients of the projection error with respect to camera parameters by scattering gradients back to the source/detector positions.

### [Explicit Radon Matrix Construction](9109-explicit-radon-matrix-construction.md)
Materializes the full forward projection matrix for parallel-beam CT, enabling direct algebraic solutions and explicit adjoint verification.

## Operating characteristics
*   **Time Complexity:** $O(N_v \cdot N_r \cdot N_s)$, where $N_v$ is the number of views, $N_r$ is the number of rays per view, and $N_s$ is the number of samples per ray [S3, unofficial]. In Fourier space, complexity is dominated by FFTs: $O(N^3 \log N)$ [S2, unofficial].
*   **Memory Complexity:** $O(N_v \cdot N_r)$ for storing projection data and $O(N_x \cdot N_y \cdot N_z)$ for the volume buffer [S3, unofficial]. Explicit Radon Matrix Construction requires $O(N_v \cdot N_r \cdot N_x \cdot N_y \cdot N_z)$, which is prohibitive for large volumes [S9109, unofficial].
*   **Performance:** TorchRadon is up to 125× faster than Astra Toolbox for parallel/fan-beam projections due to GPU batch operations [S1, unofficial]. Fourier-space methods show 1–2 orders of magnitude improvement over `torch-fourier-slice` [S6356, unofficial].
*   **Strengths:** Differentiable, stable for iterative optimization, efficient on GPUs [S1, unofficial][S6, unofficial].
*   **Limitations:** Unfiltered BP is blurry [S5, unofficial]. FDK yields worse results than iterative methods (e.g., SART) with limited projections [S7, unofficial]. Atomic contention in spatial back-projection can limit throughput [S3, unofficial].

## Background
Back-projection is the inverse of the Radon transform, forming the basis of analytical tomographic reconstruction methods like Filtered Back-Projection (FBP) [S5, unofficial][S7, unofficial]. It was historically implemented on CPUs and later optimized for GPUs using CUDA and OpenCL [S1, unofficial][S5110, unofficial]. Recent developments focus on making back-projection fully differentiable for integration into deep learning pipelines, enabling end-to-end optimization of volume representations and camera parameters [S1, unofficial][S2, unofficial][S4, unofficial].

## Sources
[S1] TorchRadon: Fast Differentiable Routines for Computed Tomography, https://doi.org/10.48550/arxiv.2009.14788, fetched 2026-10-26 (unofficial)
[S2] torch-projectors: A High-Performance Differentiable Projection Library for PyTorch, https://doi.org/10.64898/2026.03.10.710885, fetched 2026-10-26 (unofficial)
[S3] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
[S4] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
[S5] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
[S6] PDF Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
[S7] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
