# Perceptual Loss Function for Color Holography

Also called: Filtered RGB Loss, Opponent Color Space Loss, Perceptual Loss Function, Perceptual loss

## Overview
This method addresses the overdetermined nature of generating full-color holograms with a single Spatial Light Modulator (SLM) illuminated by simultaneous red, green, and blue light sources, where the output has three times the degrees of freedom of the SLM pattern [S1]. It reduces color deviations and desaturation by transforming the error metric into opponent color space and applying low-pass filters that mimic the human visual system's (HVS) contrast sensitivity, thereby de-prioritizing high-frequency chrominance errors [S1]. Unlike standard RGB loss, which treats all color channels equally, this approach shifts optimization error into imperceptible regions of the color space, resulting in visually less noisy holograms with superior color fidelity [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Target RGB image $I_{\lambda_r}, I_{\lambda_g}, I_{\lambda_b}$ at wavelengths $\lambda_r, \lambda_g, \lambda_b$; SLM pattern $s$ (initially random or pre-optimized).
*   **Output:** Optimized SLM phase pattern $s$ that minimizes the perceptual loss.

**Data Structures**
*   **Target Image:** 2D array of shape $(H, W, 3)$ containing normalized intensity values for Red, Green, and Blue channels.
*   **Opponent Channels:** 2D arrays of shape $(H, W)$ for Luminance ($O_1$), Red-Green Chrominance ($O_2$), and Blue-Yellow Chrominance ($O_3$).
*   **Filters:** 2D frequency-domain masks $k_1, k_2, k_3$ representing low-pass filters in the Fourier domain.

**Parameters**
*   **Luminance Filter Width ($\gamma_1$):** 0.75 (75% of Fourier space width) [S1].
*   **Chrominance Filter Width ($\gamma_2, \gamma_3$):** 0.45 (45% of Fourier space width) [S1].
*   **System Constants:** SLM pixel pitch $p$ (e.g., 8 µm) and eyepiece focal length $f$ (e.g., 36.6 mm) [S1].
*   **Gamma:** $\gamma = 0.75$ for luminance, $\gamma = 0.45$ for chrominance [S1].

**Numbered Steps**
1.  **Forward Propagation:** Compute the simulated hologram intensity $I$ from the current SLM pattern $s$ using the differentiable physical model (e.g., Fresnel propagation) for each wavelength $\lambda_r, \lambda_g, \lambda_b$ [S1].
2.  **Color Space Transformation:** Convert the RGB intensities of both the target image $\hat{I}$ and the simulated hologram $I$ into opponent color space channels $O_1, O_2, O_3$ using the linear transformation defined below [S1].
3.  **Filter Generation:** Construct low-pass filters $k_1, k_2, k_3$ in the Fourier domain. The filter width is defined as a fraction $\gamma$ of the total Fourier space width. For a system with $p=8 \mu m$ and $f=36.6 mm$, the cutoff frequencies are 30 cycles/deg for luminance and 18 cycles/deg for chrominance [S1].
4.  **Frequency Domain Filtering:** Apply the filters $k_i$ to the opponent channels of both the target and the simulation via 2D convolution (or multiplication in the Fourier domain) [S1].
5.  **Loss Calculation:** Compute the pixel-wise loss (e.g., Mean Squared Error) between the filtered target channels and the filtered simulated channels for all three opponent components [S1].
6.  **Optimization:** Update the SLM pattern $s$ using gradient descent to minimize the total loss [S1].

**Formulas**
The opponent color channels are computed as [S1]:
$$
O_1 = 0.299 \cdot I_{\lambda_r} + 0.587 \cdot I_{\lambda_g} + 0.114 \cdot I_{\lambda_b}
$$
$$
O_2 = I_{\lambda_r} - I_{\lambda_g}
$$
$$
O_3 = I_{\lambda_b} - I_{\lambda_r} - I_{\lambda_g}
$$
where $O_1$ is luminance, $O_2$ is red-green chrominance, and $O_3$ is blue-yellow chrominance.

The total loss function is [S1]:
$$
\arg\min_s \sum_{i=1}^3 L(\hat{O}_i * k_i, O_i * k_i)
$$
where $*$ denotes 2D convolution, $\hat{O}_i$ and $O_i$ are the $i$-th opponent channels of the target and simulated hologram respectively, and $L$ is a pixel-wise loss function (typically MSE) [S1].

