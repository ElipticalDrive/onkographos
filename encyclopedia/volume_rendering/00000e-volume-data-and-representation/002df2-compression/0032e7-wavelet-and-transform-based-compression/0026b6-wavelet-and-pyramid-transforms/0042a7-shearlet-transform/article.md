# Shearlet Transform
Also called: Alpha Shearlet Transform, AlphaTransforms-based Shearlet Transform

## Overview
The Shearlet Transform is a GPU-accelerated implementation of the Alpha Shearlet transform that computes Fourier coefficients once at initialization and performs all subsequent operations on the GPU to allow parallel batch processing [S1, unofficial]. It differs from the AlphaTransforms library by moving all post-initialization operations to the GPU [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 2D image or 3D volume $x$ [S1, unofficial].
- Output: A set of shearlet coefficients $C$ [S1, unofficial].
- Reconstruction: The adjoint transform $\mathcal{S}^*$ applied to $C$ yields the reconstructed volume $\hat{x}$ [S1, unofficial].

**Data Structures:**
The implementation relies on a precomputed set of Fourier coefficients computed once at initialization, which can be cached on disk [S1, unofficial]. These coefficients define the basis functions in the frequency domain [G]. For a volume of size $N \times N$ (or $N \times N \times N$), the transform generates a specific number of coefficients based on the number of scales and shear parameters [S1, unofficial].

**Parameters:**
- $\alpha$: The alpha parameter for the shearlet transform. Typically set to $0.5$ to normalize on the Fourier side to get a Parseval frame [S1, unofficial].
- $n_{\text{scales}}$: The number of scales for the transform. A typical value is $5$ [S1, unofficial].
- Image Size: The spatial dimensions of the input data, e.g., $512 \times 512$ [S1, unofficial].

**Steps:**
1. **Initialization:** Create the transform object with the specified image size, number of scales, and alpha values. The Fourier coefficients are computed and stored [S1, unofficial].
2. **Forward Transform:** Compute the shearlet coefficients by applying the forward transform to the input volume $x$ [S1, unofficial].
3. **Backward Transform (Reconstruction):** Reconstruct the volume from the coefficients using the adjoint transform [S1, unofficial]. If $\alpha=0.5$, the composition of the transform with its adjoint is the identity mapping [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization: Compute Fourier coefficients once at initialization [S1, unofficial]
# Internal basis generation math is not given in the sources.
fourier_coeffs = generate_fourier_basis(width, height, [alpha] * n_scales)

# Forward Transform: Compute shearlet coefficients [S1, unofficial]
# Internal projection math is not given in the sources.
coeff = project_onto_basis(x, fourier_coeffs)

# Backward Transform (Reconstruction): Reconstruct image using adjoint transform [S1, unofficial]
# Internal adjoint math is not given in the sources.
rec = apply_adjoint(coeff, fourier_coeffs)
```

**Edge Cases and Pitfalls:**
- **Precision:** Half precision (16-bit) data is packed 4 images into the channels of a single texture to reduce memory reads by a factor of 4, then converted to single precision (32-bit) for arithmetic to minimize accuracy loss [S1, unofficial].
- **Accuracy Loss:** Converting from single to half precision introduces a relative error [S1, unofficial]. For the Shepp-Logan phantom, the relative error between single and half representation is $1.40 \cdot 10^{-4}$ [S1, unofficial]. The relative error between AlphaTransforms coefficients and TorchRadon coefficients is $3.86 \cdot 10^{-7}$ using single precision [S1, unofficial].
- **Parseval Frame:** Setting $\alpha = 0.5$ ensures the transform forms a Parseval frame, meaning the energy is preserved and the adjoint acts as the inverse [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Performance:** Using half precision with texture packing provides a speedup of more than $2.5\times$ compared to single precision [S1, unofficial].
- **Strengths:** GPU-accelerated, supports parallel batch processing [S1, unofficial].
- **Limitations:** Requires precomputation of Fourier coefficients at initialization [S1, unofficial].

## Background
The Shearlet Transform is implemented in the TorchRadon library, which provides fast differentiable routines for computed tomography [S1, unofficial]. The implementation is based on the AlphaTransforms library but differs by performing all operations after Fourier coefficient initialization on the GPU, enabling parallel batch processing [S1, unofficial]. The specific variant discussed uses real-valued alpha-shearlets with $\alpha = 0.5$ and $5$ scales, producing $59$ coefficients for a $512 \times 512$ image [S1, unofficial].

## Sources
[S1] TorchRadon: Fast Differentiable Routines for Computed Tomography, https://doi.org/10.48550/arxiv.2009.14788, fetched 2026-10-26 (unofficial)
