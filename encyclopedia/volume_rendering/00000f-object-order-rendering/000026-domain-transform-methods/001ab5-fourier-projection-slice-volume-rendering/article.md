# Fourier Projection-Slice Volume Rendering

Also called: FDVR, FT-based DVR, Fourier Domain Volume Rendering, Fourier projection rendering

## Overview
Fourier Projection-Slice Volume Rendering (FDVR) is an indirect volume rendering technique that generates attenuation-only images (X-ray-like projections) by exploiting the Fourier projection-slice theorem [S1, S3, S4, S13, unofficial]. It solves the problem of high computational complexity in spatial domain ray-casting by transforming the volume into the frequency domain once, then generating views via 2D resampling and inverse transforms, reducing per-view complexity from $O(N^3)$ to $O(N^2 \log N)$ [S3, S13, unofficial]. Unlike [Ray Casting](52-ray-casting.md) or [Digital Compositing](53-digital-compositing.md), FDVR produces order-independent linear projections without hidden surface effects or transparency control [S3, S9, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume $f(x, y, z)$ of size $N \times N \times N$, sampled on a regular grid. $N$ is typically an integer power of two for FFT efficiency [S3, S12, unofficial].
*   **Output:** A 2D projection image $I(u, v)$ representing the line integral of density along viewing rays [S2, S14, unofficial].

**Data Structures:**
*   **3D Spectrum:** An array of $N^3$ complex coefficients (or real coefficients if using Hartley Transform). Requires 32-bit floating point precision for accuracy, increasing memory by a factor of 4 compared to 8-bit input volume [S3, S15, unofficial]. If using double precision, 16 bytes per voxel are required [S4, S11, unofficial].
*   **2D Slice:** An array of $N^2$ coefficients extracted from the 3D spectrum [S3, S15, unofficial].
*   **Interpolation Filter:** A filter $H(s)$ with nonzero width $W$ (typically $W=4$) used to resample the 3D spectrum [S3, S15, unofficial].

**Preprocessing (One-time):**
1.  **Origin Shift:** Rearrange spatial data to move the origin from corner $[0,0,0]$ to the center (wrap-around operation) [S12, unofficial].
2.  **Vignetting Correction (Optional but recommended):** Premultiply the volume data $f(x)$ by the reciprocal of the filter response $p_m(x) = 1/H(x)$ before the forward transformation to correct for attenuation at the periphery [S10, S11, S13, unofficial].
3.  **Zero Padding (Optional):** To mitigate ghosting artifacts, surround the data with a margin of zeros. If the volume width is increased by a factor $k_2$ (e.g., $k_2 = 4/3$, a 16% margin), memory and preprocessing costs increase by $k_2^3$ [S3, S15, unofficial].
4.  **3D Transform:** Compute the 3D Discrete Fourier Transform (DFT) or Discrete Hartley Transform (DHT) of the volume $f(x)$ to obtain the frequency domain representation $F(s)$ [S3, S4, S13, unofficial].
    $$F(\omega_1, \omega_2, \omega_3) = \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} f(x, y, z) e^{-i2\pi(x\omega_1 + y\omega_2 + z\omega_3)} dx dy dz$$
    [S2, S14, unofficial]

**Per-View Rendering:**
1.  **Define View:** Let $V$ be the normalized viewing direction vector (pointing away from the eye) [S3, unofficial].
2.  **Slice Extraction:** Extract a 2D slice from the 3D spectrum $F(s)$ along a plane passing through the origin and perpendicular to $V$ [S3, S13, unofficial]. Since the slicing plane rarely coincides with the integer lattice of the 3D spectrum, interpolation is required.
    *   Interpolate $F(s)$ along the extraction plane using filter $H(s)$ [S4, S13, unofficial].
    *   Resample the interpolated spectrum to obtain a 2D spectrum $I(u, v)$ [S4, S13, unofficial].
    *   Cost: $O(W^3 N^2)$, where $W$ is the filter width [S3, unofficial].
3.  **Inverse Transform:** Compute the 2D Inverse Fast Fourier Transform (IFFT) or Inverse Fast Hartley Transform (IFHT) of the 2D slice $I(u, v)$ to generate the spatial domain projection image [S2, S3, S13, unofficial].
    $$I(u, v) = \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} \int_{-\infty}^{\infty} f(x, y, z) e^{-i2\pi(x\omega_1(u, v) + y\omega_2(u, v) + z\omega_3(u, v))} dx dy dz$$
    [S2, unofficial]
