# TorchRadon

Also called: Differentiable Radon Transform, Implementation Details (TorchRadon), TorchRadon library

## Overview
TorchRadon is a PyTorch-integrated library providing high-performance, differentiable forward and backward projection operators for computed tomography [S2, unofficial]. It solves the problem of computing Radon transforms and their adjoints within a deep learning pipeline, enabling gradient-based optimization for tomographic reconstruction [S1, unofficial]. It distinguishes itself from non-differentiable alternatives by supporting automatic differentiation for all inputs, allowing gradients to flow through the projection and backprojection operations [S1, unofficial].

## How it works
TorchRadon implements linear projection operators that approximate the continuous Radon transform. The core operations are the forward projection (mapping an image to a sinogram) and the backward projection (mapping a sinogram to an image), corresponding to matrix-vector multiplications with $A$ and $A^T$ respectively [S1, unofficial].

**Inputs and Outputs:**
*   **Forward Projection:** Input is a 2D image tensor $x$ of shape $(N, 1, H, W)$ or $(H, W)$. Output is a sinogram tensor $y$ of shape $(N, 1, D, A)$, where $D$ is the number of detector bins and $A$ is the number of angles.
*   **Backward Projection:** Input is a sinogram tensor $y$. Output is an image tensor $x_{bp}$ of shape $(N, 1, H, W)$ or $(H, W)$.

**Data Structures:**
The library uses PyTorch tensors. The geometry is defined by parameters such as image size, number of angles, detector spacing, and interpolation method.

**Parameters:**
*   `angles`: Array of sampling angles in radians. For parallel beam, typically $[0, \pi)$. For fan-beam, $[0, 2\pi)$ [S2, unofficial].
*   `det_count`: Number of detector pixels.
*   `det_spacing`: Distance between detector pixels.
*   `source_distance`: Distance from X-ray source to object center (fan-beam only). Default: 512 [S2, unofficial].
*   `det_distance`: Distance from detector to object center (fan-beam only). Default: equals `source_distance` [S2, unofficial].
*   Interpolation: Linear or cubic [Parent].

**Algorithm Steps:**
1.  Initialize the Radon operator with specified geometry parameters (image size, angles, detector configuration) [S2, unofficial].
2.  **Forward Projection:** For each angle $\theta$, compute the line integrals of the image $x$ along parallel (or fan-beam) rays. This is implemented using differentiable interpolation kernels to sample the image at sub-pixel locations determined by the ray geometry [S2, unofficial].
3.  **Backward Projection:** For each angle $\theta$, distribute the sinogram values back to the image grid. This involves interpolating the sinogram values and adding them to the corresponding image pixels, effectively accumulating contributions from all angles [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from torchradon import Radon, RadonFanbeam

# Define image size and angles
size = 512
angles = np.linspace(0, np.pi, 512, endpoint=False)

# Initialize parallel beam Radon operator
radon = Radon(size, angles)

# Create a dummy input image (batch of 1, 1 channel, 512x512)
x = torch.randn(1, 1, size, size)

# Forward projection: Image -> Sinogram
# y shape: (1, 1, 512, 512) -> (1, 1, 512, 512) where last dim is angles
sino = radon.forward(x)

# Backward projection: Sinogram -> Image
# This approximates A^T * y
bp = radon.backprojection(sino)

# Fan-beam example
angles_fan = np.linspace(0, 2 * np.pi, 512, endpoint=False)
radon_fan = RadonFanbeam(size, angles_fan, source_distance=512)
sino_fan = radon_fan.forward(x)
bp_fan = radon_fan.backprojection(sino_fan)
```

**Edge Cases and Pitfalls:**
*   **Boundary Effects:** Rays passing near the edge of the image may have incomplete support for interpolation kernels, potentially causing artifacts.
*   **Precision:** The library uses floating-point arithmetic. Relative errors compared to reference implementations (e.g., Astra Toolbox) are small but non-zero [S2, unofficial].
*   **Memory:** High-resolution images and large numbers of angles increase memory consumption linearly with the sinogram size.

## Subtopics

### [Radon Transform](11306-radon-transform.md)
The Radon transform maps a 2D image to its projections (sinogram) by integrating pixel intensities along parallel lines at various angles. It solves the forward acquisition model in computed tomography, converting spatial domain data into projection domain data. Unlike its inverse counterpart (e.g., Filtered Back-Projection), which reconstructs images from projections, the forward Radon transform generates the measurements required for reconstruction. It is distinguished from other projection methods by its strict parallel-beam geometry and linear integration nature.

### Fan-Beam Radon Transform
Computes Radon transforms for fan-beam geometry where X-rays originate from a point source [S2, unofficial]. It is distinguished from parallel-beam by the diverging ray paths. Parameters include `source_distance` (default 512) and `det_distance` (default equals `source_distance`) [S2, unofficial]. Angles are equally spaced in $[0, 2\pi)$ [S2, unofficial]. Relative error compared to Astra Toolbox is $8.34 \cdot 10^{-4}$ for forward projection and $4.84 \cdot 10^{-5}$ for backprojection [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels and angles. The differentiable nature adds minimal overhead compared to non-differentiable C++ implementations due to GPU acceleration [S2, unofficial].
*   **Memory Complexity:** Proportional to the size of the sinogram ($H \times W \times A$).
*   **Performance:** High performance on NVIDIA GPUs via PyTorch [S1, unofficial].
*   **Accuracy:** Relative errors are low ($< 10^{-3}$) compared to established toolboxes like Astra [S2, unofficial].
*   **Strengths:** Seamless integration with PyTorch, differentiability, support for both parallel and fan-beam geometries.
*   **Limitations:** Limited to 2D images in the core library (3D support may vary by version); interpolation artifacts at boundaries.

## Background
TorchRadon was developed to provide a fast, differentiable implementation of Radon transforms for use in deep learning-based tomographic reconstruction [S2, unofficial]. It is integrated with PyTorch, allowing for end-to-end training of reconstruction networks [S1, unofficial]. The library supports various interpolation methods and filter types for filtered backprojection [S2, unofficial].

## Sources
[S1] Generative Tomography Reconstruction, https://core.ac.uk/works/91148923, fetched 2026-10-31 (unofficial)
[S2] TorchRadon: Fast Differentiable Routines for Computed Tomography, https://doi.org/10.48550/arxiv.2009.14788, fetched 2026-10-26 (unofficial)
