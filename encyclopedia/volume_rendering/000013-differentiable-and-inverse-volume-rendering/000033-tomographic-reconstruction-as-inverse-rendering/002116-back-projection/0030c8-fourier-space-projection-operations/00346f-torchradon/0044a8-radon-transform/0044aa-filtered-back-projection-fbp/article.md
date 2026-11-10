# Filtered Back-Projection (FBP)

Also called: IRADON Reconstruction, Inverse Radon Transform, Ramp Filter Reconstruction

## Overview
Filtered Back-Projection (FBP) is an approximate algorithm for the inverse Radon transform used to reconstruct a 2D image from measured projections (sinogram) [S2, unofficial]. It is among the fastest methods for inverse Radon transform [S2, unofficial]. The method relies on the Fourier slice theorem, applying a frequency-domain filter to projections before back-projecting them to correct for the blurring inherent in simple back-projection [S4, unofficial]. It builds on [Radon Transform](11306-radon-transform.md).

## How it works
**Inputs:**
- `sinogram`: A 2D array of shape `(num_detectors, num_angles)` containing projection data.
- `theta`: An array of projection angles in radians, typically spanning 0 to $\pi$ (180 degrees).
- `filter_name`: A string specifying the frequency filter to apply (e.g., 'ramp', 'shepp-logan', 'cosine', 'hamming', 'hann') [S2, unofficial].

**Outputs:**
- `reconstructed_image`: A 2D array representing the reconstructed image.

**Data Structures:**
- The sinogram is stored as a contiguous 2D array where rows correspond to detector positions and columns correspond to rotation angles.
- Angles are stored as a 1D array of floating-point values in radians.

**Steps:**
1. **Angle Preparation**: Generate projection angles `theta` from 0 to 180 degrees. The step size determines the angular resolution. For example, a 5-degree step yields 36 angles [S1, unofficial]. Convert degrees to radians: $\theta_i = \text{angle}_i \times \frac{\pi}{180}$.
2. **Filter Selection**: Select the filter type. The 'ramp' filter is the standard choice for basic reconstruction [S1, unofficial]. Other filters like 'shepp-logan', 'cosine', 'hamming', and 'hann' are available to suppress high-frequency noise [S2, unofficial].
3. **Frequency Domain Filtering**:
   - For each projection column in the sinogram:
     - Compute the 1D Fast Fourier Transform (FFT) of the projection: $P_k = \text{FFT}(p)$.
     - Determine the frequency bins $f$ corresponding to the projection width.
     - Multiply the FFT result by the filter function $H(f)$. For the ramp filter, $H(f) = |f|$ [S4, unofficial].
     - Compute the inverse FFT to return to the spatial domain: $p_{filtered} = \text{IFFT}(P_k \cdot H(f))$.
4. **Back-Projection**:
   - Initialize the reconstructed image grid to zero.
   - For each filtered projection $p_{filtered}$ at angle $\theta_i$:
     - Map the projection values back onto the 2D image grid along the ray paths defined by $\theta_i$.
     - Accumulate the values into the image grid. This involves interpolating the projection values onto the image pixel centers.
5. **Output**: The accumulated grid is the reconstructed image.