The spatial frequency cutoff in cycles per degree is calculated as [S1]:
$$
\text{cutoff} = \frac{f}{\pi} \cdot \frac{\gamma}{2p} \cdot \frac{180}{\pi}
$$
where $f$ is the focal length, $p$ is the pixel pitch, and $\gamma$ is the filter width fraction [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def opponent_color_transform(rgb_image):
    """
    Converts RGB image to opponent color space (Luminance, Red-Green, Blue-Yellow).
    Args:
        rgb_image: np.array of shape (H, W, 3) with values in [0, 1].
    Returns:
        opponent_channels: np.array of shape (H, W, 3) for O1, O2, O3.
    """
    r = rgb_image[:, :, 0]
    g = rgb_image[:, :, 1]
    b = rgb_image[:, :, 2]
    
    # Luminance
    o1 = 0.299 * r + 0.587 * g + 0.114 * b
    # Red-Green Chrominance
    o2 = r - g
    # Blue-Yellow Chrominance
    o3 = b - r - g
    
    return np.stack([o1, o2, o3], axis=-1)

def create_fourier_lowpass_filter(shape, cutoff_fraction):
    """
    Creates a 2D low-pass filter mask in the Fourier domain.
    Args:
        shape: Tuple (H, W) for the spatial domain size.
        cutoff_fraction: Float, fraction of the Fourier space width to keep (e.g., 0.75).
    Returns:
        filter_mask: np.array of shape (H, W) with values 0 or 1.
    """
    # Create coordinate grids centered at (0,0) in frequency space
    # Assuming standard FFT shift center
    h, w = shape
    y, x = np.ogrid[:h, :w]
    
    # Shift origin to center
    y = y - h // 2
    x = x - w // 2
    
    # Calculate distance from center in normalized frequency units
    # Using max(h, w) to normalize across aspect ratios if necessary, 
    # but typically circular filter uses radius relative to min dimension or specific axis.
    # The source implies a width of Fourier space. We assume a circular mask 
    # where radius is cutoff_fraction * (min(h,w)/2).
    max_radius = min(h, w) / 2.0
    radius = cutoff_fraction * max_radius
    
    distance = np.sqrt(x**2 + y**2)
    
    # Create mask: 1 inside radius, 0 outside
    filter_mask = (distance <= radius).astype(np.float32)
    
    return filter_mask

def perceptual_loss_step(target_rgb, simulated_rgb, cutoff_lum, cutoff_chrom):
    """
    Computes the perceptual loss between target and simulated holograms.
    Args:
        target_rgb: np.array (H, W, 3) target image.
        simulated_rgb: np.array (H, W, 3) simulated hologram output.
        cutoff_lum: Float, cutoff fraction for luminance (0.75).
        cutoff_chrom: Float, cutoff fraction for chrominance (0.45).
    Returns:
        loss: Scalar float value.
    """
    # 1. Transform to opponent color space
    target_opp = opponent_color_transform(target_rgb)
    sim_opp = opponent_color_transform(simulated_rgb)
    
    # 2. Create filters
    h, w = target_rgb.shape[:2]
    k_lum = create_fourier_lowpass_filter((h, w), cutoff_lum)
    k_chrom = create_fourier_lowpass_filter((h, w), cutoff_chrom)
    
    # 3. Apply filters via convolution
    # Note: In implementation, this is often done in Fourier domain for efficiency.
    # Here we simulate spatial convolution for clarity.
    from scipy.signal import fftconvolve
    
    # Filter Luminance (Channel 0)
    target_lum_filt = fftconvolve(target_opp[:,:,0], k_lum, mode='same')
    sim_lum_filt = fftconvolve(sim_opp[:,:,0], k_lum, mode='same')
    
    # Filter Chrominance (Channels 1 and 2)
    target_chrom_filt = fftconvolve(target_opp[:,:,1], k_chrom, mode='same')
    sim_chrom_filt = fftconvolve(sim_opp[:,:,1], k_chrom, mode='same')
    
    target_chrom2_filt = fftconvolve(target_opp[:,:,2], k_chrom, mode='same')
    sim_chrom2_filt = fftconvolve(sim_opp[:,:,2], k_chrom, mode='same')
    
    # 4. Compute MSE Loss
    loss_lum = np.mean((target_lum_filt - sim_lum_filt) ** 2)
    loss_chrom1 = np.mean((target_chrom_filt - sim_chrom_filt) ** 2)
    loss_chrom2 = np.mean((target_chrom2_filt - sim_chrom2_filt) ** 2)
    
    total_loss = loss_lum + loss_chrom1 + loss_chrom2
    
    return total_loss
```

**Edge Cases and Pitfalls**
*   **Precision:** The linear transformation coefficients (0.299, 0.587, etc.) are standard ITU-R BT.601 luminance weights. Ensure input RGB values are normalized consistently with these weights [S1].
*   **Boundary Conditions:** When applying convolution, boundary effects can introduce artifacts. Using 'same' mode padding or circular padding (via FFT) is recommended to maintain spatial alignment [S1].
*   **Degenerate Input:** If the target image is grayscale, chrominance channels will be zero. The loss will effectively ignore chrominance errors, which is consistent with HVS behavior [S1].

## Subtopics

### Simultaneous Color Focal Stack Optimization
Extends the simultaneous color approach to multiple depth planes (focal stacks) rather than a single plane. It uses the same perceptual loss principles but applies them across depth layers. [Link](17716-simultaneous-color-focal-stack-optimization.md)

### Phase Range Optimization for Simultaneous Color
Addresses the physical limitation of SLM phase range (e.g., $0-2\pi$ vs extended range) to reduce depth replicas. It complements the perceptual loss by improving the physical accuracy of the forward model. [Link](17722-phase-range-optimization-for-simultaneous-color.md)

## Operating characteristics
**Time and Memory Complexity**
*   **Time:** The addition of the perceptual loss increases computational cost per iteration compared to standard RGB loss due to the extra color space transformations and convolutions. However, the convolutions are low-pass and can be implemented efficiently in the Fourier domain. The overall convergence may be faster visually due to better gradient guidance, though exact iteration counts are not provided [S1].
*   **Memory:** Requires storing three additional channel arrays (opponent space) and filter masks. Memory overhead is negligible compared to the hologram simulation itself [S1].

**Measured Performance**
*   **Hardware/Setup:** Tested with an 8 µm pixel pitch SLM and a 36.6 mm focal length eyepiece [S1].
*   **Metrics:** On a test set of 294 images, the perceptual loss yielded an average PSNR increase of 6.4 dB and an average SSIM increase of 0.266 compared to standard RGB loss [S1].
*   **Visual Quality:** Holograms are visually less noisy with better color fidelity. The error is shifted into imperceptible regions of the opponent color space [S1].

**Strengths and Limitations**
*   **Strengths:** Significantly improves color fidelity and reduces noise in the perceived image. Aligns with human visual system characteristics (JPEG compression, subpixel rendering principles) [S1].
*   **Limitations:** The method assumes a specific human visual model (Mullen 1985) which may not account for all individual variations in color perception. It does not address depth replicas or SLM physical limitations directly (these are handled by sibling methods) [S1].

**When to Use**
*   Use when generating full-color holograms with a single SLM and simultaneous RGB illumination where color fidelity is critical.
*   Use when standard RGB loss results in desaturated or noisy color outputs.

**When Not to Use**
*   Use standard RGB loss if computational resources are extremely constrained and color fidelity is secondary to speed.
*   Use if the display system does not rely on human visual perception (e.g., machine vision applications) where spectral accuracy might be more important than perceptual uniformity [S1].

## Background
The method was introduced by Markley et al. in 2023 as part of the "Simultaneous Color Computer Generated Holography" framework [S1]. It leverages established principles from image compression (JPEG) and subpixel rendering, which utilize the HVS's lower sensitivity to high-frequency chrominance information [S1]. The specific cutoff frequencies (30 cycles/deg for luminance, 18 cycles/deg for chrominance) are based on human vision measurements by Mullen (1985) [S1]. This approach was novel in its application to hologram optimization, addressing the overdetermined nature of single-SLM color holography [S1].

## Sources
[S1] Simultaneous Color Computer Generated Holography, https://doi.org/10.48550/arxiv.2303.11287, fetched 2026-10-30
