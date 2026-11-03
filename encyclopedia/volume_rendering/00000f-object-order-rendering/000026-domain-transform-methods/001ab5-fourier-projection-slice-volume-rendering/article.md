# Fourier Projection-Slice Volume Rendering

Also called: Fourier projection rendering, Fourier approach to volume rendering, Attenuation-only rendering via Fourier projection-slice theorem, Frequency Domain Volume Rendering

## Overview
Fourier Projection-Slice Volume Rendering is a method for generating attenuation-only images (X-ray-like projections) by exploiting the Fourier projection-slice theorem to compute projections in $O(N^2 \log N)$ time per view, avoiding the $O(N^3)$ cost of digital compositing [S1, S2, S4, S5]. Unlike conventional volume rendering or ray-casting, this method does not exhibit occlusion or hidden surface removal; it computes an order-independent linear projection where voxels on a viewing ray contribute equally regardless of distance [S3, S4]. It is distinguished from its sibling [Domain-transform methods](28-domain-transform-methods.md) by its specific reliance on frequency-domain slicing to achieve computational efficiency for parallel projections [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume $f(\mathbf{x})$ sampled on a 3D integer lattice of size $N \times N \times N$ [S1, S3].
*   **Output:** A 2D projection image $I$ for a given viewing direction [S1, S2].

**Data Structures:**
*   **3D Spectrum:** An array of $N^3$ complex coefficients (or real coefficients if using a Hartley transform). Requires 32-bit floating point precision for accuracy, increasing memory by a factor of 4 compared to 8-bit input volume [S1]. If using standard complex arithmetic, it requires 16 bytes per voxel (pair of 64-bit doubles) [S2, S4, S5].
*   **2D Slice:** An array of $N^2$ coefficients extracted from the 3D spectrum [S1].

**Parameters:**
*   $N$: Side length of the cubical volume. Assumed to be an integer power of two for FFT efficiency [S1].
*   $V$: Normalized viewing direction vector. Defines the orientation of the slice in frequency space. Note: $V$ points away from the eye [S1].
*   $W$: Nonzero width of the interpolating filter used during slice extraction. Typical value: 4 (a $4^3$ tap filter) [S1].
*   $k_2$: Factor by which volume width is increased via zero-padding. Typical value: $k_2 = 4/3$ (16% margin) [S1].
*   $M$: Resolution of the projection slice ($M \times M$). Must be high enough to prevent aliasing [S3].

**Steps:**

1.  **Preprocessing (Frequency Domain Transformation):**
    Compute the 3D discrete Fourier transform (DFT) of the scalar volume $f(\mathbf{x})$ [S1]. For real volumes, a Discrete Hartley Transform (DHT) may be substituted to halve memory and time [S1, S3].
    *   *Vignetting Correction:* To cope with vignetting (attenuation of the periphery due to non-ideal filter response), premultiply the volume data $f(\mathbf{x})$ by the reciprocal of the filter response $P_m(\mathbf{x}) = 1/h(\mathbf{x})$ before the forward transformation [S2, S4]. Here $H(\mathbf{s})$ is the interpolation filter in frequency space, and $h(\mathbf{x})$ is its spatial response [S2, S4].
    *   *Zero Padding:* Surround the data with a margin of zeros to avoid ghosting artifacts. A typical margin is 16% ($k_2=4/3$) [S1].

2.  **Slice Extraction (Per View):**
    For each view defined by direction $V$:
    *   Define an extraction plane passing through the origin of the frequency domain and perpendicular to $V$ [S1, S2].
    *   Interpolate the discrete spectrum $F(\mathbf{s})$ along this plane using a filter $H(\mathbf{s})$ [S2]. Since the plane does not typically pass through integer lattice points, interpolation from nearby samples is required [S1, S2, S4].
    *   Resample the interpolated spectrum to obtain a 2D spectrum [S2, S4, S5].

3.  **Inverse Transform:**
    Compute the inverse 2D Fourier transform (or inverse FHT) of the extracted 2D slice to generate the final projection image $I$ [S1, S2, S4, S5].

**Formulas:**
The Fourier projection-slice theorem states that the 2D image obtained by taking line integrals of the volume along rays perpendicular to the image plane is a Fourier transform pair with the 2D spectrum obtained by extracting a slice from the Fourier transform of the volume along a plane which includes the origin and is parallel to the image plane [S4].

Mathematically, the projection operator $\Pi_V$ is defined as [S1]:
$$ I = \Pi_V(f) = \mathcal{F}_2^{-1} \{ \mathcal{F}_3 \{ f \} \delta_V \} $$
where $\mathcal{F}_3$ is the 3D Fourier transform, $\mathcal{F}_2^{-1}$ is the 2D inverse Fourier transform, and $\delta_V$ restricts the spectrum to a plane passing through the origin and perpendicular to viewing direction $V$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fourier_projection_slice_rendering(volume, viewing_direction):
    """
    volume: 3D list of lists of lists of floats, shape (N, N, N).
    viewing_direction: list of 3 floats, normalized vector V pointing away from eye.
    Returns: 2D list of lists of floats, shape (M, M), the projection image.
    """
    N = len(volume)
    k2 = 4.0 / 3.0
    margin = int(N * (k2 - 1) / 2)
    padded_size = int(N * k2)
    M = N

    # 1. Preprocessing: Zero Padding
    padded_volume = [[[0.0] * padded_size for _ in range(padded_size)] for _ in range(padded_size)]
    for i in range(N):
        for j in range(N):
            for k in range(N):
                padded_volume[i + margin][j + margin][k + margin] = volume[i][j][k]

    # 2. Vignetting Correction
    # Premultiply by reciprocal of filter spatial response h(x) [S2, S4]
    corrected_volume = [[[0.0] * padded_size for _ in range(padded_size)] for _ in range(padded_size)]
    for i in range(padded_size):
        for j in range(padded_size):
            for k in range(padded_size):
                x, y, z = i - margin, j - margin, k - margin
                h_val = spatial_filter_response(x, y, z)
                corrected_volume[i][j][k] = padded_volume[i][j][k] / max(h_val, 1e-8)

    # 3. Forward 3D FFT
    spectrum = fft3d(corrected_volume)

    # 4. Slice Extraction
    # Generate orthonormal basis (U, T) for the plane perpendicular to V
    U, T = generate_orthonormal_basis(viewing_direction)
    center = padded_size // 2
    slice_2d = [[[0.0] * M for _ in range(M)] for _ in range(2)]

    for i in range(M):
        for j in range(M):
            # Map 2D pixel indices to 3D frequency coordinates
            sx = center + U[0] * (i - M / 2) + T[0] * (j - M / 2)
            sy = center + U[1] * (i - M / 2) + T[1] * (j - M / 2)
            sz = center + U[2] * (i - M / 2) + T[2] * (j - M / 2)
            # Interpolate complex spectrum at (sx, sy, sz)
            slice_2d[0][i][j], slice_2d[1][i][j] = interpolate_3d_complex(spectrum, sx, sy, sz)

    # 5. Inverse 2D FFT
    projection = ifft2d(slice_2d)
    return projection

def spatial_filter_response(x, y, z):
    # [G] Returns the spatial response h(x) of the interpolation filter.
    # Typically a separable kernel (e.g., 4^3 tap tricubic spline).
    return 1.0

def generate_orthonormal_basis(V):
    # [G] Standard vector math to find two vectors orthogonal to V
    arbitrary = [1.0, 0.0, 0.0] if abs(V[0]) < 0.9 else [0.0, 1.0, 0.0]
    u = cross(V, arbitrary)
    u = normalize(u)
    t = cross(V, u)
    t = normalize(t)
    return u, t

def interpolate_3d_complex(grid, x, y, z):
    # [G] Standard 3D grid interpolation (e.g., tricubic spline) for complex values.
    return 0.0, 0.0

def fft3d(data):
    # [G] Standard radix-2 3D FFT algorithm
    pass

def ifft2d(data):
    # [G] Standard radix-2 2D IFFT algorithm
    pass
```

## Subtopics

### [Implementation Details for Fourier Volume Rendering](846-implementation-details-for-fourier-volume-rendering.md)
This variant focuses on practical optimizations for the base algorithm. Slices are extracted using a tricubic interpolating spline. A key optimization is replacing the FFT with a Fast Hartley Transform (FHT), which halves per-view rendering time and memory requirements for real-valued data. Further speedups can be achieved by optimizing the filter and trimming zero padding.

### [Depth Cueing via Linear Attenuation](966-depth-cueing-via-linear-attenuation.md)
A shading model that approximates exponential attenuation with locally varying coefficients by linear attenuation with a constant coefficient, allowing the integrand to be expressed as a Fourier projection. This restores depth perception to the otherwise occlusion-free X-ray projections.

## Operating characteristics
**Time Complexity:**
*   **Preprocessing:** $O(N^3 \log N)$ operations to compute the 3D spectrum [S1, S3].
*   **Per-View Rendering:** $O(N^2 \log N)$ operations for the inverse transform, plus $O(W^3 N^2)$ for slice extraction where $W$ is the filter width [S1]. The overall cost is dominated by the 2D inverse FFT, $O(n^2 \log n)$ [S4].
*   **Interpolation Cost:** High interpolation cost is a bottleneck [S2, S4]. Interpolation is $O(n^2)$ with large constant factors, often taking more time than the FFT for practical data sizes (128^3 or 256^3) [S2, S4].

**Memory Complexity:**
*   Requires 16 bytes per voxel (pair of 64-bit doubles) for complex arithmetic, compared to 1 byte in spatial domain algorithms [S2, S4, S5].
*   Zero padding increases memory by a factor of $k_2^3$ [S1].

**Performance:**
*   For $N=256$, digital compositing requires 16M operations, while FVR requires 512K operations (speedup >10x) theoretically [S1].
*   In practice, interpolation overhead reduces this gain [S2, S4]. GPU acceleration can achieve highly interactive framerates (factor of 17 speedup reported) [S3].

**Strengths and Limitations:**
*   **Strengths:** Fast generation of X-ray-like projections; suitable for MR angiography and attenuation-only renderings where speed is prioritized [S1, S3].
*   **Limitations:** No occlusion or depth cues (unless added via shading models); vignetting artifacts; ghosting artifacts from interpolation; high memory usage; only supports parallel projection [S1, S2, S3, S4].

## Background
The method is based on the Fourier projection-slice theorem, which states that the inverse transform of a slice extracted from the frequency domain representation of a volume yields a projection of the volume in a direction perpendicular to the slice [S1]. It was independently proposed for image synthesis by Dunne and Malzbender [S1]. Marc Levoy published the foundational paper "Volume Rendering using the Fourier Projection-Slice Theorem" in 1992 [S1]. Takashi Totsuka and Marc Levoy extended this in 1993 with "Frequency Domain Volume Rendering," introducing shading models in the frequency domain [S2, S4]. Later, Viola et al. accelerated the method using GPU-based pipelines [S3].

## Sources
[S1] Volume Rendering using the Fourier Projection-Slice Theorem, https://graphics.stanford.edu/papers/fourier/fourier.pdf, fetched 2026-10-27
[S2] Frequency Domain Volume Rendering - Stanford University, https://graphics.stanford.edu/papers/fvr/totsuka-fvr-sig93.pdf, fetched 2026-10-27
[S3] GPU-based Frequency Domain Volume Rendering - TU Wien, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27
[S4] FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27
[S5] Frequency Domain Volume Rendering, https://graphics.stanford.edu/papers/fvr/fvr.pdf, fetched 2026-10-27
[S6] Frequency Domain Volume Rendering, https://graphics.stanford.edu/papers/fvr/fvr.pdf, fetched 2026-10-27
[S7] Fourier Projection-Slice Theorem in Volume Rendering, https://www.scribd.com/document/796298963/Marc-Levoy-Volume-rendering-using-the-Fourier-projection-slice-theorem, fetched 2026-10-27 [S7, unofficial]