**Parameters:**
- `filter_name`: Determines the frequency response. 'ramp' is default. 'shepp-logan', 'cosine', 'hamming', 'hann' reduce noise but may blur edges [S2, unofficial].
- `theta`: Angular sampling. Finer steps improve accuracy but increase computation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Noise**: Simple back-projection causes blurring. The filter is essential to correct this [S4, unofficial].
- **Aliasing**: Insufficient angular sampling leads to streak artifacts.
- **Boundary Conditions**: Filter generation may require overscan to handle boundaries [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def filtered_back_projection(sinogram, theta, filter_name='ramp'):
    """
    Reconstruct an image from a sinogram using Filtered Back-Projection.
    
    Args:
        sinogram: 2D array (num_detectors, num_angles)
        theta: 1D array of angles in radians
        filter_name: String, one of 'ramp', 'shepp-logan', 'cosine', 'hamming', 'hann'
        
    Returns:
        reconstructed_image: 2D array
    """
    num_detectors, num_angles = sinogram.shape
    # Assume square image with same number of pixels as detectors
    image_size = num_detectors
    reconstructed_image = np.zeros((image_size, image_size))
    
    # Define filter function
    def get_filter(name, n):
        # Frequency bins from -N/2 to N/2-1
        freq = np.fft.fftfreq(n)
        if name == 'ramp':
            return np.abs(freq)
        elif name == 'shepp-logan':
            # Standard Shepp-Logan filter
            cutoff = 0.5
            return np.where(np.abs(freq) <= cutoff, 
                            (2 * np.sqrt(1 - (freq/cutoff)**2) / np.pi) * np.abs(freq), 
                            0)
        elif name == 'cosine':
            return np.abs(freq) * np.cos(np.pi * freq / 2)
        elif name == 'hamming':
            return np.abs(freq) * (0.54 + 0.46 * np.cos(np.pi * freq))
        elif name == 'hann':
            return np.abs(freq) * (0.5 + 0.5 * np.cos(np.pi * freq))
        else:
            return np.abs(freq) # Default to ramp
            
    filter_func = get_filter(filter_name, num_detectors)
    
    # Filter the sinogram
    filtered_sinogram = np.zeros_like(sinogram)
    for i in range(num_angles):
        proj = sinogram[:, i]
        # FFT, apply filter, IFFT
        proj_fft = np.fft.fft(proj)
        proj_fft_filtered = proj_fft * filter_func
        filtered_sinogram[:, i] = np.fft.ifft(proj_fft_filtered).real
        
    # Back-project
    # Create coordinate grid
    y, x = np.meshgrid(np.arange(image_size), np.arange(image_size), indexing='ij')
    center = image_size / 2.0
    
    for i in range(num_angles):
        angle = theta[i]
        # Rotate coordinates
        cos_a = np.cos(angle)
        sin_a = np.sin(angle)
        
        # Projected coordinate r
        r = (x - center) * cos_a + (y - center) * sin_a
        
        # Interpolate filtered projection values at r
        # Normalize r to detector indices (assuming sinogram columns are aligned)
        # Detector indices range from 0 to num_detectors-1
        # r is in physical units, need to map to indices
        # Assuming sinogram is centered, r ranges from -center to center
        # Map r to index: idx = r + center
        idx = r + center
        
        # Clamp indices to valid range
        idx = np.clip(idx, 0, num_detectors - 1)
        
        # Linear interpolation
        idx0 = np.floor(idx).astype(int)
        idx1 = np.ceil(idx).astype(int)
        w = idx - idx0
        
        # Get values from filtered sinogram
        val0 = filtered_sinogram[idx0, i]
        val1 = filtered_sinogram[idx1, i]
        
        # Interpolated value
        val = val0 * (1 - w) + val1 * w
        
        # Accumulate
        reconstructed_image += val
        
    # Normalize by number of angles
    reconstructed_image /= num_angles
    
    return reconstructed_image
```

## Subtopics

### Overscan Convolution Filter Generation
A method to generate a convolution filter for tomographic reconstruction that includes an overscan border (typically 30% extra pixels) to handle boundary conditions and ensure symmetry [S3, unofficial]. It calculates overscan dimensions as $w_{over} = \lceil w \times 0.3 \rceil$ and $h_{over} = \lceil h \times 0.3 \rceil$, then calls `build_convolution_filter` with base and overscan dimensions [S3, unofficial]. This preserves the parity (even/odd nature) of dimensions compared to non-overscanned filters, ensuring pixel alignment consistency [S3, unofficial].

### Ramp Filter
A frequency-domain filter applied to projections before backprojection to correct for the blurring inherent in simple backprojection [S4, unofficial]. It computes the FFT of the projection $p$, determines frequency bins, and multiplies the FFT result by the absolute value of the frequency bins ($|f|$) [S4, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N^2 \log N)$ due to FFTs, where $N$ is the image size. It is among the fastest methods for inverse Radon transform [S2, unofficial].
- **Memory Complexity**: $O(N^2)$ for storing the sinogram and image.
- **Performance**: On the Shepp-Logan phantom with a ramp filter, the RMS reconstruction error is 0.0283 [S2, unofficial].
- **Strengths**: Fast, simple, and effective for high-contrast, noise-free data.
- **Limitations**: Sensitive to noise and artifacts from insufficient sampling or misalignment. Iterative methods may be preferred for noisy or sparse data [S3, unofficial].
- **When to Use**: When speed is critical and data quality is high.
- **When Not to Use**: For noisy data or sparse views, where iterative reconstruction may perform better [S3, unofficial].

## Background
Filtered Back-Projection is a standard technique in tomographic reconstruction. It is based on the Fourier slice theorem [S2, unofficial]. The method is widely implemented in libraries such as scikit-image (`iradon`) [S1, unofficial][S2, unofficial]. It is considered a relatively noisy approach compared to iterative statistical methods [S3, unofficial].

## Sources
[S1] guibo-tech/Tomographic-Reconstruction, https://github.com/guibo-tech/Tomographic-Reconstruction, fetched 2026-10-26 (unofficial)
[S2] plot_radon_transform.rst.txt, https://scikit-image.org/docs/stable/_sources/auto_examples/transform/plot_radon_transform.rst.txt, fetched 2026-10-30 (unofficial)
[S3] simon-frankau/rediscovering-tomography, https://github.com/simon-frankau/rediscovering-tomography, fetched 2026-10-26 (unofficial)
[S4] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
