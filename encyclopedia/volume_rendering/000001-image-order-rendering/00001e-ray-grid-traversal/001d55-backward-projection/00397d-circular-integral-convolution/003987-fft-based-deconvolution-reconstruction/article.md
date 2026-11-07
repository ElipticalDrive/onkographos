# FFT-based Deconvolution Reconstruction

Also called: Deconvolution-based reconstruction

## Overview
FFT-based deconvolution reconstruction is a method that uses Fast Fourier Transform (FFT) techniques to invert the convolution process inherent in back-projection, aiming to reduce alignment and blurring errors compared to simple back-propagation [S1, unofficial]. It is distinct from [Circular Integral Convolution](8507-circular-integral-convolution.md), which generates a blurred image as a pre-processing step; this method performs the actual inversion to recover the original image from that blurred state [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A blurred reconstruction image (result of integrating ray values over all angles, equivalent to convolving the true image with a $1/r$ weighted filter) [S1, unofficial].
- **Output:** The reconstructed image approximating the original source [S1, unofficial].

**Data Structures:**
- **Image:** A 2D grid of pixel values. The internal representation must use floating-point types (e.g., `f64`) to avoid discretization errors that artificially raise RMS error [S1, unofficial].
- **Filter:** A 2D array representing the convolution kernel in the spatial domain, which is transformed into the frequency domain.

**Steps:**
1. **Generate Blurred Reconstruction:** For each point in the image space, integrate over all rays passing through that point. This yields a blurred image equivalent to convolving the original image with a symmetric $1/r$ weighted filter [S1, unofficial]. This step is often referred to as "back propagation" [S1, unofficial].
2. **Build Convolution Filter:** Construct the spatial domain filter $H$ where the weight for each pixel is proportional to $1/r$, with $r$ being the distance from the center [S1, unofficial].
   - **Singularity Handling:** The weight at the origin ($r=0$) is infinite. Discretize this by setting a minimal radius $r_{min}$ equal to half the radius of a pixel [S1, unofficial].
   - **Centering:** Ensure the filter is centered correctly. Odd-sized filters are centered on a pixel; even-sized filters are centered between pixels. This requires correcting for rounding and accounting for FFT conventions where sample points are at the side of the pixel (left-most at zero, right-most under $2\pi$) [S1, unofficial].
3. **Transform to Frequency Domain:** Compute the FFT of the convolution filter $H$ to obtain $H_{fft}$ [S1, unofficial].
4. **Invert Filter:** Compute the inverse of the filter in the frequency domain. This is done by point-wise division of 1 by $H_{fft}$, with a small regularization term added to avoid division by zero [S1, unofficial].
   - Formula: $H_{inv}(u,v) = \frac{1}{H_{fft}(u,v) + \epsilon}$
   - $\epsilon$ is a small value (e.g., $10^{-2}$ to $10^{-7}$) to stabilize inversion [S1, unofficial].
5. **Deconvolve:** Multiply the FFT of the blurred reconstruction image $I_{blur}$ by the inverted filter $H_{inv}$ [S1, unofficial].
   - Formula: $I_{rec\_fft}(u,v) = I_{blur\_fft}(u,v) \times H_{inv}(u,v)$
6. **Inverse Transform:** Compute the Inverse FFT (IFFT) of $I_{rec\_fft}$ to obtain the reconstructed image in the spatial domain [S1, unofficial].
7. **Normalization and Shifting:** Scale the result by $1/(width \times height)$ and shift the image to correct for any FFT-induced shifts [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def build_convolution_filter(width, height, r_min=0.5):
    """
    Builds the spatial domain convolution filter with 1/r weighting.
    r_min is half the radius of a pixel to handle the singularity at the origin.
    """
    filter_img = np.zeros((height, width), dtype=np.float64)
    center_x = width / 2.0
    center_y = height / 2.0
    
    # Handle even/odd centering: 
    # For even sizes, the center is between pixels. 
    # We iterate over pixel centers.
    for y in range(height):
        for x in range(width):
            # Distance from center
            dx = x - center_x
            dy = y - center_y
            r = math.sqrt(dx*dx + dy*dy)
            
            # Apply 1/r weighting with minimal radius to avoid infinity
            if r < r_min:
                r = r_min
                
            filter_img[y, x] = 1.0 / r
            
    return filter_img

def fft_deconvolution_reconstruction(blurred_image, filter_width, filter_height, epsilon=1e-2):
    """
    Performs FFT-based deconvolution to reconstruct the original image.
    blurred_image: 2D numpy array (f64) of the back-projected/blurred image.
    filter_width, filter_height: dimensions of the convolution filter.
    epsilon: regularization constant for inversion.
    """
    # 1. Build the convolution filter
    # Note: The filter size should match the image size for circular convolution
    # or be padded appropriately. Here we assume the filter is built to match
    # the image dimensions for simplicity, or the blurred image is padded.
    # In practice, the filter is often built at the image resolution.
    filter_img = build_convolution_filter(filter_width, filter_height, r_min=0.5)
    
    # 2. FFT of the filter
    filter_fft = np.fft.fft2(filter_img)
    
    # 3. Invert the filter in frequency domain
    # Add epsilon to avoid division by zero
    filter_inv = 1.0 / (filter_fft + epsilon)
    
    # 4. FFT of the blurred image
    blurred_fft = np.fft.fft2(blurred_image)
    
    # 5. Point-wise multiplication (deconvolution)
    rec_fft = blurred_fft * filter_inv
    
    # 6. Inverse FFT
    rec_img = np.fft.ifft2(rec_fft)
    
    # 7. Take real part and normalize
    # The FFT implementation might produce small imaginary parts due to precision
    rec_img = np.real(rec_img)
    
    # Normalize by image size if the FFT implementation doesn't do it automatically
    # Note: numpy's fft2 does not normalize by default in ifft2, but ifft2 
    # normalizes by 1/(N*M). However, depending on the specific FFT library 
    # or custom implementation, explicit normalization might be needed.
    # Based on the source code, scaling by 1/(width*height) is applied after ifft 
    # if the FFT was forward without normalization. 
    # Standard numpy ifft2 normalizes, so this step might be redundant or 
    # specific to the custom FFTImage class in the source. 
    # We include it as per the source's explicit scaling step.
    rec_img = rec_img / (filter_width * filter_height)
    
    # 8. Shift the image to correct for FFT centering
    # The source code mentions shifting by width/2, height/2.
    # This is likely to correct for the zero-frequency component placement.
    rec_img = np.fft.ifftshift(rec_img) # Or fftshift depending on convention
    
    return rec_img

def check_alignment(source_image, reconstructed_image):
    """
    Generates a difference image to detect misalignment.
    Misalignment appears as positive difference on one edge and negative on the other.
    """
    diff = source_image - reconstructed_image
    return diff
```

**Parameters:**
- **Minimal Radius ($r_{min}$):** Set to half the radius of a pixel to discretize the singularity at the origin where the $1/r$ weight would otherwise be infinite [S1, unofficial].
- **Epsilon ($\epsilon$):** A small value (e.g., $10^{-2}$ to $10^{-7}$) added to the filter in the frequency domain to prevent division by zero during inversion [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Using `u8` internally for images introduces discretization error that artificially raises RMS error. Switching to `f64` removes this error [S1, unofficial].
- **Filter Centering:** Filter behavior differs for odd vs. even filter sizes. Odd filters are centered in the middle of a pixel, while even filters are centered between pixels. Incorrect centering leads to misalignment [S1, unofficial].
- **Convergence:** Achieving good deconvolution results is numerically difficult and may not achieve perfect convergence [S1, unofficial].
- **Small Kernel Artifacts:** Small kernel deconvolution (e.g., 9x9) fails to account for distant parts of the image, resulting in dark grey artifacts where black is expected. FFT-based methods handle the full filter accurately [S1, unofficial].

## Subtopics

### Back Propagation with Convolution/Deconvolution
A reconstruction method where integrating over all rays passing through a point yields a blurred image equivalent to convolving the original with a $1/r$ weighted filter, which is then deconvolved to recover the original [S1, unofficial]. It is invoked via the command line flag `--algorithm=convolution` [S1, unofficial]. The blurring filter is symmetric, allowing the skip of signal flipping usually required in convolution definitions [S1, unofficial]. Deconvolution is performed in the frequency domain by point-wise division by the filter [S1, unofficial]. Achieving good results is numerically difficult and may not achieve perfect convergence [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by the FFT operations, which are $O(N \log N)$ for an image of size $N \times N$.
- **Memory Complexity:** $O(N)$ for storing the image and filter in both spatial and frequency domains.
- **Performance:** With matrix-inversion based tests using `f64` internal representation, per-pixel RMS error goes to zero for sufficient rays and angles, achieving fully-accurate reconstruction when the inverse matrix is fully determined [S1, unofficial].
- **Strengths:** Handles the full filter accurately, avoiding artifacts associated with small kernel deconvolution [S1, unofficial].
- **Limitations:** Numerically difficult to achieve perfect convergence; sensitive to filter centering and precision [S1, unofficial].
- **When to Use:** When high accuracy is required and computational resources allow for FFT operations.
- **When Not to Use:** When small kernel approximations are sufficient or when numerical stability is a major concern without proper regularization.

## Background
This method originates from the work of Simon Frankau, as documented in the "rediscovering-tomography" repository [S1, unofficial]. The key insight was that integrating over all rays passing through a point is equivalent to convolving the original image with a $1/r$ weighted filter, a process known as "back propagation" [S1, unofficial]. The deconvolution approach was developed to invert this blurring effect, with the author noting that while the standard algorithm uses a "ramp" high-pass filter, a numerical inversion of the specific back-projection filter was used in this implementation [S1, unofficial].

## Sources
[S1] simon-frankau/rediscovering-tomography, https://github.com/simon-frankau/rediscovering-tomography, fetched 2026-10-26 (unofficial)
