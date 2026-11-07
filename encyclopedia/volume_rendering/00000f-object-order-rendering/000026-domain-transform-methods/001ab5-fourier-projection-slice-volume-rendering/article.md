# Fourier Projection-Slice Volume Rendering

Also called: FT-based DVR, Fourier Domain Volume Rendering, Fourier projection rendering

## Overview
Fourier Projection-Slice Volume Rendering (FDVR) is an indirect volume rendering technique that generates attenuation-only, X-ray-like projections of a 3D scalar volume by exploiting the Fourier projection-slice theorem [S1, S3, S4, S5, unofficial][S8][S13, S15, unofficial]. It solves the problem of high computational complexity in spatial domain ray-casting by transforming the volume into the frequency domain once, then generating views via 2D resampling and inverse transforms, reducing per-view complexity from $O(N^3)$ to $O(N^2 \log N)$ [S1, S3, S4, S6, unofficial][S7, S8][S12, S13, unofficial]. Unlike [Ray Casting](52-ray-casting.md) or [Digital Compositing](53-digital-compositing.md), FDVR produces line-integral projections without hidden surface effects or direct transparency control, making it suitable for interactive X-ray simulation and medical imaging where occlusion is not required [S3, S4, S10, S13, unofficial].

## How it works
The method follows the three-stage pipeline of domain-transform methods: Global Transformation, Slice Extraction, and Inverse Transformation [S1, S824-fourier-projection-slice-volume-rendering.md, S895-hartley-transform-for-memory-reduction.md, S5396-dct-based-volume-rendering.md, unofficial].

### Inputs and Outputs
*   **Input:** A 3D scalar volume $f(x, y, z)$ of size $N \times N \times N$ (typically $N$ is a power of two for FFT efficiency) [S4, S13, S15, unofficial].
*   **Output:** A 2D projection image $I(u, v)$ of resolution $M \times M$ (typically $M=N$) representing the density integral along viewing rays [S3, S4, S16, unofficial].
*   **Parameters:**
    *   $V$: Normalized viewing direction vector (points away from the eye) [S4, S16, unofficial].
    *   $W$: Nonzero width of the interpolating filter used during slice extraction (typical value: 4) [S4, S16, unofficial].
    *   $k_2$: Zero-padding factor (typical value: $4/3$, i.e., 16% margin) [S4, unofficial].

### Data Structures
1.  **Input Volume:** Scalar array, typically 8 bits per voxel [S16, unofficial].
2.  **3D Spectrum:** Array of $N^3$ complex coefficients (or real if using Hartley Transform). Requires 32-bit floating point precision (16 bytes per voxel if complex double precision is used, or 8 bytes if complex float) [S4, S5, S11, S16, unofficial].
3.  **2D Slice:** Array of $M^2$ complex coefficients extracted from the 3D spectrum [S4, S16, unofficial].

### Steps

1.  **Preprocessing (Global Transformation):**
    *   **Zero-Padding:** To mitigate ghosting artifacts caused by periodic boundary conditions of the DFT, zero-pad the input volume $f(x)$ by a factor of $k_2$. This creates a margin of zeros around the data [S4, S16, unofficial].
    *   **Vignetting Correction:** Premultiply the padded volume data $f(x)$ by the reciprocal of the interpolation filter response $P_m(x) = 1/H(x)$ before the forward transform. This ensures that the filter response $H(s)$ and $P_m(s)$ cancel during interpolation, preventing attenuation of the image periphery [S5, S11, S12, S14, unofficial].
    *   **3D FFT:** Compute the 3D Discrete Fourier Transform (DFT) of the corrected volume to obtain the frequency domain representation $F(s)$ [S4, S5, unofficial][S7][S13, S14, unofficial]. If memory efficiency is prioritized, a Discrete Hartley Transform (DHT) may be used, though it requires a correction pass as it is not separable [S4, S15, unofficial].
    *   **Origin Shift:** Rearrange the spectrum so that the origin (zero frequency) is at the center of the 3D array [S13, unofficial].

2.  **Per-View Rendering (Slice Extraction):**
    *   **Define Slice Plane:** Define a 2D plane passing through the origin of the frequency domain and perpendicular to the viewing direction $V$ [S4, S16, unofficial].
    *   **Interpolation:** Since the arbitrary plane does not align with the integer lattice of the 3D spectrum, interpolate the values of $F(s)$ along the plane using a filter $H(s)$ (e.g., tricubic spline) [S5, S16, unofficial]. This step has complexity $O(W^3 M^2)$ [S4, unofficial].
    *   **Resampling:** Resample the interpolated values onto a regular 2D grid to form the 2D spectrum $I(u, v)$ [S5, S12, unofficial].

