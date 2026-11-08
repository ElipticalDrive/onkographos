# Fourier Volume Rendering

Also called: FVR, Frequency Domain Volume Rendering (FDVR)

## Overview
Fourier Volume Rendering (FVR) is a visualization technique that generates attenuation-only projections resembling X-ray radiographs by operating on the spectral representation of a 3D volume rather than its spatial representation [S1][S2, unofficial]. It exploits the Fourier projection-slice theorem to compute these projections in $O(N^2 \log N)$ time per view, avoiding the $O(N^3)$ cost of spatial domain methods like ray casting [S1][S2, S8, unofficial]. Unlike its sibling [Fourier Projection-Slice Volume Rendering](824-fourier-projection-slice-volume-rendering.md), which focuses on the theoretical basis of generating projections, FVR describes the complete algorithmic pipeline including preprocessing, resampling, and inverse transformation to produce the final image [S1][S8, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D scalar volume $x(u_1, u_2, u_3)$ of size $N \times N \times N$.
*   **Output:** A 2D projection image $p_\theta(\hat{u}_2, \hat{u}_3)$ corresponding to a viewing direction $\theta$.

**Data Structures**
*   **Frequency Spectrum:** A 3D complex array $X(U_1, U_2, U_3)$ of size $N \times N \times N$.
*   **Projection Slice:** A 2D complex array $P_\theta(\hat{U}_2, \hat{U}_3)$ representing the resampled plane through the origin of the frequency spectrum.
*   **Image Buffer:** A 2D real array $p_\theta(\hat{u}_2, \hat{u}_3)$ of size $N \times N$ (or the target resolution).

**Algorithm Steps**
The method consists of a one-time preprocessing stage and a per-view rendering loop [S1][S2, unofficial].

1.  **Preprocessing:**
    1.  **Shift:** Apply a 3D `fft-shift` to the input volume $x$ to center it at the origin of the 3D space [S1][S2, unofficial].
    2.  **Transform:** Compute the forward 3D Discrete Fourier Transform (DFT) of the shifted volume to obtain the frequency spectrum $X(U_1, U_2, U_3)$ [S1][S2, S8, unofficial].
        $$X(U_1, U_2, U_3) = \sum_{u_1} \sum_{u_2} \sum_{u_3} x(u_1, u_2, u_3) e^{2\pi i (u_1 U_1 + u_2 U_2 + u_3 U_3)}$$
    3.  **Center:** Apply a 3D `fft-shift` to $X$ to center the zero-frequency component at the origin of the frequency space [S1][S2, unofficial].

2.  **Rendering Loop (for each viewing angle $\theta$):**
    1.  **Extract Slice:** Extract a 2D plane $P_{raw}$ passing through the origin of the centered spectrum $X$, with a normal vector parallel to the viewing direction $\theta$ [S1][S2, S8, unofficial].
    2.  **Resample:** If the viewing angle is not orthogonal to the grid axes, resample $P_{raw}$ onto a regular 2D grid $P_\theta(\hat{U}_2, \hat{U}_3)$ to reduce aliasing and ghosting artifacts [S1][S2, unofficial]. This step is mandatory for non-orthogonal views [S1].
        *   *Note:* Linear interpolation causes significant artifacts. High-order interpolation filters (e.g., POCS 5x5x5) or Hamming-windowed sinc filters are required [S1][S2, S7, S8, unofficial].
    3.  **Inverse Transform:** Compute the 2D Inverse Discrete Fourier Transform (IDFT) of $P_\theta$ to obtain the spatial projection $p_{raw}(\hat{u}_2, \hat{u}_3)$ [S1][S2, S8, unofficial].
        $$p_\theta(\hat{u}_2, \hat{u}_3) = \sum_{\hat{U}_2} \sum_{\hat{U}_3} P_\theta(\hat{U}_2, \hat{U}_3) e^{-2\pi i (\hat{u}_2 \hat{U}_2 + \hat{u}_3 \hat{U}_3)}$$
    4.  **Shift:** Apply a 2D `fft-shift` to $p_{raw}$ to move the center of the image from the edge to the origin of the display grid [S1][S2, unofficial].
    5.  **Output:** The result is theoretically real. Discard small imaginary parts resulting from resampling artifacts and store the real part in the frame buffer [S8, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fourier_volume_rendering(volume, view_direction):
    """
    volume: 3D numpy array of shape (N, N, N)
    view_direction: 3D vector (nx, ny, nz)
    Returns: 2D numpy array of shape (N, N)
    """
    N = volume.shape[0]
    
    # --- Preprocessing (Execute Once) ---
    # 1. Shift volume to center
    shifted_volume = fft_shift_3d(volume)
    
    # 2. Forward 3D FFT
    spectrum = fft_3d(shifted_volume)
    
    # 3. Center zero-frequency component
    centered_spectrum = fft_shift_3d(spectrum)
    
    # --- Rendering Loop (Execute Per View) ---
    
    # 1. Extract plane perpendicular to view_direction through origin
    # This requires defining the coordinate system for the 2D plane
    # and sampling the 3D spectrum at those coordinates.
    raw_slice = extract_plane(centered_spectrum, view_direction)
    
    # 2. Resample if necessary (mandatory for non-orthogonal views)
    # Use high-order interpolation (e.g., sinc or POCS) to avoid ghosting
    if not is_orthogonal(view_direction):
        resampled_slice = resample_slice(raw_slice, filter_order=5)
    else:
        resampled_slice = raw_slice
        
    # 3. Inverse 2D FFT
    projected_image = ifft_2d(resampled_slice)
    
    # 4. Shift image center to origin
    final_image = fft_shift_2d(projected_image)
    
    # 5. Return real part (discard numerical noise)
    return final_image.real
```

**Parameters**
*   **Interpolation Filter Size:** $5 \times 5 \times 5$ POCS filter is recommended to minimize aliasing artifacts during frequency domain interpolation [S3, S8, unofficial].
*   **Precision:** 16-bit integer precision is sufficient for visual quality in hardware implementations [S8, unofficial]. Internal precision of 19 bits may be used to compensate for FFT loss [S8, unofficial].
*   **Zero-Padding:** Zero-padding the volume in the spatial domain before the 3D FFT is required to resolve ghosting artifacts caused by signal replication in the spatial domain [S1][S2, unofficial].

**Edge Cases and Pitfalls**
*   **Ghosting Artifacts:** Sampling the frequency domain is equivalent to replicating the signal in the spatial domain, causing ghosting. Mitigated by zero-padding and high-order interpolation [S1][S2, S7, unofficial].
*   **Aliasing:** 3-D aliasing occurs due to imperfect resampling filters; 2-D aliasing occurs if the 2-D plane is not resampled quickly enough [S8, unofficial].
*   **Data Structure:** Conventional FFT output is not structured for resampling. `fft-shift` stages are mandatory to rearrange data [S1][S2, unofficial].
*   **No Occlusion:** The basic FVR integral computes line integrals (X-ray-like) and does not model emission, scattering, or opacity control directly [S1][S2, S8, unofficial].

## Subtopics

### [Fourier Domain Approximate Shading](6892-fourier-domain-approximate-shading.md)
Fourier Domain Approximate Shading is a method for approximating diffuse shading in the Fourier domain by expressing shading terms (linear, quadratic, cubic) as convolutions of the volume's Fourier transform $F$ and frequency vectors. It solves the problem of applying spatial lighting models to volume data rendered via Fourier Volume Rendering (FVR) without requiring repeated 3D Fourier transforms of the shaded volume for constant light sources. Unlike [Spherical Harmonics](6052-spherical-harmonics.md), which allows efficient relighting, this method is efficient only for constant light sources because changing the light direction requires recomputing expensive full-volume convolutions.

### [Polar Coordinate Fourier Volume Rendering](7426-polar-coordinate-fourier-volume-rendering.md)
Polar Coordinate Fourier Volume Rendering (Polar FVR) is a volume rendering technique that stores and processes frequency domain data in polar coordinates to reduce computational complexity from $O(n^2 \log n)$ to $O(n^2)$ and improve memory access efficiency. It sets itself apart from traditional Rectilinear FVR by exploiting data regularity and high sampling density near the origin, allowing slice extraction via nearest-neighbor or bilinear interpolation instead of trilinear interpolation, thereby shifting computational cost to preprocessing.

### Frequency Domain Depth Cueing
Frequency Domain Depth Cueing is a method to implement depth cueing (weighting voxel intensities by distance) in FVR by convolving the 3-D Fourier transform with a combined filter $G(U)$ in the frequency domain, avoiding the need to recompute the 3-D forward transform for every view [S8, unofficial].

## Operating characteristics
*   **Time Complexity:** The preprocessing stage (3D FFT) has a complexity of $O(N^3 \log N)$ but is executed only once [S2, unofficial]. The rendering loop (resampling + 2D IFFT) has a complexity of $O(N^2 \log N)$ [S1][S2, S8, unofficial]. This is significantly faster than spatial domain algorithms which are $O(N^3)$ [S1][S2, S3, S8, unofficial].
*   **Memory Complexity:** Requires storage for the 3D frequency spectrum. For a $512^3$ dataset, this requires approximately 256 Mb of memory (assuming 2 bytes/voxel or similar density implied by the calculation) [S8, unofficial].
*   **Performance:** On a HP 9000/735 (99 MHz, 256^3 dataset, 8-bit voxels), the 3-D Forward Hartley Transform took 5.5 minutes, while the FVR projection (linear interp) took 9 seconds and the 2-D Inverse Hartley Transform took 4 seconds [S8, unofficial]. FVR is typically 100 to 1000 times faster than conventional ray casting [S8, unofficial].
*   **Hardware Implementation:** A hardware implementation using PDSP16510/A FFT chips (1024 points in 100μs) estimated a frame rate of 26 fps for a 512^3 dataset, with a hardware cost of ~$1100 per chip [S8, unofficial]. Scaling to 512^3 requires 4 interpolation chips and 4 FFT chips, and faster static RAM (~9 ns clock) [S8, unofficial].
*   **Strengths:** Fast generation of arbitrary viewing angles; efficient for attenuation-only (X-ray-like) projections; suitable for compression-based rendering [S1][S2, unofficial][S4][S6, unofficial].
*   **Limitations:** Cannot model emission, scattering, or opacity control directly; lacks occlusion; sensitive to aliasing and ghosting artifacts requiring high-quality resampling; MIP artifacts are significant [S1][S2, S8, unofficial].

## Background
FVR was introduced independently by Malzbender [12] and another group [3] at approximately the same time [S8, unofficial]. It is based on the Fourier projection-slice theorem, which states that the 2-D Fourier transform of a 2-D projection of a 3-D object is a 2-D plane passing through the origin of the 3-D Fourier transform of that object [S8, unofficial]. The method was initially explored for hardware implementation to achieve real-time performance [S3, unofficial]. It has been used in digital radiography and GPU-based implementations using CUDA [S1][S2, S6, unofficial].

## Sources
[S1] High Performance GPU-Based Fourier Volume Rendering, https://doi.org/10.1155/2015/590727, fetched 2026-10-27
[S2] High Performance GPU-Based Fourier Volume Rendering - PMC, https://pmc.ncbi.nlm.nih.gov/articles/PMC4381991/, fetched 2026-10-27 (unofficial)
[S3] Fourier Volume Rendering final version, https://core.ac.uk/works/16976649, fetched 2026-10-27 (unofficial)
[S4] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S5] Indirect volume rendering, https://slidetodoc.com/direct-volume-rendering-what-is-volume-rendering-accumulate/, fetched 2026-10-29 (unofficial)
[S6] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
[S7] (PDF) Fourier volume rendering on the GPU using a split- ..., https://www.researchgate.net/publication/220838994_Fourier_volume_rendering_on_the_GPU_using_a_split-stream-FFT, fetched 2026-10-31 (unofficial)
[S8] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
