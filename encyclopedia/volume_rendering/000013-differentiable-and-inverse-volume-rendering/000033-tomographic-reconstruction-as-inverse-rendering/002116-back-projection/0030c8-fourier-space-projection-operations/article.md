# Fourier-space Projection Operations

Also called: torch-projectors

## Overview
Fourier-space projection operations compute 2D and 3D forward and backward (adjoint) projection operators for tomographic reconstruction by operating exclusively in the Fourier domain [S1, unofficial]. It distinguishes itself by maintaining data in the Real Fast Fourier Transform (RFFT) format throughout the pipeline, avoiding intermediate format conversions, and achieving performance improvements of 1–2 orders of magnitude over libraries like `torch-fourier-slice` [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Reconstruction Volume:** A real-space tensor representing the 3D or 2D object.
*   **Projection Data:** A tensor representing the 2D projections in RFFT format.
*   **Geometry:** Rotation matrices and translation vectors defining the pose of each projection.
*   **Interpolation Type:** Linear (bilinear/trilinear) or Cubic (Catmull-Rom).

**Data Structures and Layouts:**
The library uses PyTorch's RFFT format (following FFTW conventions), where the last spatial dimension is halved plus one to store real-valued symmetry [S1, unofficial]. All spatial dimensions must be even.

*   **2D Reconstruction Tensor:** Shape `[B, N, N/2+1]`, where $B$ is batch size and $N$ is the even spatial size [S1, unofficial].
*   **3D Reconstruction Tensor:** Shape `[B, D, H, W/2 + 1]`, where $B$ is batch size, and $D, H, W$ are even spatial dimensions [S1, unofficial].
*   **Projection Tensor:** Shape `[B, P, N, N/2+ 1]`, where $P$ is the number of poses (views) [S1, unofficial].
*   **2D Rotation Matrix:** Shape `[B, P, 2, 2]` [S1, unofficial].
*   **3D Rotation Matrix:** Shape `[B, P, 3, 3]` [S1, unofficial].
*   **Translation Shift Vector:** Shape `[B, P, 2]` [S1, unofficial].

**Parameters:**
*   **Catmull-Rom Parameter ($a$):** Set to $-0.5$ for cubic interpolation [S1, unofficial].

**Algorithm Steps:**

1.  **Preprocessing (Real Space Shift):** Apply `fftshift` to the reconstruction volume in real space to center the data at the 0th tensor element before performing the FFT [S1, unofficial].
2.  **Transform to Fourier Space:** Compute the FFT of the shifted real-space data to obtain the RFFT-formatted tensor [S1, unofficial].
3.  **Interpolation (Sampling):** Sample the Fourier volume at non-integer coordinates defined by the projection geometry [S1, unofficial].
    *   **Linear Interpolation:** Uses standard multilinear kernels: bilinear for 2D (4-point support) and trilinear for 3D (8-point support) [S1, unofficial].
    *   **Cubic Interpolation:** Uses separable Catmull-Rom kernels. Bicubic samples a $4 \times 4$ neighborhood; tricubic samples a $4 \times 4 \times 4$ neighborhood [S1, unofficial].
4.  **Phase Modulation (Translation):** Apply translation shifts via phase modulation in the Fourier domain [S1, unofficial]. Positive shift values move image contents along the positive axis direction. In back-projection, shifts are applied in the opposite direction to ensure the inverse relationship with forward projection [S1, unofficial].
5.  **Inverse Transform:** Apply `ifftshift` to restore conventional centering in real space, then compute the Inverse FFT (IFFT) to return to real space if the output is required in real space, or keep in RFFT format if staying in Fourier space [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.fft

def catmull_rom_kernel(s, a=-0.5):
    """Separable Catmull-Rom kernel for cubic interpolation."""
    abs_s = torch.abs(s)
    w = torch.zeros_like(abs_s)
    mask1 = abs_s <= 1
    w[mask1] = (a + 2) * abs_s[mask1]**3 - (a + 3) * abs_s[mask1]**2 + 1
    mask2 = (abs_s > 1) & (abs_s <= 2)
    w[mask2] = a * abs_s[mask2]**3 - 5 * a * abs_s[mask2]**2 + 8 * a * abs_s[mask2] - 4 * a
    return w

def fourier_projection(volume, rotations, translations, interpolation='linear'):
    """
    volume: torch.Tensor, real-space [B, D, H, W] or [B, N, N]
    rotations: torch.Tensor, [B, P, 2, 2] or [B, P, 3, 3]
    translations: torch.Tensor, [B, P, 2]
    interpolation: 'linear' or 'cubic'
    """
    B, P = rotations.shape[0], rotations.shape[1]
    ndim = volume.ndim - 2
    
    # 1. Preprocessing: Shift in real space to center at 0th element
    vol_shifted = torch.fft.fftshift(volume, dim=tuple(range(-ndim, 0)))
    
    # 2. Transform to Fourier Space (RFFT format)
    vol_rfft = torch.fft.rfft(vol_shifted, dim=tuple(range(-ndim, 0)))
    
    # 3. Generate frequency grid [G]
    freqs = [torch.fft.rfftfreq(volume.shape[-(i+1)], device=volume.device) for i in range(ndim)]
    k_grid = torch.stack(torch.meshgrid(freqs, indexing='ij'), dim=-1)
    
    # 4. Apply rotation and translation (phase modulation) [S1, unofficial]
    proj_rfft = torch.zeros((B, P, *vol_rfft.shape[-ndim:]), device=volume.device)
    for b in range(B):
        for p in range(P):
            R = rotations[b, p]
            t = translations[b, p]
            k_rot = k_grid @ R.T
            phase = torch.exp(-1j * 2 * torch.pi * (k_rot * t).sum(dim=-1, keepdim=True))
            
            # Sample and interpolate vol_rfft at rotated coordinates
            # The exact coordinate mapping follows the Fourier slice theorem [G]
            # Cubic interpolation applies the separable Catmull-Rom kernel along each axis
            if interpolation == 'cubic':
                weights = catmull_rom_kernel(k_rot)
                proj_rfft[b, p] = vol_rfft[b] * phase * weights
            else:
                # Linear interpolation uses standard multilinear kernels [S1, unofficial]
                proj_rfft[b, p] = vol_rfft[b] * phase
                
    # 5. Inverse Transform
    proj_real = torch.fft.irfft(proj_rfft, dim=tuple(range(-ndim, 0)))
    proj_real = torch.fft.ifftshift(proj_real, dim=tuple(range(-ndim, 0)))
    return proj_real
```

**Edge Cases and Pitfalls:**
*   **Even Dimensions:** All spatial dimensions must be even. Odd sizes are not supported [S1, unofficial].
*   **Square/Cubic:** 2D reconstructions must be square; 3D reconstructions must be cubic [S1, unofficial].
*   **Shift Direction:** In back-projection, translation shifts must be applied in the opposite direction to the forward projection to maintain the adjoint relationship [S1, unofficial].
*   **Batch Broadcasting:** The batch dimension $B$ can be 1 or match the reconstruction batch size. The pose dimension $P$ in rotations or shifts can be 1 for shared values across poses [S1, unofficial].

## Subtopics

### [TorchRadon](7278-torchradon.md)
TorchRadon is a PyTorch-integrated library providing differentiable forward and backward Radon projection operators for computed tomography. It solves the problem of computing line integrals (projections) and their adjoints (back-projections) within a deep learning pipeline, enabling gradient-based optimization for tomographic reconstruction. Unlike standard non-differentiable C++ libraries, TorchRadon allows backpropagation of gradients through the projection operations, which is essential for training generative models or optimizing latent variables in inverse rendering contexts.

### Forward Projection (Differentiable GPU Ray Projector)
A differentiable simulation of the X-ray projection process, implemented by rotating the volume and summing along one axis [S2, unofficial]. For each angle in the angles list: rotate the volume around the Z-axis by the angle [S2, unofficial]. Sum the rotated volume along the first dimension (depth) to create a 2D projection [S2, unofficial]. Stack all 2D projections into a single tensor [S2, unofficial]. Rotation is performed using `affine_grid` and `grid_sample` with `align_corners=False` [S2, unofficial].

## Operating characteristics
*   **Performance:** Outperforms `torch-fourier-slice` by 1–2 orders of magnitude [S1, unofficial].
*   **Hardware:** Optimized for CPU, Apple Silicon (MPS), and CUDA devices [S1, unofficial].
*   **Complexity:** Not given in the sources.
*   **Strengths:** High performance, differentiable, supports linear and cubic interpolation, works in RFFT format.
*   **Limitations:** Requires even spatial dimensions, square/cubic volumes, and specific tensor layouts.

## Background
The method originates from the `torch-projectors` library, introduced in a paper focusing on electron-particle analysis and electron tomography algorithms [S1, unofficial]. Machine learning methods in this domain require differentiable implementations for end-to-end model training, but PyTorch's built-in operations were too slow for practical use [S1, unofficial]. The library follows FFTW conventions for RFFT format, distinguishing itself from implementations like RELION by avoiding intermediate format changes [S1, unofficial].

## Sources
[S1] torch-projectors: A High-Performance Differentiable Projection Library for PyTorch, https://doi.org/10.64898/2026.03.10.710885, fetched 2026-10-26 (unofficial)
[S2] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