3.  **Inverse Transformation:**
    *   **2D IFFT:** Perform a 2D Inverse Fast Fourier Transform (IFFT) on the 2D slice $I(u, v)$ to obtain the spatial domain projection image [S3, S5, S6, S13, S15, unofficial].
    *   **Scaling:** Rescale the resulting intensity values to the valid display range [S15, unofficial].

### Formulas
The 3D Fourier transform of the spatial density function $f(x, y, z)$ is:
$$F(\omega_1, \omega_2, \omega_3) = \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} f(x, y, z) e^{-i2\pi(x\omega_1 + y\omega_2 + z\omega_3)} dx dy dz \quad [S3, unofficial]$$

The Fourier projection-slice theorem states that the 2D Fourier transform of the projection is equal to a central slice of the 3D Fourier transform:
$$I(u, v) = F(\omega_1(u, v), \omega_2(u, v), \omega_3(u, v)) \quad [S3, S15, unofficial]$$

The rendering operator $\Pi_V$ is defined as:
$$\Pi_V(f) = F_2^{-1} \{ F_3 \{ f \} \delta_V^{-1} \} \quad [S4, S16, unofficial]$$
where $\delta_V^{-1}$ restricts the spectrum to a plane passing through the origin and perpendicular to viewing direction $V$, and $F_N, F_N^{-1}$ are forward and inverse N-dimensional Fourier transforms.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fourier_projection_slice_rendering(volume, view_direction, filter_width=4, padding_factor=4/3):
    """
    Renders an X-ray-like projection of a 3D volume using the Fourier Projection-Slice Theorem.
    
    Args:
        volume: 3D numpy array of shape (N, N, N), scalar values.
        view_direction: 3D numpy array, normalized vector pointing away from the eye.
        filter_width: Width of the interpolation filter (W).
        padding_factor: Factor by which to increase volume size (k2).
        
    Returns:
        2D numpy array of shape (M, M), the projection image.
    """
    N = volume.shape[0]
    
    # Step 1: Preprocessing
    # Calculate padded size
    padded_size = int(N * padding_factor)
    
    # Zero-pad the volume
    padded_volume = np.zeros((padded_size, padded_size, padded_size), dtype=volume.dtype)
    # Center the original volume in the padded volume
    start_idx = (padded_size - N) // 2
    padded_volume[start_idx:start_idx+N, start_idx:start_idx+N, start_idx:start_idx+N] = volume
    
    # Vignetting Correction: Premultiply by reciprocal of filter response
    # Note: In practice, P_m(x) is the spatial domain representation of 1/H(s).
    # For a sinc-like ideal filter, this is complex. Approximations or specific
    # filter kernels (e.g., Lanczos) are used. Here we assume a function get_vignetting_correction()
    # returns the correction mask.
    vignetting_mask = get_vignetting_correction(padded_size, filter_width)
    corrected_volume = padded_volume * vignetting_mask
    
    # 3D FFT
    # Shift origin to center before FFT if using libraries that expect origin at corner,
    # or shift after. Standard FFT assumes origin at (0,0,0).
    # We compute FFT of the centered data.
    spectrum = np.fft.fftn(corrected_volume)
    
    # Shift zero frequency to center for easier slicing logic
    spectrum = np.fft.fftshift(spectrum)
    
    # Step 2: Slice Extraction
    # Define the 2D plane perpendicular to view_direction passing through origin.
    # We need to map 2D grid coordinates (u, v) to 3D frequency coordinates.
    M = N # Output resolution matches input for simplicity, or specified separately
    u_coords = np.linspace(-0.5, 0.5, M)
    v_coords = np.linspace(-0.5, 0.5, M)
    U, V = np.meshgrid(u_coords, v_coords)
    
    # Construct 3D coordinates on the slice plane
    # Basis vectors orthogonal to view_direction
    # This part requires constructing an orthonormal basis for the plane.
    # Let V_dir = view_direction.
    # Find two vectors u_vec, v_vec orthogonal to V_dir and each other.
    u_vec, v_vec = get_orthonormal_basis(view_direction)
    
    # 3D coordinates in frequency space (normalized)
    # The slice is at frequency 0 along the view direction
    freq_x = U * u_vec[0] + V * v_vec[0]
    freq_y = U * u_vec[1] + V * v_vec[1]
    freq_z = U * u_vec[2] + V * v_vec[2]
    
    # Interpolate the 3D spectrum at these coordinates
    # Using scipy.ndimage.map_coordinates or similar tricubic interpolation
    # Coordinates must be in index space, not normalized frequency
    # Map normalized freq [-0.5, 0.5] to array indices [0, padded_size-1]
    # Center is at padded_size/2
    
    indices_x = (freq_x + 0.5) * padded_size
    indices_y = (freq_y + 0.5) * padded_size
    indices_z = (freq_z + 0.5) * padded_size
    
    # Create coordinate arrays for interpolation
    coords = np.array([indices_z, indices_y, indices_x]) # Note: FFTN order is z,y,x or similar depending on implementation
    
    # Perform interpolation
    # grid_order=3 for cubic interpolation
    slice_spectrum = scipy.ndimage.map_coordinates(spectrum, coords, order=3, mode='constant', cval=0.0)
    
    # Step 3: Inverse Transformation
    # 2D IFFT
    # The result of map_coordinates is already on a 2D grid (M, M)
    projection_image = np.fft.ifft2(slice_spectrum)
    
    # Take real part (imaginary part should be near zero for real input)
    projection_image = np.real(projection_image)
    
    # Scale and return
    return projection_image

