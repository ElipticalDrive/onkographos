# Filtered Back-Projection (FBP)

Also called: Filter Back Projection, Transformée inverse de Radon, Rétroprojection des projections filtrées

## Overview
Filtered Back-Projection (FBP) is a direct tomographic reconstruction algorithm that recovers a 2D image from its measured projections (sinogram) by applying a high-pass frequency filter to the projections before back-projecting them [S2, S3, unofficial]. It solves the inverse Radon transform problem, compensating for the inherent low-pass blurring of simple back-projection by leveraging the Fourier Slice Theorem [S2, S3, unofficial]. Unlike Fourier-based reconstruction methods that require interpolation between Cartesian and polar grids (leading to $O(N^2)$ complexity or interpolation artifacts), FBP operates directly on the projections and achieves $O(N \log N)$ time complexity via Fast Fourier Transforms (FFT) [S2, unofficial].

## How it works
**Inputs:**
*   Sinogram $S$: A 2D array of size $N_\rho \times N_\theta$, where $N_\rho$ is the number of radial samples per projection and $N_\theta$ is the number of projection angles.
*   Angles $\Theta$: Array of projection angles $\theta_i$ in radians, typically spanning $[0, \pi)$ [S2, unofficial].
*   Radial sampling step $d_\rho$: The distance between adjacent samples in the projection domain [S2, unofficial].
*   Filter type: e.g., 'ramp', 'hamming', 'hann', 'shepp-logan', 'cosine' [S3, unofficial].
*   Image dimensions: Width $W$ and Height $H$ [S2, unofficial].

**Outputs:**
*   Reconstructed image $I$: A 2D array of size $H \times W$ [S2, unofficial].

**Data Structures:**
*   `sinogram`: 2D float array $(N_\rho, N_\theta)$.
*   `angles`: 1D float array $(N_\theta)$.
*   `filtered_projections`: 2D float array $(N_\rho, N_\theta)$.
*   `image`: 2D float array $(H, W)$, initialized to zero.

**Parameters:**
*   $N_\theta$: Number of projection angles. SSIM plateaus when $N_\theta \ge 40$ for a $512^2$ image [S2, unofficial].
*   $d_\rho$: Radial sampling step. Nyquist frequency is $\nu_{Nyquist} = \frac{1}{2 d_\rho}$ [S2, unofficial].
*   Window parameter $a$: Used in Hann ($a=0.5$) and Hamming ($a=0.54$) windows [S2, unofficial].

**Algorithm Steps:**

1.  **Initialize Image:** Create an output image $I$ of size $H \times W$ and set all elements to 0 [S2, unofficial].
2.  **Filter Projections:** For each projection angle $\theta_i$ in $\Theta$:
    a. Extract the 1D projection vector $R_{\theta_i}$ of length $N_\rho$ from the sinogram [S2, unofficial].
    b. Compute the 1D Discrete Fourier Transform (DFT) of $R_{\theta_i}$ to obtain $\hat{R}_{\theta_i}(\nu)$ [S2, unofficial].
    c. Define the filter function $H(\nu)$. The base ramp filter is $|\nu|$. For windowed filters, multiply by a window function:
       $$ H(\nu) = \begin{cases} a + (1-a)\cos\left(\frac{\pi\nu}{\nu_{Nyquist}}\right) & \text{if } |\nu| < \nu_{Nyquist} \\ 0 & \text{otherwise} \end{cases} $$
       where $\nu_{Nyquist} = \frac{1}{2 d_\rho}$ [S2, unofficial].
       *Note: Standard 'ramp' filter implies $H(\nu) = |\nu|$ truncated at $\nu_{Nyquist}$ (Ram-Lak) [S2, unofficial].*
    d. Apply the filter in the frequency domain: $\hat{R}_{\theta_i}^{BF}(\nu) = \hat{R}_{\theta_i}(\nu) \times H(\nu)$ [S2, unofficial].
    e. Compute the Inverse 1D DFT of $\hat{R}_{\theta_i}^{BF}(\nu)$ to obtain the filtered projection $R_{\theta_i}^{BF}$ [S2, unofficial].
