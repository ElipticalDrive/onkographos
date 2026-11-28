# Domain-transform methods

Also called: Fourier-space techniques, frequency-domain methods

## Overview
Domain-transform methods solve the problem of high computational complexity in spatial-domain volume rendering by transforming volumetric data into an alternative domain (typically frequency, compression, or wavelet) where projections can be generated more efficiently [S2]. Unlike object-order methods that rasterize primitives, these techniques compute a global transformation of the entire volume, extract a slice or representation in the transformed domain, and inverse-transform to obtain the final image [S1, unofficial][S2]. This approach reduces per-view rendering complexity to $O(N^2)$ (disregarding the one-time preprocessing cost), making it suitable for interactive X-ray simulation and medical imaging where occlusion effects are not required [S1, unofficial]. The primary drawback is the inability to interactively modify display parameters such as opacity or lighting after the initial transformation [S1, unofficial].

## Shared principles
All domain-transform methods follow a three-stage pipeline: Global Transformation, Slice Extraction (or Domain Evaluation), and Inverse Transformation [S1, unofficial][S2].

1.  **Global Transformation:** The input 3D scalar volume $f(x, y, z)$ is transformed into a global representation in an alternative domain [S2]. For frequency-domain methods, this is a 3D Discrete Fourier Transform (DFT) or Discrete Hartley Transform (DHT) [G]. For compression-based methods, it involves transforming coefficients into a compressed block format [S2]. This step is performed once per volume and has a computational complexity of $O(N^3 \log N)$ using Fast Fourier Transform (FFT) algorithms [G].
2.  **Slice Extraction:** A 2D slice or projection is extracted from the global representation corresponding to the desired viewing direction [S1, unofficial]. In frequency-domain methods, this relies on the **Projection-Slice Theorem**, which states that the 2D Fourier transform of a projection is equal to a central slice of the 3D Fourier transform perpendicular to the viewing direction [G]. Since the slicing plane rarely aligns with the discrete lattice of the transformed data, interpolation (e.g., trilinear, tri-cubic, or windowed sinc) is required to resample the data onto a 2D grid [G].
3.  **Inverse Transformation:** The extracted 2D slice is inverse-transformed back to the spatial domain to produce the final 2D projection image [S1, unofficial]. For frequency methods, this is a 2D Inverse FFT (IFFT) [S1, unofficial].

**Common Pitfalls:**
*   **Boundary Artifacts:** Transforms assume periodic boundary conditions [G]. Discontinuities at volume boundaries cause high-frequency artifacts (ghosting) [G]. Mitigation includes zero-padding the volume by a factor of $k_2$ (typically $4/3$) and applying vignetting correction (premultiplying by the reciprocal of the interpolation filter response) [G].
*   **Aliasing:** Insufficient output resolution violates the Nyquist criterion, causing aliasing [G].
*   **Fixed Parameters:** Once transformed, parameters like opacity and lighting cannot be changed interactively [S1, unofficial].

## Subtopics

### [Fourier Projection-Slice Volume Rendering](824-fourier-projection-slice-volume-rendering.md)
Generates attenuation-only, X-ray-like projections by exploiting the Fourier projection-slice theorem. It reduces per-view complexity to $O(N^2 \log N)$ by transforming the volume into the frequency domain once. Choose it for interactive X-ray simulation and medical imaging where occlusion is not required.

### [GPU-based 2D FFT](841-gpu-based-2d-fft.md)
Hardware-accelerated method for computing the DFT on GPUs, adapted for the domain-transform pipeline. It sets itself apart by utilizing texture memory and fragment shaders for parallel butterfly operations. Choose it when rendering performance is bottlenecked by 2D slice transformations.

### [Hartley Transform for Memory Reduction](895-hartley-transform-for-memory-reduction.md)
Replaces the standard DFT with the Discrete Hartley Transform (DHT) to map real-valued spatial data directly to real-valued spectra. It sets itself apart by eliminating complex number arithmetic, reducing memory requirements by half compared to a full complex spectrum. Choose it when memory bandwidth is a constraint.

### [Frequency Domain Volume Rendering](952-frequency-domain-volume-rendering.md)
Transforms scalar data to the frequency domain, extracts a 2D projection slice, and inverse-transforms to the spatial domain. It sets itself apart by decoupling rendering performance from dataset resolution via the Projection-Slice Theorem. Choose it for large volumes where rendering speed depends primarily on output image resolution.

### [Domain Repetition](1218-domain-repetition.md)
Creates infinite, periodic repetitions of an object in space by wrapping query coordinates using modulo arithmetic. It sets itself apart by tiling geometry without increasing polygon count. Choose it for rendering infinite grids of primitives.

### [Fourier Transform](2288-fourier-transform.md)
Converts a 3D scalar volume into the frequency domain to enable rendering via the Projection-Slice Theorem. It sets itself apart by performing a global transformation that allows efficient convolution but requires significant preprocessing memory. Choose it when global frequency-domain operations are needed.

### [Translation-invariant Fluorescence Rendering via Fourier Convolution](5085-translation-invariant-fluorescence-rendering-via-fourier-con.md)
Renders 3D fluorescence microscopy images by convolving a parameterized fluorophore density with a translation-invariant Point Spread Function (PSF) in the Fourier domain. It sets itself apart by modeling physical blurring and enabling gradient-based inference of shape and optical properties. Choose it for differentiable 3D image synthesis and microscopy optimization.

### [DCT-based Volume Rendering](5396-dct-based-volume-rendering.md)
Represents a 3D scalar volume using cosine waves as basis functions, mapping real input values to real output values. It sets itself apart by operating on compressed spatial blocks rather than a global frequency domain, allowing out-of-core rendering. Choose it for massive time-series data that exceeds main memory capacity.

### [Hartley Transform](6324-hartley-transform.md)
A real-valued integral transform that maps a real input function to a real spectrum, serving as an alternative to the complex DFT. It sets itself apart by using the $\cos + \sin$ kernel instead of the complex exponential, avoiding complex arithmetic. Choose it for real-valued signal processing in domain-transform pipelines.

### [Frequency-Domain Methods](9801-frequency-domain-methods.md)
Represents 4D spacetime volumetric data as a superposition of Fourier basis functions, utilizing CP tensor decomposition to reduce memory and computation. It sets itself apart by operating orthogonally to spatial decomposition and focusing on explicit spectral representation. Choose it for compact storage and reconstruction of dynamic volumetric data.

## Background
Domain-transform methods, also known as Fourier-space techniques, were introduced to include algorithms working in the frequency domain [S1, unofficial]. The basic idea is to compute the 3D Fourier transformation of the entire volume in a pre-processing step, extract a slice image in the frequency domain, and transform it back into the spatial domain using the 2D inverse Fourier transformation [S1, unofficial]. The main advantage is the computational complexity of $O(N^2)$ for the projection, disregarding the one-time 3D Fourier transformation pre-processing step [S1, unofficial]. The major drawback is that they do not allow an interactive modification of any display parameter other than the viewing direction [S1, unofficial]. These methods fall under the broader category of domain methods, where spatial data is transformed into an alternative domain (such as compression, wavelet, or frequency domain) from which a projection is directly generated [S2].

## Sources
[S1] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S2] Volume visualization, https://dl.acm.org/doi/pdf/10.1145/234313.234383, fetched 2026-11-01
