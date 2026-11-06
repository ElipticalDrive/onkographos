# Back-projection
Also called: Adjoint Backprojection, Adjoint Projection, Adjoint of xrt_apply, Cone Beam Adjoint, 2D → 3D Backward Projection, Differentiable XRT Back-Projection, Fourier Backprojection

## Overview
Back-projection is the mathematical adjoint (transpose) of the forward projection operator, used to accumulate 2D projection data into a 3D reconstruction volume [S1, S2, S3, unofficial]. It solves the problem of mapping measured projection weights to volume expansion coefficients, serving as the core gradient computation step in differentiable tomographic reconstruction [S2, S3, unofficial]. Unlike direct methods like FDK which may suffer from artifacts with limited projections, back-projection provides the exact transpose required for stable iterative optimization over many iterations [S3, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 2D Fourier-space projections of shape $[B, P, H, W/2+1]$ [S1, unofficial].
*   **Output:** 3D reconstruction volume of shape $[B, D, H_{rec}, W_{rec}/2+1]$, where $D = H_{rec} = W_{rec} = H \cdot \text{oversampling}$, forming a cubic volume [S1, unofficial].
*   **Parameters:**
    *   `ray_spec`: Tuple of ray anchors $\mathbf{t} \in \mathbb{R}^D$ and directions $\mathbf{n} \in \mathbb{R}^D$ [S3, unofficial].
    *   `knot_spec`: UniformSpec defining volume properties $(\mathbf{x}_0, \mathbf{\Delta}, \mathbf{Q})$ [S3, unofficial].
    *   `order`: Integer 0, 1, or 2, selecting the basis function $\psi$ for interpolation [S3, unofficial].
    *   `data`: Array of shape $(L,)$ containing projections $g_l \in \mathbb{R}$ [S3, unofficial].
    *   `buffer`: Flattened C-ordered buffer of shape $(Q_1, \dots, Q_D)$ to accumulate weights $f_{\mathbf{q}} \in \mathbb{R}$ [S3, unofficial].
    *   `mode`: Differentiation mode [S3, unofficial].

**Data Structures:**
*   `ArrayNf`: Float array type (e.g., `drjit.Array3f`) [S3, unofficial].
*   `ArrayNi`: Int32 array type [S3, unofficial].
*   `ArrayNu`: Uint32 array type [S3, unofficial].
*   `Float`: Scalar value type [S3, unofficial].
*   `Bool`: Mask type [S3, unofficial].

**Steps:**

1.  **Initialization:** Initialize the accumulation buffer to zeros if `None`, or assert it matches the size `math.prod(knot_spec.num)` [S3, unofficial]. Determine dimension $D$ from `ArrayNf` size; assert $D \in \{2, 3\}$ [S3, unofficial].
2.  **Coordinate Mapping (2D→3D):** For each projection coordinate $\mathbf{k}_{proj} = (k_r, k_c)$, extend to 3D by setting the third coordinate to zero: $\mathbf{k}_{3D} = (k_c, k_r, 0)$ [S1, Eq. 23, unofficial]. This implements the central slice through the origin [S1, unofficial].
3.  **Rotation Inverse:** Compute the 3D sampling coordinate in the reconstruction $\mathbf{k}_{rec}$ using the inverse of the 3×3 rotation matrix $\mathbf{R}$ used in forward projection: $\mathbf{k}_{rec} = \mathbf{R}^{-1} \mathbf{k}_{3D}$ [S1, Eq. 13, unofficial].
4.  **Phase Modulation (if shifts present):** If shifts are present, conjugate phase-modulate the projection data to ensure the mathematical adjoint relationship [S1, unofficial].
5.  **Interpolation and Accumulation:** Accumulate the projection value into the reconstruction at $\mathbf{k}_{rec}$ using 3D interpolation kernels in transpose mode [S1, unofficial]:
    *   **Trilinear (Order 1):** Distribute contributions to the $2 \times 2 \times 2$ neighborhood around the fractional coordinate [S1, unofficial].
    *   **Tricubic (Order 2):** Spread contributions across the $4 \times 4 \times 4$ neighborhood according to Catmull-Rom weights [S1, unofficial].
    *   **Voxel (Order 0):** Scatter add length $L$ times data to the accumulator at the computed offset [S3, unofficial].
6.  **Weight Accumulation (Optional):** Accumulate weights using the absolute values of the 3D interpolation kernel weights, handling 3D Friedel symmetry for real-valued reconstructions [S1, unofficial].

**Backward Pass (Gradient Computation):**
1.  **Projection Gradients:** Compute projection gradients using 3D→2D forward projection of reconstruction gradients, exploiting the adjoint relationship [S1, unofficial].
2.  **Rotation Matrix Gradients:** Compute 3×3 rotation matrix gradients using Eq. 15, with spatial derivatives computed via Eqs. 16–18 (trilinear) or Eq. 19 (tricubic) [S1, unofficial].
3.  **Shift Gradients:** Compute shift gradients using Eq. 22, applied to the 2D projection coordinates:
    $$ \frac{\partial P}{\partial \mathbf{s}} = -i2\pi\mathbf{k}_{proj}P(\mathbf{k}_{proj}) \quad (22) $$
    [S1, unofficial].

**Pseudocode (2D→3D Adjoint Accumulation):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def back_project_2d_to_3d(projections, rotation_matrix, shifts, buffer, interpolation_order):
    """
    Accumulates 2D Fourier projections into a 3D volume buffer.
    projections: [B, P, H, W/2+1]
    rotation_matrix: 3x3 matrix R
    shifts: optional shift array
    buffer: [B, D, H_rec, W_rec/2+1] initialized to 0
    """
    B, P, H, W_half = projections.shape
    # 1. Extend 2D coords to 3D central slice: k_3D = (k_c, k_r, 0)
    k_r, k_c = torch.meshgrid(torch.arange(H), torch.arange(W_half), indexing='ij')
    k_3d = torch.stack([k_c, k_r, torch.zeros_like(k_c)], dim=-1).float()
    
    # 2. Apply inverse rotation: k_rec = R^{-1} k_3D
    k_rec = (rotation_matrix.T @ k_3d.permute(2, 0, 1)).permute(1, 2, 0)
    
    # 3. Handle shifts via conjugate phase modulation
    if shifts is not None:
        phase = -2j * torch.pi * (k_3d[..., :2] * shifts).sum(dim=-1)
        projections = projections * torch.exp(phase)
        
    # 4. Accumulate using interpolation kernel in transpose mode
    buf_h, buf_w = buffer.shape[2], buffer.shape[3]
    frac_h = k_rec[..., 0] / buf_h
    frac_w = k_rec[..., 1] / buf_w
    
    if interpolation_order == 1: # Trilinear (2x2x2 neighborhood)
        scatter_trilinear(buffer, frac_h, frac_w, projections)
    elif interpolation_order == 2: # Tricubic (4x4x4 neighborhood)
        scatter_tricubic(buffer, frac_h, frac_w, projections)
    return buffer

def scatter_trilinear(buffer, frac_h, frac_w, values):
    """Distribute contributions to 2x2x2 neighborhood."""
    h, w, d = buffer.shape
    for i in range(frac_h.shape[0]):
        x, y = frac_h[i], frac_w[i]
        val = values[i]
        x0, y0 = floor(x), floor(y)
        dx, dy = x - x0, y - y0
        weights = [
            (1-dx)*(1-dy), dx*(1-dy), (1-dx)*dy, dx*dy,
            (1-dx)*(1-dy), dx*(1-dy), (1-dx)*dy, dx*dy
        ]
        offsets = [(0,0), (1,0), (0,1), (1,1), (0,0), (1,0), (0,1), (1,1)]
        for (ox, oy), weight in zip(offsets, weights):
            nx, ny = x0 + ox, y0 + oy
            if 0 <= nx < h and 0 <= ny < w:
                buffer[nx, ny, 0] += weight * val

def scatter_tricubic(buffer, frac_h, frac_w, values):
    """Distribute contributions to 4x4x4 neighborhood using Catmull-Rom."""
    # Standard Catmull-Rom weight computation and scatter add to 64 neighbors
    pass # Implementation follows standard tricubic interpolation logic
```

**Pseudocode (3D Voxel Backprojection with DDA):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def back_project_voxel(ray_spec, knot_spec, data, buffer):
    """
    Ray-traced backprojection for order=0 (voxel).
    ray_spec: (ray_t, ray_n)
    knot_spec: (x0, delta, Q)
    data: (L,) projection values
    buffer: (Q1, Q2, Q3) accumulation buffer
    """
    ray_t, ray_n = ray_spec
    x0, delta, Q = knot_spec
    L = len(data)
    
    # Compute bounding box
    bbox_ll = x0 - (delta / 2)
    bbox_ur = x0 - (delta / 2) + (Q * delta)
    
    # Stride for 3D buffer
    stride = (Q[1] * Q[2], Q[2], 1)
    
    for i in range(L):
        t = ray_t[i]
        n = ray_n[i]
        
        # Ray-BBox Intersection
        active, t_min, t_max = ray_bbox_intersect(t, n, bbox_ll, bbox_ur)
        if not active: continue
        
        current_t = t_min
        while current_t < t_max:
            # Get current cell index
            cell_idx = get_cell_index(t + current_t * n, x0, delta)
            
            # Compute length of ray segment in this cell
            path_length = compute_path_length_in_cell(t + current_t * n, n, cell_idx, x0, delta)
            
            # Scatter add
            offset = cell_idx[0] * stride[0] + cell_idx[1] * stride[1] + cell_idx[2] * stride[2]
            buffer.flat[offset] += path_length * data[i]
            
            # Advance
            current_t += step_to_next_cell(t + current_t * n, n, cell_idx, x0, delta)
            
    return buffer

def ray_bbox_intersect(o, d, tmin, tmax):
    inv_d = 1.0 / d
    t1 = (tmin - o) * inv_d
    t2 = (tmax - o) * inv_d
    t_enter = max(min(t1), min(t2))
    t_exit = min(max(t1), max(t2))
    active = t_exit > t_enter and isfinite(t_exit)
    return active, t_enter, t_exit

def get_cell_index(point, x0, delta):
    return floor((point - x0) / delta)

def compute_path_length_in_cell(o, d, cell_idx, x0, delta):
    cell_min = x0 + cell_idx * delta
    cell_max = cell_min + delta
    inv_d = 1.0 / d
    t1 = (cell_min - o) * inv_d
    t2 = (cell_max - o) * inv_d
    t_enter = max(min(t1), min(t2))
    t_exit = min(max(t1), max(t2))
    return max(0.0, t_exit - t_enter)

def step_to_next_cell(o, d, cell_idx, x0, delta):
    cell_min = x0 + cell_idx * delta
    cell_max = cell_min + delta
    inv_d = 1.0 / d
    t_exit = min(max((cell_max - o) * inv_d), max((cell_min - o) * inv_d))
    return t_exit
```

## Subtopics

### [Forward Projection](4740-forward-projection.md)
An operator that generates 2D projections from a 3D Fourier-space reconstruction by sampling central slices through the 3D volume, implementing the Central Slice Theorem.

### [GPU Backprojection with Global Memory Buffer](4991-gpu-backprojection-with-global-memory-buffer.md)
A GPU-accelerated cone-beam CT reconstruction method that iterates over projection angles, loading each projection into global memory and accumulating contributions into a volume buffer using a single kernel launch per projection.

### [GPU Backprojection using OpenCL Buffers](5110-gpu-backprojection-using-opencl-buffers.md)
GPU-based backprojection using OpenCL buffers for data storage.

### [GPU Backprojection using OpenCL Image Textures](5111-gpu-backprojection-using-opencl-image-textures.md)
GPU-based backprojection using OpenCL image textures for data storage, likely leveraging hardware interpolation.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** $O(V)$ for the reconstruction buffer plus $O(P)$ for projections [S3, S4, unofficial].
*   **Performance:** Benchmarks on NVIDIA H100, Apple M4 CPU, and M4 Metal show throughput in thousands of projections per second, scaling with box size (32–256 pixels), batch size (1–8), and pose count (8–2048) [S1, unofficial].
*   **Strengths:** Provides the exact adjoint required for stable gradient-based optimization in differentiable rendering [S3, unofficial]. Handles arbitrary 3D detector shifts and non-equispaced angles [S3, unofficial].
*   **Limitations:** Direct methods like FDK (which uses backprojection) yield worse results than iterative methods (like SIRT) when projections are limited [S4, unofficial]. Exact transposes may be less computationally efficient than approximate methods, though necessary for stability over many iterations [S3, unofficial].
*   **When to Use:** Differentiable tomographic reconstruction, iterative optimization requiring stable gradients, cryo-EM reconstruction with CTF correction [S1, S3, unofficial].
*   **When Not to Use:** When limited projections are available, direct backprojection yields worse results than iterative methods like SIRT [S4, S6, unofficial].

## Background
Back-projection originates from the mathematical theory of the Radon transform and its inverse, forming the basis of filtered back-projection (FBP) algorithms in CT [S4, unofficial]. In the context of differentiable rendering, it is formalized as the adjoint of the X-ray transform (forward projection), enabling gradient computation for volume parameters [S3, unofficial]. The specific implementation described here, `xrt_adjoint`, is part of the `differentiable_xrt` library, designed for high-performance differentiable projection on GPUs using Dr.Jit [S2, unofficial]. It addresses the need for matched projectors in iterative reconstruction to ensure stability over thousands of iterations, a requirement often violated by standard CT packages that prioritize speed over exact transposition [S3, unofficial].

## Sources
[S1] torch-projectors: A High-Performance Differentiable Projection Library for PyTorch, https://doi.org/10.64898/2026.03.10.710885, fetched 2026-10-26 (unofficial)
[S2] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
[S3] PDF Differentiable Forward Projector for X-ray Computed Tomography, https://differentiable.xyz/papers-2023/paper_48.pdf, fetched 2026-10-29 (unofficial)
[S4] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