3.  **Back-Project:** For each pixel $(j, i)$ in the image grid (where $0 \le j < W$ and $0 \le i < H$):
    a. Initialize accumulator `sum` = 0.
    b. For each projection angle $\theta_k$ in $\Theta$:
       i. Calculate the radial coordinate $\rho$ corresponding to the pixel $(j, i)$ at angle $\theta_k$:
          $$ \rho = j \cos \theta_k + i \sin \theta_k $$
          *Note: Some implementations center the coordinate system, using $\rho = (j - W/2) \cos \theta_k + (i - H/2) \sin \theta_k$ [S2, unofficial].*
       ii. Sample the filtered projection $R_{\theta_k}^{BF}$ at $\rho$. Since $\rho$ is continuous and the sinogram is discrete, interpolation is required. Common kernels include Dirac (nearest neighbor), Linear, Spline, or Area [S2, unofficial].
       iii. Add the interpolated value to `sum`.
    c. Set $I(i, j) = \text{sum}$ [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def filtered_back_projection(sinogram, angles, d_rho, width, height, filter_name='ramp'):
    """
    Reconstructs an image from a sinogram using Filtered Back-Projection.
    
    Parameters:
    sinogram: ndarray of shape (N_rho, N_theta)
    angles: ndarray of shape (N_theta,) in radians
    d_rho: float, radial sampling step
    width: int, image width W
    height: int, image height H
    filter_name: str, type of filter ('ramp', 'hamming', 'hann')
    
    Returns:
    image: ndarray of shape (height, width)
    """
    N_rho, N_theta = sinogram.shape
    image = np.zeros((height, width))
    
    # Calculate Nyquist frequency
    nu_nyquist = 1.0 / (2.0 * d_rho)
    
    # Precompute frequency bins for FFT
    # np.fft.fftfreq returns frequencies in cycles per sample
    freqs = np.fft.fftfreq(N_rho, d=d_rho)
    
    # Define filter kernel
    if filter_name == 'ramp':
        # Ram-Lak filter: |nu|
        kernel = np.abs(freqs)
    elif filter_name == 'hamming':
        a = 0.54
        kernel = np.where(np.abs(freqs) < nu_nyquist, 
                          a + (1 - a) * np.cos(np.pi * freqs / nu_nyquist), 
                          0) * np.abs(freqs)
    elif filter_name == 'hann':
        a = 0.5
        kernel = np.where(np.abs(freqs) < nu_nyquist, 
                          a + (1 - a) * np.cos(np.pi * freqs / nu_nyquist), 
                          0) * np.abs(freqs)
    else:
        # Default to ramp if unknown
        kernel = np.abs(freqs)
        
    # Step 2: Filter Projections
    filtered_projections = np.zeros_like(sinogram)
    for k in range(N_theta):
        # 1D FFT of the projection
        proj_fft = np.fft.fft(sinogram[:, k])
        # Apply filter
        proj_fft_filtered = proj_fft * kernel
        # Inverse 1D FFT
        filtered_projections[:, k] = np.real(np.fft.ifft(proj_fft_filtered))
        
    # Step 3: Back-Projection
    # Precompute cosine and sine for all angles to avoid repeated calculation
    cos_angles = np.cos(angles)
    sin_angles = np.sin(angles)
    
    # Create coordinate grids
    # j corresponds to x-axis (width), i corresponds to y-axis (height)
    j_coords, i_coords = np.meshgrid(np.arange(width), np.arange(height))
    
    for k in range(N_theta):
        # Calculate rho for all pixels for this angle
        # Using center-based coordinates as per [S2, unofficial] discrete inverse formula
        rho = (j_coords - width / 2.0) * cos_angles[k] + (i_coords - height / 2.0) * sin_angles[k]
        
        # Interpolate filtered projection at rho
        # Using linear interpolation for robustness; nearest neighbor is Dirac kernel
        # np.interp handles boundary conditions by clamping
        values = np.interp(rho, np.linspace(-width/2.0 * np.sqrt(2), width/2.0 * np.sqrt(2), N_rho), 
                           filtered_projections[:, k])
        
        # Accumulate
        image += values
        
    return image
```

**Edge Cases and Pitfalls:**
*   **Noise Amplification:** The ramp filter amplifies high frequencies, which are often dominated by noise. Using windowed filters (Hann, Hamming) attenuates these very high frequencies to reduce artifacts, though SNR may remain statistically similar to Ram-Lak in some contexts [S2, unofficial].
*   **Under-sampling:** Large $d_\rho$ (under-sampling) maximizes SNR but degrades image sharpness, making reconstruction difficult to exploit [S2, unofficial].
*   **Angular Coverage:** High radial resolution ($N_\rho$ increase) without sufficient angular coverage ($N_\theta < 40$) causes complete loss of contrast and luminance [S2, unofficial].
*   **Interpolation Artifacts:** The choice of pixel kernel (Dirac, Spline, Linear, Area) affects SSIM. The Area kernel provides the best SSIM and most uniform Point Spread Function (PSF) [S2, unofficial].

## Subtopics

### Overscan Convolution Filter Generation
A method to generate a convolution filter for tomographic reconstruction that includes an overscan border (typically 30% extra pixels) to handle boundary conditions and ensure symmetry [S4, unofficial]. It calculates overscan dimensions ($w_{over} = \lceil w \times 0.3 \rceil$, $h_{over} = \lceil h \times 0.3 \rceil$) and builds the filter with these extended dimensions [S4, unofficial]. This preserves the parity (even/odd nature) of dimensions compared to non-overscanned filters, ensuring pixel alignment consistency [S4, unofficial].

### Ramp Filter
A frequency-domain filter applied to projections before backprojection to correct for the blurring inherent in simple backprojection [S5, unofficial]. It is computed by multiplying the FFT of the projection by the absolute value of the frequency bins ($|\nu|$) [S5, unofficial]. This corresponds to the Ram-Lak filter when truncated at the Nyquist frequency [S2, unofficial].

### Discrete Inverse Radon Transform
The core back-projection step of FBP, reconstructing an image $I$ of size $W \times H$ from a sinogram $S$ by summing contributions of projection values to pixels, assuming parallel beam geometry [S2, unofficial]. It initializes $I(i,j)$ to zero and updates it by adding $S(i_\theta, i_\rho) \times \Delta(\rho - j \cos \theta - i \sin \theta)$ for all angles and radial samples, where $\Delta$ is the discrete Dirac impulse [S2, unofficial]. Optimized implementations map each pixel to contributing projection lines, achieving $O(N \log N)$ complexity [S2, unofficial].

### Fourier Slice Theorem
The mathematical foundation of FBP, stating that the 1D Fourier transform of a projection $R_\theta$ at angle $\theta$ is equal to a radial slice through the 2D Fourier transform of the original image $f$ at the same angle $\theta$ [S2, unofficial]. Formally, $F_\theta(\nu) = F^{2D}(\psi, \nu)|_{\psi=\theta}$ [S2, unofficial]. This allows the reconstruction of the 2D Fourier space of the domain by combining the 1D Fourier transforms of each projection, which is then inverted to obtain the image [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$, where $N$ is the number of pixels/projections, due to the use of FFTs [S2, S3, unofficial]. This makes it competitive with Fourier-based methods, which can be $O(N^2)$ if avoiding interpolation or suffer from artifacts if interpolating [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the sinogram, filtered projections, and output image [S2, unofficial].
*   **Performance:** FBP is among the fastest methods for inverse Radon transform [S3, unofficial]. On the Shepp-Logan phantom with a ramp filter, the RMS reconstruction error is approximately 0.0283 [S3, unofficial].
*   **Strengths:** Fast execution, simple implementation, good preservation of contrast and uniform PSF [S2, unofficial]. It is widely used in tomography and is easily parallelizable on GPUs due to independent pixel contributions [S2, unofficial].
*   **Limitations:** Less robust to noise than iterative methods or some Fourier methods, as the ramp filter amplifies high-frequency noise [S2, S3, unofficial]. Quality depends heavily on sufficient angular coverage ($N_\theta$) and radial sampling ($d_\rho$) [S2, unofficial].
*   **When to Use:** When speed is critical and the data is well-sampled with low noise. Suitable for real-time applications or large datasets where iterative methods are too slow [S3, unofficial].
*   **When Not to Use:** When data is severely under-sampled or noisy, as FBP will produce significant artifacts. Iterative reconstruction methods may be preferred in these cases [S3, unofficial].

## Background
Filtered Back-Projection (FBP) is one of the most popular and widely used direct reconstruction methods in tomography [S2, unofficial]. Its mathematical foundation is the Fourier Slice Theorem [S3, unofficial]. The method was developed to provide a computationally efficient alternative to Fourier-based reconstruction while avoiding the interpolation artifacts associated with polar-to-Cartesian grid conversion [S2, unofficial]. The "Ram-Lak" filter is named after its inventors, Ramachandran and Lakshiminarayanan [S2, unofficial]. It is implemented in standard libraries such as `scikit-image` (`iradon`) [S1, S3, unofficial].

## Sources
[S1] guibo-tech/Tomographic-Reconstruction, https://github.com/guibo-tech/Tomographic-Reconstruction, fetched 2026-10-26 (unofficial)
[S2] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
[S3] plot_radon_transform.rst.txt, https://scikit-image.org/docs/stable/_sources/auto_examples/transform/plot_radon_transform.rst.txt, fetched 2026-10-30 (unofficial)
[S4] simon-frankau/rediscovering-tomography, https://github.com/simon-frankau/rediscovering-tomography, fetched 2026-10-26 (unofficial)
[S5] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