def get_orthonormal_basis(view_direction):
    """
    Returns two orthonormal vectors u_vec, v_vec orthogonal to view_direction.
    """
    v = view_direction / np.linalg.norm(view_direction)
    # Pick an arbitrary vector not parallel to v
    if abs(v[0]) < 0.9:
        arbitrary = np.array([1, 0, 0])
    else:
        arbitrary = np.array([0, 1, 0])
    
    u_vec = np.cross(v, arbitrary)
    u_vec = u_vec / np.linalg.norm(u_vec)
    v_vec = np.cross(v, u_vec)
    v_vec = v_vec / np.linalg.norm(v_vec)
    
    return u_vec, v_vec

def get_vignetting_correction(size, filter_width):
    """
    Placeholder for vignetting correction mask generation.
    In practice, this involves the inverse Fourier transform of 1/H(s).
    """
    # Not given in the sources. Implementation depends on the specific interpolation filter H(s).
    return np.ones((size, size, size))
```

### Edge Cases and Pitfalls
*   **Ghosting Artifacts:** Convolution with a finite-support interpolation filter causes incomplete suppression of periodic replicas in the spatial domain, appearing as ghosting [S4, S5, S11, S16, unofficial]. Mitigation includes zero-padding (increasing memory by $k_2^3$) or using wider interpolation filters (increasing per-view time by $k_1^3$) [S4, unofficial].
*   **Vignetting:** Unless an ideal lowpass filter is used, the filter response attenuates the periphery of the volume. Mitigation involves premultiplying the volume by the reciprocal of the filter response before the forward transform [S5, S11, S12, S14, unofficial].
*   **Aliasing:** The output image resolution must meet the Nyquist criterion. If the projection slice resolution is too low, overlapping copies of rendered data occur [S2, S3, S10, S13, unofficial].
*   **Memory Cost:** The frequency domain representation requires 16 bytes per voxel (complex double precision) or 8 bytes (complex float), significantly higher than the 1 byte per voxel in the spatial domain [S5, S11, S12, unofficial].
*   **Lack of Occlusion:** The method produces line integrals where voxels on a viewing ray contribute equally regardless of distance, resulting in no occlusion or hidden surface removal [S3, S4, S5, S11, S12, S15, unofficial].

## Subtopics

### [Implementation Details for Fourier Volume Rendering](846-implementation-details-for-fourier-volume-rendering.md)
This article details the specific algorithmic steps, data structures, and parameter choices required to implement Fourier Volume Rendering (FVR) as described in the foundational literature. It solves the practical engineering problems of spectral leakage (ghosting), numerical stability in frequency-domain operations, and performance optimization for the 3D FFT and slice extraction pipeline. It sets itself apart from generic descriptions by providing exact margin sizes, interpolation kernels, and stability thresholds for degenerate cases, building on [Fourier Projection-Slice Volume Rendering](824-fourier-projection-slice-volume-rendering.md).

### [Depth Cueing via Linear Attenuation](966-depth-cueing-via-linear-attenuation.md)
This method approximates the perceptual depth cues of exponential attenuation in volume rendering by replacing the variable absorption coefficient with a linear function of position, enabling the integrand to be evaluated as a Fourier projection. It sets itself apart from standard exponential attenuation models by allowing efficient computation via the Fourier Projection-Slice Theorem.

### [Fourier Volume Rendering](6050-fourier-volume-rendering.md)
Fourier Volume Rendering (FVR) is a direct volume rendering technique that computes X-ray-like attenuation-only projections of a three-dimensional dataset by exploiting the Fourier projection-slice theorem. It reduces the per-view computational complexity from $O(N^3)$ to $O(N^2 \log N)$ by computing a 3D frequency transform once during preprocessing, then generating views via 2D resampling and inverse transforms. Unlike [Ray Casting](52-ray-casting.md) or [Digital Compositing](53-digital-compositing.md), FVR produces line-integral projections without hidden surface effects or direct transparency control.

### Fourier Projection-Slice Integration over Irregular Domains
A technique to compute integrals of functions over irregular spatial domains by decomposing the domain into spatially disjoint intervals and summing their Fourier projection-slice integrals [S4, unofficial]. It decomposes the ray segment connecting two interior points into spatially disjoint intervals, computes each interval integral using the projection-slice theorem, and applies recursive subdivision of volumes into octants [S4, unofficial]. Time complexity is $O(N^2 \log^2 N)$ for the octant subdivision algorithm [S4, unofficial]. Applications include rendering spatially clipped volumes for medical visualization, fast calculation of inter-element attenuation for zonal radiosity algorithms, and approximate visibility determination for geometrically defined scenes [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N^3 \log N)$ for the 3D FFT [S4, unofficial][S7][S10, S13, S15, unofficial]. Per-view rendering is $O(M^2 \log M)$ for the 2D IFFT plus $O(W^3 M^2)$ for slice extraction, where $W$ is the filter width [S4, unofficial][S7, S8][S10, S13, unofficial]. For $N=256$, preprocessing is ~16M operations, while per-view is significantly faster than spatial domain $O(N^3)$ [S4, S16, unofficial].
*   **Memory Complexity:** Requires $O(N^3)$ memory for the 3D spectrum. Complex arithmetic doubles the storage requirement compared to real-valued spatial data (16 bytes/voxel for double precision complex vs 1 byte/voxel for 8-bit spatial) [S4, S5, S11, S12, unofficial].
*   **Performance:** Achieves interactive framerates on GPU hardware for volumes up to $256^3$ [S10, S13, S15, unofficial]. The interpolation step is often the bottleneck due to high constant factors [S5, S11, unofficial].
*   **Strengths:** Fast generation of attenuation-only projections; suitable for X-ray simulation and MR angiography; allows filtering (lowpass, highpass) with little overhead [S3, S4, unofficial][S8][S15, unofficial].
*   **Limitations:** No occlusion or hidden surface removal; limited to parallel projection; cannot model complex optical phenomena like emission or scattering directly; high memory usage; sensitive to interpolation artifacts (ghosting) [S3, S4, S5, unofficial][S7][S11, S15, unofficial].
*   **When to Use:** When speed is prioritized over occlusion, such as in medical X-ray simulation or quick previews [S4, S10, S12, unofficial].
*   **When Not to Use:** When realistic depth cues, occlusion, or complex lighting models are required [S3, S4, S15, unofficial].

## Background
FDVR was first applied by Dunne, Malzbender, and Napel et al. in the early 1990s for attenuation-only rendering and MR angiography [S4, unofficial][S7, S9]. Levoy and Totsuka independently generalized the method to include shading and depth cueing in the frequency domain [S5, unofficial][S9][S15, unofficial]. The method was later implemented on graphics hardware (GPU) by Jansen et al. and Viola et al. to achieve interactive framerates [S9][S10, S13, S15, unofficial]. The Fast Fourier Transform (FFT) was originally introduced by Cooley, and the Fast Hartley Transform (FHT) was proposed by Bracewell and Hao as an alternative [S15, unofficial].

## Sources
[S1] Direct Volume Rendering Direct Volume Rendering. What is volume rendering?. Accumulate information along 1 dimension line through volume. Volume rendering vs. isosurfaces. No intermediate geometry No , https://www.slideserve.com/lenci/direct-volume-rendering, fetched 2026-10-25 (unofficial)
[S2] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S3] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S4] PDF Volume Rendering using the Fourier Projection-Slice Theorem, https://graphics.stanford.edu/papers/fourier/fourier.pdf, fetched 2026-10-27 (unofficial)
[S5] PDF Frequency Domain Volume Rendering - Stanford University, https://graphics.stanford.edu/papers/fvr/totsuka-fvr-sig93.pdf, fetched 2026-10-27 (unofficial)
[S6] Fourier Volume Rendering, https://core.ac.uk/works/16671645, fetched 2026-10-27 (unofficial)
[S7] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S8] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S9] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S10] PDF GPU-based Frequency Domain Volume Rendering - TU Wien, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27 (unofficial)
[S11] PDF FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27 (unofficial)
[S12] Frequency Domain Volume Rendering - studylib.net, https://studylib.net/doc/18162906/frequency-domain-volume-rendering, fetched 2026-10-27 (unofficial)
[S13] PDF Hardware-Accelerated Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/viola-2004-har2/viola-2004-har2-Paper.pdf, fetched 2026-10-27 (unofficial)
[S14] Frequency Domain Volume Rendering, https://graphics.stanford.edu/papers/fvr/fvr.pdf, fetched 2026-10-27 (unofficial)
[S15] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S16] Fourier Projection-Slice Theorem in Volume Rendering, https://www.scribd.com/document/796298963/Marc-Levoy-Volume-rendering-using-the-Fourier-projection-slice-theorem, fetched 2026-10-27 (unofficial)
