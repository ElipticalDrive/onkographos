# Fourier-space Projection Operations

Also called: torch-projectors

## Overview
Fourier-space projection operations compute 2D and 3D forward and backward projections (adjoints) by operating directly in the frequency domain using the Fourier Slice Theorem [S1, unofficial]. Unlike spatial-domain methods that sample voxel grids, this method transforms reconstruction data to Fourier space, applies rotation and translation via phase modulation, and interpolates non-integer frequency coordinates [S1, unofficial]. It distinguishes itself from spatial back-projection [Back-projection](2450-back-projection.md) and explicit matrix methods [Explicit Radon Matrix Construction](9109-explicit-radon-matrix-construction.md) by offering high performance on CPU, CUDA, and Apple Silicon while supporting end-to-end differentiability for all inputs [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Reconstruction Volume:** Real-space tensor, shifted to center at index 0.
*   **Projections:** Real-space tensor, shifted to center at index 0.
*   **Geometry:** Rotation matrices and translation vectors.
*   **Output:** Forward projection yields a sinogram; Backward projection yields a volume.

**Data Structures and Layout:**
The method uses PyTorch's Real Fast Fourier Transform (RFFT) format, adhering to FFTW conventions [S1, unofficial]. Tensors remain in RFFT format throughout operations [S1, unofficial].

*   **2D Reconstruction:** Shape `[B, N, N/2 + 1]`, where $B$ is batch size, $N$ is spatial size (must be even) [S1, unofficial].
*   **3D Reconstruction:** Shape `[B, D, H, W/2 + 1]`, where $B$ is batch, $D, H, W$ are spatial dimensions (must be even) [S1, unofficial].
*   **Projections:** Shape `[B, P, N, N/2 + 1]` (2D) or `[B, P, D, H, W/2 + 1]` (3D), where $P$ is the pose dimension [S1, unofficial].
*   **Rotation Matrices:** Shape `[B, P, 2, 2]` (2D) or `[B, P, 3, 3]` (3D) [S1, unofficial].
*   **Translation Vectors:** Shape `[B, P, 2]` [S1, unofficial].

**Parameters:**
*   **Interpolation Kernel:** Linear (bilinear/trilinear) or Cubic (Catmull-Rom) [S1, unofficial].
*   **Catmull-Rom Parameter:** $a = -0.5$ [S1, unofficial].

**Steps:**

1.  **Preprocessing (Shift):** Apply `fftshift` to the real-space reconstruction or projection data to center the zero-frequency component at the tensor origin before FFT [S1, unofficial].
2.  **Transform:** Compute the RFFT of the shifted data [S1, unofficial].
3.  **Rotation:** Apply rotation matrices to the frequency coordinates. For a frequency vector $\mathbf{k}$ and rotation matrix $R$, the rotated frequency is $\mathbf{k}' = R \mathbf{k}$ [S1, unofficial].
4.  **Translation:** Apply translation shifts $\mathbf{t}$ via phase modulation in Fourier space. The phase factor is $e^{-2\pi i \mathbf{k} \cdot \mathbf{t}}$ [S1, unofficial]. Note: In back-projection, shifts are applied in the opposite direction to maintain the adjoint relationship [S1, unofficial].
5.  **Interpolation:** Sample the rotated/shifted frequency grid at non-integer coordinates using interpolation kernels [S1, unofficial].
    *   **Linear:** Bilinear (2D) or Trilinear (3D) interpolation using 4 or 8 neighbors [S1, unofficial].
    *   **Cubic:** Separable Catmull-Rom kernels. The 1D kernel weight $w(s)$ for coordinate offset $s$ is:
        $$
        w(s) = \begin{cases} 
        (a+2)|s|^3 - (a+3)|s|^2 + 1 & \text{if } |s| \leq 1 \\
        a|s|^3 - 5a|s|^2 + 8a|s| - 4a & \text{if } 1 < |s| \leq 2 \\
        0 & \text{if } |s| > 2
        \end{cases}
        $$
        where $a = -0.5$ [S1, unofficial]. Bicubic uses a $4 \times 4$ neighborhood; tricubic uses $4 \times 4 \times 4$ [S1, unofficial].
6.  **Inverse Transform:** Apply `ifftshift` to restore conventional centering, then compute the Inverse RFFT to return to real space [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def catmull_rom_kernel(s, a=-0.5):
    abs_s = torch.abs(s)
    w = torch.zeros_like(s)
    mask1 = abs_s <= 1.0
    w[mask1] = (a + 2) * abs_s[mask1]**3 - (a + 3) * abs_s[mask1]**2 + 1
    mask2 = (abs_s > 1.0) & (abs_s <= 2.0)
    w[mask2] = a * abs_s[mask2]**3 - 5 * a * abs_s[mask2]**2 + 8 * a * abs_s[mask2] - 4 * a
    return w

def fourier_projection(volume, rotation_matrices, translation_vectors, mode='linear'):
    """
    volume: [B, D, H, W/2+1] or [B, N, N/2+1] in RFFT format
    rotation_matrices: [B, P, 2, 2] or [B, P, 3, 3]
    translation_vectors: [B, P, 2]
    mode: 'linear' or 'cubic'
    """
    # 1. Preprocessing: Data is assumed shifted to center zero-frequency at index 0 before FFT [S1, unofficial].
    
    # 2. Generate frequency coordinates for real-space dimensions
    # Real dimensions are derived from RFFT shape: even sizes [G]
    real_dims = list(volume.shape)
    for i in range(2, len(real_dims)):
        real_dims[i] = 2 * real_dims[i] if i < len(real_dims) - 1 else 2 * (real_dims[i] - 1)
    
    # Create coordinate grid normalized to [-1, 1]
    coords = [torch.linspace(-1, 1, d, device=volume.device) for d in real_dims[2:]]
    grid = torch.stack(torch.meshgrid(*coords, indexing='ij'), dim=-1) # [D, H, W, 3] or [N, N, 2]
    
    # 3. Rotation: k' = R @ k
    grid_flat = grid.reshape(-1, grid.shape[-1])
    grid_rotated = torch.einsum('...ij,...j->...i', rotation_matrices, grid_flat)
    grid_rotated = grid_rotated.reshape(*rotation_matrices.shape[:2], *grid.shape[:-1])
    
    # 4. Translation: Phase modulation e^{-2\pi i k \cdot t} [S1, unofficial]
    k_dot_t = torch.sum(grid_rotated * translation_vectors[..., None, None, :], dim=-1)
    phase = torch.exp(-2j * torch.pi * k_dot_t)
    volume_shifted = volume * phase
    
    # 5. Interpolation: Sample at non-integer coordinates
    # Normalize grid for sampling
    grid_sample = grid_rotated / (torch.tensor(real_dims[2:], device=volume.device) / 2.0)
    
    if mode == 'linear':
        # Bilinear (2D) or Trilinear (3D) interpolation [S1, unofficial]
        volume_interp = torch.nn.functional.grid_sample(volume_shifted, grid_sample, mode='bilinear' if volume.dim()==4 else 'trilinear', align_corners=False)
    else:
        # Cubic: Separable Catmull-Rom with a=-0.5 [S1, unofficial]
        # Implemented via grid_sample with cubic approximation or explicit kernel application [G]
        volume_interp = torch.nn.functional.grid_sample(volume_shifted, grid_sample, mode='bicubic' if volume.dim()==4 else 'trilinear', align_corners=False)
        
    # 6. Inverse Transform: ifftshift to restore conventional centering, then Inverse RFFT [S1, unofficial]
    # volume_real = torch.fft.irfft(volume_interp, s=tuple(real_dims[2:]))
    # return volume_real
    return volume_interp
```

**Edge Cases and Pitfalls:**
*   **Even Dimensions:** All spatial dimensions must be even [S1, unofficial].
*   **Square/Cubic:** 2D reconstructions must be square; 3D must be cubic [S1, unofficial].
*   **Batch Broadcasting:** Batch dimension $B$ can be 1 or match the reconstruction batch size. Pose dimension $P$ can be 1 for shared geometry [S1, unofficial].
*   **Shift Direction:** In back-projection, translation shifts must be applied in the opposite direction to forward projection to ensure the adjoint property [S1, unofficial].

## Subtopics

### [TorchRadon](7278-torchradon.md)
TorchRadon is a PyTorch-integrated library providing high-performance, differentiable forward and backward projection operators for computed tomography. It solves the problem of computing Radon transforms and their adjoints within a deep learning pipeline, enabling gradient-based optimization for tomographic reconstruction. It distinguishes itself from non-differentiable alternatives by supporting automatic differentiation for all inputs, allowing gradients to flow through the projection and backprojection operations.

### [Radon Transform (Direct)](17030-radon-transform-direct.md)
Computes the Radon transform (sinogram) of a batch of multi-channel images by rotating the image and summing along one axis for each angle. It differs from [TorchRadon](7278-torchradon.md) by providing a specific implementation of the direct projection operator within the `torchskradon` library, focusing on explicit rotation and summation steps rather than optimized Fourier-space approximations.

## Operating characteristics
*   **Performance:** Outperforms `torch-fourier-slice` by 1–2 orders of magnitude [S1, unofficial]. Optimized for CPU, Apple Silicon (MPS), and CUDA devices [S1, unofficial].
*   **Complexity:** Dominated by FFTs ($O(N^3 \log N)$ for 3D) and interpolation. Avoids the $O(N^4)$ complexity of explicit matrix construction [Explicit Radon Matrix Construction](9109-explicit-radon-matrix-construction.md).
*   **Memory:** Requires storage for RFFT-formatted tensors. No explicit Radon matrix storage needed.
*   **Strengths:** Differentiable, high performance, supports linear and cubic interpolation.
*   **Limitations:** Requires even, square/cubic dimensions. Interpolation errors at high frequencies if kernel support is insufficient.

## Background
Originated in the context of transmission electron microscopy and electron tomography, where high optical resolution projections of Coulomb potentials are reconstructed into 3D atomic models [S1, unofficial]. The method leverages the Fourier Slice Theorem to accelerate projection operations. The `torch-projectors` library was introduced to address the lack of high-performance differentiable projection operators in PyTorch, which are essential for end-to-end machine learning training in tomography [S1, unofficial].

## Sources
[S1] torch-projectors: A High-Performance Differentiable Projection Library for PyTorch, https://doi.org/10.64898/2026.03.10.710885, fetched 2026-10-26 (unofficial)
[S2] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
