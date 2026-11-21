# Fourier-space Projection Operations

Also called: torch-projectors

## Overview
Fourier-space projection operations compute 2D and 3D forward and backward projections (adjoints) by operating directly in the frequency domain using the Fourier Slice Theorem. Unlike spatial-domain methods that sample voxel grids, this method transforms reconstruction data to Fourier space, applies rotation and translation via phase modulation, and interpolates non-integer frequency coordinates. It distinguishes itself from spatial back-projection [Back-projection](2450-back-projection.md) and explicit matrix methods [Explicit Radon Matrix Construction](9109-explicit-radon-matrix-construction.md) by offering high performance on CPU, CUDA, and Apple Silicon while supporting end-to-end differentiability for all inputs [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Reconstruction Volume:** Real-space tensor, shifted to center at index 0.
*   **Projections:** Real-space tensor, shifted to center at index 0.
*   **Geometry:** Rotation matrices and translation vectors.
*   **Output:** Forward projection yields a sinogram; Backward projection yields a volume.

**Data Structures and Layout:**
The method uses PyTorch's Real Fast Fourier Transform (RFFT) format, adhering to FFTW conventions [S1, unofficial]. Tensors remain in RFFT format throughout operations.

*   **2D Reconstruction:** Shape `[B, N, N/2 + 1]`, where $B$ is batch size, $N$ is spatial size (must be even).
*   **3D Reconstruction:** Shape `[B, D, H, W/2 + 1]`, where $B$ is batch, $D, H, W$ are spatial dimensions (must be even).
*   **Projections:** Shape `[B, P, N, N/2 + 1]` (2D) or `[B, P, D, H, W/2 + 1]` (3D), where $P$ is the pose dimension.
*   **Rotation Matrices:** Shape `[B, P, 2, 2]` (2D) or `[B, P, 3, 3]` (3D).
*   **Translation Vectors:** Shape `[B, P, 2]` (2D) or `[B, P, 3]` (3D).

**Parameters:**
*   **Interpolation Kernel:** Linear (bilinear/trilinear) or Cubic (Catmull-Rom).
*   **Catmull-Rom Parameter:** $a = -0.5$ [S1, unofficial].

**Steps:**

1.  **Preprocessing (Shift):** Apply `fftshift` to the real-space reconstruction or projection data to center the zero-frequency component at the tensor origin.
2.  **Transform:** Compute the RFFT of the shifted data.
3.  **Rotation:** Apply rotation matrices to the frequency coordinates. For a frequency vector $\mathbf{k}$ and rotation matrix $R$, the rotated frequency is $\mathbf{k}' = R \mathbf{k}$.
4.  **Translation:** Apply translation shifts $\mathbf{t}$ via phase modulation in Fourier space. The phase factor is $e^{-2\pi i \mathbf{k} \cdot \mathbf{t}}$. Note: In back-projection, shifts are applied in the opposite direction to maintain the adjoint relationship [S1, unofficial].
5.  **Interpolation:** Sample the rotated/shifted frequency grid at non-integer coordinates using interpolation kernels.
    *   **Linear:** Bilinear (2D) or Trilinear (3D) interpolation using 4 or 8 neighbors.
    *   **Cubic:** Separable Catmull-Rom kernels. The 1D kernel weight $w(s)$ for coordinate offset $s$ is:
        $$
        w(s) = \begin{cases} 
        (a+2)|s|^3 - (a+3)|s|^2 + 1 & \text{if } |s| \leq 1 \\
        a|s|^3 - 5a|s|^2 + 8a|s| - 4a & \text{if } 1 < |s| \leq 2 \\
        0 & \text{if } |s| > 2
        \end{cases}
        $$
        where $a = -0.5$ [S1, unofficial]. Bicubic uses a $4 \times 4$ neighborhood; tricubic uses $4 \times 4 \times 4$.
6.  **Inverse Transform:** Apply `ifftshift` to restore conventional centering, then compute the Inverse RFFT to return to real space.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def fourier_forward_projection(volume, rotation_matrices, translation_vectors, interpolation='linear'):
    """
    volume: [B, D, H, W/2+1] (RFFT format, real-space shifted)
    rotation_matrices: [B, P, 3, 3]
    translation_vectors: [B, P, 3]
    """
    # 1. Shift real space to center
    # Note: torch.fft.rfft assumes standard ordering; fftshift moves zero freq to center
    # However, the library description implies input is already shifted or handled internally.
    # We assume input 'volume' is in RFFT format corresponding to shifted real space.
    
    # 2. Get frequency grid
    # D, H, W are spatial dims. W is the last dim.
    # RFFT output shape for dim L is L//2 + 1.
    # We need to generate frequency coordinates for the original real-space dimensions.
    # Let real dims be (D_real, H_real, W_real).
    # D_real = D * 2 (if D > 1), H_real = H * 2, W_real = (W-1)*2
    # This mapping is implicit in the library's internal coordinate generation.
    # For implementation from text: we assume a helper `get_frequency_grid` exists.
    
    # 3. Rotate frequencies
    # k' = R @ k
    # This requires broadcasting rotation matrices over the frequency grid.
    
    # 4. Translate frequencies (Phase modulation)
    # Phase = exp(-2 * pi * i * k' . t)
    
    # 5. Interpolate
    # Sample the rotated/shifted grid.
    
    # 6. Inverse Shift and IFFT
    # ifftshift -> irfft
    
    raise NotImplementedError("Full implementation requires internal coordinate grid generation logic specific to torch-projectors.")

def catmull_rom_kernel(s, a=-0.5):
    """
    Computes the Catmull-Rom kernel weight for offset s.
    """
    abs_s = torch.abs(s)
    w = torch.zeros_like(s)
    
    # Region 1: |s| <= 1
    mask1 = abs_s <= 1.0
    w[mask1] = (a + 2) * abs_s[mask1]**3 - (a + 3) * abs_s[mask1]**2 + 1
    
    # Region 2: 1 < |s| <= 2
    mask2 = (abs_s > 1.0) & (abs_s <= 2.0)
    w[mask2] = a * abs_s[mask2]**3 - 5 * a * abs_s[mask2]**2 + 8 * a * abs_s[mask2] - 4 * a
    
    # Region 3: |s| > 2
    # w remains 0
    
    return w
```

**Edge Cases and Pitfalls:**
*   **Even Dimensions:** All spatial dimensions must be even. If not, the RFFT shape `[N/2 + 1]` is invalid for the assumed symmetry [S1, unofficial].
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
