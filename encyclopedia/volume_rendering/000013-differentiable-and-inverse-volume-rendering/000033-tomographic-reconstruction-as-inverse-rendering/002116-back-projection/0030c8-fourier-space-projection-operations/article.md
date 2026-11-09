# Fourier-space Projection Operations

Also called: torch-projectors

## Overview
Fourier-space projection operations compute 2D and 3D forward and backward (adjoint) projection operators for tomographic reconstruction by operating exclusively in the Fourier domain. It solves the problem of efficiently computing line integrals and their gradients for differentiable volume rendering, specifically addressing the performance limitations of standard spatial-domain implementations in deep learning frameworks [S1, unofficial]. It distinguishes itself by maintaining data in the Real Fast Fourier Transform (RFFT) format throughout the pipeline, avoiding intermediate format conversions, and achieving performance improvements of 1–2 orders of magnitude over libraries like `torch-fourier-slice` [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Reconstruction Volume:** A tensor representing the 3D or 2D object in Fourier space.
*   **Projection Data:** A tensor representing the 2D projections in Fourier space.
*   **Geometry:** Rotation matrices and translation vectors defining the pose of each projection.
*   **Interpolation Type:** Linear (bilinear/trilinear) or Cubic (Catmull-Rom).

**Data Structures and Layouts:**
The library uses PyTorch's RFFT format (following FFTW conventions), where the last spatial dimension is halved plus one to store real-valued symmetry [S1, unofficial]. All spatial dimensions must be even.

*   **2D Reconstruction Tensor:** Shape `[B, N, N/2+1]`, where $B$ is batch size and $N$ is the even spatial size.
*   **3D Reconstruction Tensor:** Shape `[B, D, H, W/2 + 1]`, where $B$ is batch size, and $D, H, W$ are even spatial dimensions.
*   **Projection Tensor:** Shape `[B, P, N, N/2+ 1]`, where $P$ is the number of poses (views).
*   **2D Rotation Matrix:** Shape `[B, P, 2, 2]`.
*   **3D Rotation Matrix:** Shape `[B, P, 3, 3]`.
*   **Translation Shift Vector:** Shape `[B, P, 2]`.

**Parameters:**
*   **Catmull-Rom Parameter ($a$):** Set to $-0.5$ for cubic interpolation [S1, unofficial].

**Algorithm Steps:**

1.  **Preprocessing (Real Space Shift):**
    Apply `fftshift` to the reconstruction volume in real space to center the data at the 0th tensor element before performing the FFT. For back-projection, inverse shifts are applied in the opposite direction to ensure the adjoint relationship [S1, unofficial].

2.  **Transform to Fourier Space:**
    Compute the FFT of the shifted real-space data to obtain the RFFT-formatted tensor.

3.  **Interpolation (Sampling):**
    Sample the Fourier volume at non-integer coordinates defined by the projection geometry.
    *   **Linear Interpolation:** Uses standard multilinear kernels. Bilinear for 2D (4-point support) and trilinear for 3D (8-point support) [S1, unofficial].
    *   **Cubic Interpolation:** Uses separable Catmull-Rom kernels. Bicubic samples a $4 \times 4$ neighborhood; tricubic samples a $4 \times 4 \times 4$ neighborhood [S1, unofficial].

4.  **Phase Modulation (Translation):**
    Apply translation shifts via phase modulation in the Fourier domain. A shift vector $\mathbf{t}$ corresponds to multiplying by $e^{-i 2\pi \mathbf{k} \cdot \mathbf{t}}$, where $\mathbf{k}$ is the frequency coordinate [S1, unofficial].

5.  **Inverse Transform (if Back-projection):**
    Apply `ifftshift` to restore conventional centering in real space, then compute the Inverse FFT (IFFT) to return to real space if the output is required in real space, or keep in RFFT format if staying in Fourier space.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.fft

def fourier_projection(volume, projections, rotations, translations, interpolation='linear'):
    """
    volume: torch.Tensor, shape [B, D, H, W/2+1] (3D) or [B, N, N/2+1] (2D) in RFFT format.
    projections: torch.Tensor, shape [B, P, N, N/2+1] in RFFT format.
    rotations: torch.Tensor, shape [B, P, 2, 2] or [B, P, 3, 3].
    translations: torch.Tensor, shape [B, P, 2].
    """
    # 1. Preprocessing: Shift to center at 0th element in real space
    # Note: If input is already in RFFT, we must shift in real space first.
    # Assuming input 'volume' is in RFFT, we need to convert to real, shift, then FFT?
    # The source states: "Reconstruction and projection data must be shifted (fftshift) 
    # in real space... before fast Fourier transform (FFT)."
    # This implies the input to this function should be real-space data, or we handle the shift.
    # However, the library operates in Fourier space. Let's assume the standard workflow:
    # Real Space -> Shift -> FFT -> Fourier Ops.
    
    # If volume is passed as RFFT, we must inverse FFT to real, shift, then FFT again?
    # Or does the library expect real-space input?
    # "torch-projectors operates exclusively in Fourier space... tensors remain RFFT-formatted 
    # throughout projection operations".
    # This suggests the inputs are already in RFFT. But the shift instruction is for "before FFT".
    # This implies the library handles the real-to-complex transition internally or expects 
    # the user to provide real-space data which is then shifted and FFT'd.
    # Given "inputs and outputs" usually implies the API signature, and the shapes are RFFT,
    # let's assume the function takes RFFT data but the internal logic requires the shift 
    # to have been applied to the real-space representation before the RFFT was taken, 
    # OR the library provides a helper to shift RFFT data correctly.
    # Actually, fftshift on RFFT data is non-trivial. The source says "shift ... in real space ... before FFT".
    # This implies the standard pipeline is: Real Data -> fftshift -> FFT -> RFFT Data.
    # If the function takes RFFT data, it assumes the shift was done prior.
    
    # Let's write the core Fourier-space operation assuming inputs are correctly shifted RFFT tensors.
    
    B, P = rotations.shape[0], rotations.shape[1]
    
    # 2. Generate Frequency Grid
    # This depends on the spatial dimensions of the volume.
    # For 3D: D, H, W. For 2D: N.
    # We need the frequency coordinates k_x, k_y, k_z corresponding to the RFFT grid.
    # This is a standard operation: torch.fft.fftfreq or similar, adjusted for RFFT.
    # Not given in sources, but standard for Fourier methods. [G]
    
    # 3. Apply Rotation and Translation to Frequency Coordinates
    # The projection direction is determined by the rotation matrix.
    # The translation is applied via phase modulation.
    
    # 4. Interpolation
    # Sample the volume at the projected coordinates.
    # This involves mapping the 2D projection grid to the 3D Fourier volume.
    # The source mentions "oversampling" is supported but doesn't detail the grid generation.
    
    # 5. Compute Projection
    # Sum/Integrate along the projection path in Fourier space.
    # This is the core "Fourier Slice Theorem" application.
    
    # Since the exact interpolation kernel application in RFFT is complex and not fully detailed 
    # in the provided text beyond the kernel definition, we outline the logical steps.
    
    # Cubic Kernel Definition (Catmull-Rom, a = -0.5)
    def catmull_rom(s, a=-0.5):
        abs_s = torch.abs(s)
        w = torch.zeros_like(abs_s)
        # Case 1: |s| <= 1
        mask1 = abs_s <= 1
        w[mask1] = (a + 2) * abs_s[mask1]**3 - (a + 3) * abs_s[mask1]**2 + 1
        # Case 2: 1 < |s| <= 2
        mask2 = (abs_s > 1) & (abs_s <= 2)
        w[mask2] = a * abs_s[mask2]**3 - 5 * a * abs_s[mask2]**2 + 8 * a * abs_s[mask2] - 4 * a
        return w

    # Linear Interpolation (Bilinear/Trilinear)
    # Uses 4-point (2D) or 8-point (3D) support.
    
    # The actual implementation of sampling RFFT data at arbitrary coordinates 
    # requires careful handling of the Hermitian symmetry.
    
    return projections # Placeholder for the computed projection tensor
```

**Edge Cases and Pitfalls:**
*   **Even Dimensions:** All spatial dimensions must be even. Odd sizes are not supported [S1, unofficial].
*   **Square/Cubic:** 2D reconstructions must be square; 3D reconstructions must be cubic [S1, unofficial].
*   **Shift Direction:** In back-projection, translation shifts must be applied in the opposite direction to the forward projection to maintain the adjoint relationship [S1, unofficial].
*   **Batch Broadcasting:** The batch dimension $B$ can be 1 or match the reconstruction batch size. The pose dimension $P$ in rotations or shifts can be 1 for shared values across poses [S1, unofficial].

## Subtopics

### [TorchRadon](7278-torchradon.md)
TorchRadon is a PyTorch-integrated library providing differentiable forward and backward Radon projection operators for computed tomography. It solves the problem of computing line integrals (forward projection) and their adjoints (backward projection) within a deep learning framework, enabling gradient-based optimization for tomographic reconstruction. It distinguishes itself by offering high-performance CUDA implementations of these linear operators with support for both parallel-beam and fan-beam geometries, directly supporting backpropagation through the projection steps.

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
