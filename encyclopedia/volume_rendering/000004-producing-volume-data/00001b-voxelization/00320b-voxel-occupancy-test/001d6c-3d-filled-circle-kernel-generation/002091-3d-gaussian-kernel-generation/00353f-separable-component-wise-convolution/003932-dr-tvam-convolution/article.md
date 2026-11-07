# Dr. TVAM Convolution

Also called: Separable 3D Convolution with Gaussian Diffusion Kernels

## Overview
Dr. TVAM Convolution is a separable 3D convolution method using Gaussian diffusion kernels, implemented via Dr.Jit's `dr.convolve` function for tomographic volumetric additive manufacturing simulations [S1, unofficial]. It builds on [Separable Component-Wise Convolution](7446-separable-component-wise-convolution.md) by applying axis-aligned Gaussian smoothing to decomposed low-rank tensor volumes, specifically tailored for the physical forward models of additive manufacturing processes [S1, unofficial]. Unlike general-purpose Gaussian blurring, this method explicitly models diffusion dynamics through time-step and diffusivity parameters, distinguishing it from the illustrative filtering approach of [Separable 3D Gaussian Filter](#separable-3d-gaussian-filter) which focuses on spatial importance maps for visualization [S2, unofficial].

## How it works
**Inputs:**
- Volume data tensor $r$ (typically a Dr.Jit tensor representing the volume state).
- Spatial spacing parameters: $spacing_z, spacing_x, spacing_y$ (floats, units: length per voxel).
- Kernel radius parameters: $radius_z, radius_x, radius_y$ (integers, default: 10).
- Physical parameters: $diffusion\_D$ (diffusivity constant), $\delta\_t$ (time step).

**Outputs:**
- Filtered volume tensor $r_{filtered}$ with the same shape and layout as input $r$.

**Data Structures:**
- The kernel is a 1D array of floats. For each axis, the kernel size is $2 \times radius + 1$.
- Indices $i$ range from $-radius$ to $radius$.
- Coordinates $x_i = i \times spacing$.
- Kernel values $K_i = \exp\left(\frac{-x_i^2}{4 \cdot diffusion\_D \cdot \delta\_t}\right)$.
- The kernel is normalized such that $\sum K_i = 1$.

**Steps:**
1. **Kernel Generation:** Construct three 1D Gaussian diffusion kernels (one for each axis Z, X, Y) using the specific spacing and physical parameters for that axis.
2. **Z-Axis Convolution:** Apply separable convolution along the Z-axis (axis 0) using the Z-kernel.
3. **X-Axis Convolution:** Apply separable convolution along the X-axis (axis 1) using the X-kernel.
4. **Y-Axis Convolution:** Apply separable convolution along the Y-axis (axis 2) using the Y-kernel.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import drjit as dr
import numpy as np

def dr_tvam_convolution(r, spacing_z, spacing_x, spacing_y, 
                        radius_z=10, radius_x=10, radius_y=10,
                        diffusion_D=1.0, delta_t=1.0):
    """
    Performs separable 3D convolution with Gaussian diffusion kernels.
    
    Args:
        r: Input volume tensor (Dr.Jit tensor).
        spacing_z, spacing_x, spacing_y: Spatial spacing for each axis.
        radius_z, radius_x, radius_y: Kernel radius for each axis.
        diffusion_D: Diffusivity constant.
        delta_t: Time step.
        
    Returns:
        Filtered volume tensor.
    """
    # Helper to generate 1D Gaussian diffusion kernel
    def generate_kernel_1d(spacing, radius, diffusion_D, delta_t):
        # Indices from -radius to radius
        indices = np.arange(-radius, radius + 1)
        # Coordinates
        coords = indices * spacing
        # Gaussian diffusion formula: exp(-x^2 / (4 * D * dt))
        # Note: The source formula uses 4 * diffusion_D * delta_t in the denominator
        kernel_values = np.exp(-(coords ** 2) / (4 * diffusion_D * delta_t))
        # Normalize
        kernel_values = kernel_values / np.sum(kernel_values)
        return kernel_values

    # Generate kernels for each axis
    filter_z = generate_kernel_1d(spacing_z, radius_z, diffusion_D, delta_t)
    filter_x = generate_kernel_1d(spacing_x, radius_x, diffusion_D, delta_t)
    filter_y = generate_kernel_1d(spacing_y, radius_y, diffusion_D, delta_t)

    # Apply separable convolution along Z-axis (axis 0)
    # dr.convolve is assumed to be available from the Dr.Jit library context
    # The source specifies: dr.convolve(r, filter=filter_z, filter_radius=radiusz, axis=0)
    r = dr.convolve(r, filter=filter_z, filter_radius=radius_z, axis=0)

    # Apply separable convolution along X-axis (axis 1)
    r = dr.convolve(r, filter=filter_x, filter_radius=radius_x, axis=1)

    # Apply separable convolution along Y-axis (axis 2)
    r = dr.convolve(r, filter=filter_y, filter_radius=radius_y, axis=2)

    return r
```

**Edge Cases and Pitfalls:**
- **Precision:** The exponential calculation must handle small values carefully to avoid underflow if $diffusion\_D \cdot \delta\_t$ is very small relative to coordinate squared.
- **Boundaries:** The `dr.convolve` implementation must handle boundary conditions (e.g., zero-padding, reflection, or periodic) consistently with the physical simulation requirements. The source does not specify the boundary mode used by `dr.convolve` [S1, unofficial].
- **Degenerate Input:** If $diffusion\_D$ or $\delta\_t$ is zero, the denominator becomes zero. This is a physical singularity; in practice, a small epsilon should be added to the denominator or the time step must be non-zero [G].

## Subtopics

### Separable 3D Gaussian Filter
A 3D Gaussian filter implemented as three consecutive 1D filters to leverage separability for interactive performance [S2, unofficial]. It is used in volume illustration to compute spatial importance maps for modulating color and opacity transfer functions [S2, unofficial]. Unlike Dr. TVAM Convolution, it subtracts the original volume from the filtered volume to highlight high-frequency details or edges, rather than modeling physical diffusion [S2, unofficial]. It is implemented in CUDA for interactive rates [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot (radius_x + radius_y + radius_z))$ where $N$ is the number of voxels, due to the separability of the 3D Gaussian into three 1D passes.
- **Memory Complexity:** $O(N)$ for the input and output volumes, plus $O(radius)$ for the kernel storage.
- **Performance:** The source [S1, unofficial] does not provide specific timing benchmarks for the Dr. TVAM convolution itself, only noting it is implemented via Dr.Jit. The related [Separable 3D Gaussian Filter] provides timings for a CUDA implementation on CT datasets (e.g., 256x256x415) showing interactive rates are achievable [S2, unofficial].
- **Strengths:** Computationally efficient due to separability; physically motivated for diffusion processes in additive manufacturing [S1, unofficial].
- **Limitations:** Requires specific physical parameters ($D, \delta_t$); boundary conditions are not detailed in the source [S1, unofficial].

## Background
The method originates from the Dr. TVAM project (Inverse Rendering for Tomographic Volumetric Additive Manufacturing) [S1, unofficial]. It is designed to simulate the physical forward model of volumetric additive manufacturing, where light or material diffusion is modeled via Gaussian kernels. The implementation relies on the Dr.Jit library for differentiable programming [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Illustration Using Intensity Filtering, https://doi.org/10.2312/compaesth/compaesth10/051-058, fetched 2026-10-26 (unofficial)