4.  **Post-processing:** Rescale intensity values to a valid range and display [S14, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fourier_projection_slice_rendering(volume, view_direction):
    """
    volume: 3D numpy array of shape (N, N, N), real-valued scalar field.
    view_direction: 3D numpy array, normalized vector V pointing away from eye.
    Returns: 2D numpy array of shape (N, N), the projection image.
    """
    N = volume.shape[0]
    
    # --- Preprocessing (Assumed done once, stored in global state) ---
    # 1. Shift origin to center
    volume_centered = fftshift(volume)
    
    # 2. Vignetting correction (if enabled)
    # p_m(x) = 1 / H(x). In practice, H(x) is the spatial response of the interpolation filter.
    # For simplicity, we assume this is pre-computed or negligible for basic implementation.
    # volume_corrected = volume_centered * vignetting_correction_mask
    
    # 3. 3D FFT
    # Note: Real input allows use of RFFT for efficiency, but standard FFT is shown for clarity.
    spectrum = fftn(volume_centered)
    
    # --- Per-View Rendering ---
    
    # 1. Define the slicing plane in frequency domain
    # The plane passes through the origin (center of spectrum) and is perpendicular to V.
    # We need to map 2D image coordinates (u, v) to 3D frequency coordinates (w1, w2, w3).
    
    # Create 2D grid for the output image
    u, v = np.meshgrid(np.arange(-N/2, N/2), np.arange(-N/2, N/2))
    
    # Define two orthogonal vectors U_vec and V_vec in the frequency domain
    # that span the plane perpendicular to view_direction.
    # Let V_vec = view_direction.
    # Choose an arbitrary vector not parallel to V, e.g., [1, 0, 0] unless V is [1,0,0].
    arbitrary = np.array([1.0, 0.0, 0.0])
    if np.abs(np.dot(view_direction, arbitrary)) > 0.9:
        arbitrary = np.array([0.0, 1.0, 0.0])
    
    U_vec = np.cross(view_direction, arbitrary)
    U_vec = U_vec / np.linalg.norm(U_vec)
    V_vec = np.cross(view_direction, U_vec)
    V_vec = V_vec / np.linalg.norm(V_vec)
    
    # Map 2D indices to 3D frequency coordinates
    # The resolution of the slice should match the Nyquist criterion.
    # For simplicity, we assume N samples cover the relevant frequency range.
    # Frequency coordinates are scaled by 1/N relative to array indices.
    freq_u = u / N
    freq_v = v / N
    
    # 3D coordinates on the slice
    w1 = freq_u * U_vec[0] + freq_v * V_vec[0]
    w2 = freq_u * U_vec[1] + freq_v * V_vec[1]
    w3 = freq_u * U_vec[2] + freq_v * V_vec[2]
    
    # 2. Interpolate and Resample
    # Since w1, w2, w3 are continuous, we must interpolate from the discrete spectrum.
    # This is the most expensive step.
    # Using trilinear or tricubic interpolation from the 3D spectrum array.
    # Note: The spectrum array indices correspond to integer frequencies.
    # We must map continuous frequencies to array indices.
    # Index = (Frequency * N) + N/2 (to handle negative frequencies and 0-centering)
    
    idx1 = (w1 * N) + N/2
    idx2 = (w2 * N) + N/2
    idx3 = (w3 * N) + N/2
    
    # Perform interpolation (e.g., trilinear)
    # This requires a helper function for multi-dimensional interpolation
    slice_spectrum = interpolate_3d(spectrum, idx1, idx2, idx3)
    
    # 3. Inverse 2D FFT
    # The result of the interpolation is the 2D spectrum I(u,v)
    # Apply Inverse FFT to get the spatial projection
    projection = np.real(ifft2(slice_spectrum))
    
    return projection

def interpolate_3d(data, x, y, z):
    """
    Trilinear interpolation of 3D array 'data' at continuous coordinates x, y, z.
    Assumes data is centered and periodic (wrapped).
    """
    # Handle boundary conditions by wrapping
    x = x % data.shape[0]
    y = y % data.shape[1]
    z = z % data.shape[2]
    
    x0 = np.floor(x).astype(int)
    y0 = np.floor(y).astype(int)
    z0 = np.floor(z).astype(int)
    
    x1 = (x0 + 1) % data.shape[0]
    y1 = (y0 + 1) % data.shape[1]
    z1 = (z0 + 1) % data.shape[2]
    
    dx = x - x0
    dy = y - y0
    dz = z - z0
    
    # Trilinear interpolation formula
    val = (1-dx)*(1-dy)*(1-dz)*data[x0, y0, z0] + \
          dx*(1-dy)*(1-dz)*data[x1, y0, z0] + \
          (1-dx)*dy*(1-dz)*data[x0, y1, z0] + \
          dx*dy*(1-dz)*data[x1, y1, z0] + \
          (1-dx)*(1-dy)*dz*data[x0, y0, z1] + \
          dx*(1-dy)*dz*data[x1, y0, z1] + \
          (1-dx)*dy*dz*data[x0, y1, z1] + \
          dx*dy*dz*data[x1, y1, z1]
          
    return val
```

**Edge Cases and Pitfalls:**
*   **Ghosting Artifacts:** Convolution with a finite support interpolation filter causes incomplete suppression of periodic replicas in the spatial domain, appearing as ghosting [S3, S4, S13, unofficial]. Solutions include using a wider interpolation filter (increasing cost by $k_1^3$) or zero-padding the input volume (increasing memory by $k_2^3$) [S3, unofficial].
*   **Vignetting:** Unless $H(s)$ is an ideal lowpass filter, its response attenuates the periphery of the volume and projected image [S10, S11, unofficial]. Corrected by premultiplying $f(x)$ by $1/H(x)$ before the forward FFT [S10, S11, S13, unofficial].
*   **Aliasing:** If the projection slice resolution is too low, overlapping copies of rendered data occur [S9, S12, unofficial]. Ensure slice resolution meets Nyquist criterion or reduce resampling area to low frequencies (results in blurry preview) [S12, unofficial].
*   **Hardware Limits:** 3D FFT requires two copies of data in GPU memory, limiting rendered data size to half of available graphics memory [S12, unofficial]. Non-power-of-two resolutions require zero-padding [S12, unofficial].

## Subtopics

### [Implementation Details for Fourier Volume Rendering](846-implementation-details-for-fourier-volume-rendering.md)
This article details the specific algorithmic steps, data structures, and parameter choices required to implement the Fourier Projection-Slice Volume Rendering method [Parent](824-fourier-projection-slice-volume-rendering.md). It solves the problem of efficiently generating attenuation-only images (X-ray-like projections) and shaded volume renderings by exploiting the Fourier projection-slice theorem. What sets this implementation apart from standard digital compositing is the use of frequency-domain slicing and inverse transformation, reducing the per-view complexity from $O(N^3)$ to $O(N^2 \log N)$.

### [Depth Cueing via Linear Attenuation](966-depth-cueing-via-linear-attenuation.md)
This method approximates the perceptual depth cues of exponential attenuation in volume rendering by replacing the variable absorption coefficient with a linear function of position, enabling the integrand to be evaluated as a Fourier projection. It sets itself apart from standard exponential attenuation models by allowing efficient computation via the Fourier Projection-Slice Theorem.

### [Fourier Volume Rendering](6050-fourier-volume-rendering.md)
Fourier Volume Rendering (FVR) is a direct volume rendering technique that computes X-ray-like projections of a three-dimensional dataset by exploiting the Fourier projection-slice theorem. It reduces the computational complexity from $O(N^3)$ to $O(N^2 \log N)$ by transforming the volume into the frequency domain once, then generating views via 2D resampling and inverse transforms. Unlike [Ray Casting](52-ray-casting.md) or [Digital Compositing](53-digital-compositing.md), FVR produces attenuation-only images without hidden surface effects or transparency control.

### Fourier Projection-Slice Integration over Irregular Domains
A technique to compute integrals of functions over irregular spatial domains by decomposing the domain into spatially disjoint intervals and summing their Fourier projection-slice integrals [S3, unofficial]. It decomposes the ray segment connecting two interior points into spatially disjoint intervals, computes each interval integral using the projection-slice theorem, and applies recursive subdivision of volumes into octants to compute final images [S3, unofficial]. Time complexity is $O(N^2 \log^2 N)$ for the octant subdivision algorithm [S3, unofficial]. Applications include rendering spatially clipped volumes for medical visualization, fast calculation of inter-element attenuation for zonal radiosity algorithms, and approximate visibility determination for geometrically defined scenes [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Preprocessing:** $O(N^3 \log N)$ operations to compute the 3D spectrum [S3, S9, S12, S14, S15, unofficial].
*   **Per-View Rendering:** $O(N^2 \log N)$ operations for the inverse transform, plus $O(W^3 N^2)$ for slice extraction where $W$ is the nonzero width of the interpolating filter [S3, unofficial]. The slicing step alone performs in $O(M^2)$ for an $M \times M$ slice [S7].
*   **Memory:** Requires 16 bytes per voxel (pair of floating point numbers, 64-bit double precision) for the frequency domain representation, compared to 1 byte per voxel in spatial domain [S4, S11, unofficial]. If using 32-bit floats, 8 bytes per voxel [S3, unofficial].

**Measured Performance:**
*   For $N=256$, volume rendering using digital compositing requires 16M operations, while rendering using the projection-slice theorem requires only 512K operations, a speedup of more than an order of magnitude [S3, unofficial].
*   On a 36 MHz Silicon Graphics 4D/320 VGX workstation, no optimization was attempted [S15, unofficial].
*   GPU implementations accelerated the rendering stage by a factor of 17 [S14, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Fast generation of attenuation-only images; interactive frame rates achievable on GPU; supports filtering (lowpass, highpass, bandpass) with little overhead [S14, unofficial].
*   **Limitations:** Does not handle occlusion or hidden surface removal; lacks depth information (voxels on a viewing ray contribute equally regardless of distance) [S3, S4, S9, S11, S14, unofficial]; cannot implement local (image-space) operations or non-trivial shading models directly [S6, S8]; high interpolation cost and ghosting artifacts [S4, S10, S13, unofficial]; limited to parallel projection [S14, unofficial].

**When to Use:**
*   Used for generating X-rays, MR angiography, and attenuation-only renderings where speed is prioritized over occlusion [S3, S9, unofficial].
*   Useful for interactive segmentation and previewing large datasets [S12, S15, unofficial].

**When Not to Use:**
*   When occlusion, transparency, or complex lighting models are required [S3, unofficial][S6, S8][S14, unofficial].
*   When high-quality depth cues are essential without approximation [S3, unofficial].

## Background
**Origin:**
*   First applied by Dunne, Malzbender, and Napel et al. [S6, S8].
*   The first method rendering directly from the frequency domain was developed by Chiueh et al. [S6].
*   Developed by Levoy, Totsuka, and Malzbender [S14, unofficial]. Levoy and Malzbender independently applied the projection-slice theorem to DVR [S8].
*   The Fast Fourier Transform (FFT) was originally introduced by Cooley [S14, unofficial]. The Fast Hartley Transform (FHT) was proposed by Bracewell and Hao as an alternative to FFT [S14, unofficial].

**History and Development:**
*   Levoy and Totsuka extended the method with depth cues and shading [S14, unofficial].
*   Viola et al. proposed single-resolution slice-based rendering [S6].
*   Jansen et al. performed FDVR directly on graphics hardware [S8].
*   Implemented on GPU using CUDA and OpenGL shaders to achieve interactive framerates [S14, unofficial].

**Usage:**
*   Used to render human skull (CT data), Utah teapot (3D scan-converted), and turbine blade [S11, unofficial].
*   Implemented on programmable graphics hardware (GPU) using shader programs to achieve highly interactive framerates [S12, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S3] PDF Volume Rendering using the Fourier Projection-Slice Theorem, https://graphics.stanford.edu/papers/fourier/fourier.pdf, fetched 2026-10-27 (unofficial)
[S4] PDF Frequency Domain Volume Rendering - Stanford University, https://graphics.stanford.edu/papers/fvr/totsuka-fvr-sig93.pdf, fetched 2026-10-27 (unofficial)
[S5] Fourier Volume Rendering, https://core.ac.uk/works/16671645, fetched 2026-10-27 (unofficial)
[S6] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S7] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S8] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S9] PDF GPU-based Frequency Domain Volume Rendering - TU Wien, https://www.cg.tuwien.ac.at/research/publications/2004/Viola-2004-GPU/Viola-2004-GPU-Paper.pdf, fetched 2026-10-27 (unofficial)
[S10] PDF FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27 (unofficial)
[S11] Frequency Domain Volume Rendering - studylib.net, https://studylib.net/doc/18162906/frequency-domain-volume-rendering, fetched 2026-10-27 (unofficial)
[S12] PDF Hardware-Accelerated Frequency Domain Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2004/viola-2004-har2/viola-2004-har2-Paper.pdf, fetched 2026-10-27 (unofficial)
[S13] Frequency Domain Volume Rendering, https://graphics.stanford.edu/papers/fvr/fvr.pdf, fetched 2026-10-27 (unofficial)
[S14] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S15] Fourier Projection-Slice Theorem in Volume Rendering, https://www.scribd.com/document/796298963/Marc-Levoy-Volume-rendering-using-the-Fourier-projection-slice-theorem, fetched 2026-10-27 (unofficial)
