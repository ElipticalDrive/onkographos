# Aperture Apodization

Also called: Hann Window, Hanning Window (incorrectly used), Hanning window apodization, Suppression of aperture diffraction, Synthetic Aperture Selection & Apodization

## Overview
Aperture Apodization suppresses side-lobe leakage and aperture diffraction artifacts caused by the sharp intensity edges of a rectangular synthetic aperture during numerical holographic reconstruction [S2]. It smoothly tapers the aperture edges to zero, reducing strong diffraction effects at the borders [S1, S2]. The method is applied after selecting a sub-region of the hologram to define a specific viewpoint [S1].

## How it works
**Inputs:**
- Complex hologram matrix $H$ of size $N \times M$ [S1].
- Aperture definition parameters: center coordinates $(x_c, y_c)$, reconstruction distance $d_r$, and aperture dimension angle $\psi$ [S1].
- Window parameter: Full Width at Half Maximum $\alpha$ [S2].

**Outputs:**
- Apodized synthetic aperture matrix $A_{apo}$ of size $N \times M$ (zero-padded to original hologram size) [S2].

**Data Structures:**
- The synthetic aperture is a 2D sub-region of the hologram [S1].
- The Hanning window is a 2D mask applied element-wise to the aperture [S1].

**Steps:**
1. **Define Synthetic Aperture:** Calculate the horizontal and vertical aperture angles $\theta$ and $\phi$ using the center coordinates $(x_c, y_c)$ and reconstruction distance $d_r$ [S1]:
   $$ \theta = \arctan\left(\frac{x_c}{d_r}\right) $$
   $$ \phi = \arctan\left(\frac{y_c}{d_r}\right) $$
   The aperture dimension is controlled by the angle $\psi$ [S1]. The specific pixel coordinates defining the rectangular sub-region are determined by these angles and the hologram resolution [S1].
2. **Extract Sub-Region:** Extract the rectangular region from the hologram $H$ defined by the calculated boundaries. Let this sub-region be $A_{sub}$ with dimensions $W \times H$ [S1].
3. **Generate Hanning Window:** Create a 1D Hanning window function $h_n(x)$ [S2]:
   $$ h_n(x) = \cos^2\left(\frac{\pi x}{2\alpha}\right) $$
   where $\alpha$ is the Full Width at Half Maximum (FWHM) [S2].
4. **Construct 2D Window:** The 2D window $W_{2D}(u, v)$ is constructed by applying the 1D function along both axes [G].
5. **Apply Apodization:** Multiply the extracted sub-region $A_{sub}$ element-wise by the 2D Hanning window $W_{2D}$:
   $$ A_{apo\_sub} = A_{sub} \odot W_{2D} $$
   This smoothly brings the aperture edges to zero [S2].
6. **Zero-Pad:** Zero-pad the apodized sub-region $A_{apo\_sub}$ to the original hologram size $N \times M$ to create the final apodized aperture matrix $A_{apo}$ [S2].

**Parameters:**
- $\alpha$: Full Width at Half Maximum of the Hanning window. Not given in the sources.
- $\psi$: Aperture dimension angle. Not given in the sources.
- $d_r$: Reconstruction distance. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Resolution Loss:** Apodization reduces resolution due to the limited signal area [S2].
- **Precision:** Ensure the window function is correctly normalized or scaled if amplitude preservation is critical, though the source does not specify normalization [G].
- **Boundary Alignment:** The window must be correctly aligned with the extracted sub-region to ensure smooth tapering at the correct edges [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_hanning_window_1d(length, alpha):
    """
    Generates a 1D Hanning window.
    :param length: Length of the window.
    :param alpha: Full Width at Half Maximum (FWHM).
    :return: 1D numpy array.
    """
    # Coordinate system ranges from 0 to length [G]
    x = np.linspace(0, length, length)
    # Formula from [S2]: h_n(x) = cos^2(pi * x / (2 * alpha))
    window_1d = np.cos(np.pi * x / (2 * alpha)) ** 2
    return window_1d

def aperture_apodization(hologram, center_x, center_y, aperture_width, aperture_height, alpha):
    """
    Applies aperture apodization to a hologram.
    :param hologram: 2D complex numpy array (N, M).
    :param center_x: x-coordinate of aperture center.
    :param center_y: y-coordinate of aperture center.
    :param aperture_width: Width of the synthetic aperture in pixels.
    :param aperture_height: Height of the synthetic aperture in pixels.
    :param alpha: FWHM for the Hanning window.
    :return: Apodized hologram (N, M) complex array.
    """
    N, M = hologram.shape
    
    # 1. Define Synthetic Aperture boundaries
    x_start = int(center_x - aperture_width / 2)
    x_end = int(center_x + aperture_width / 2)
    y_start = int(center_y - aperture_height / 2)
    y_end = int(center_y + aperture_height / 2)
    
    # Clamp to hologram boundaries
    x_start = max(0, x_start)
    x_end = min(M, x_end)
    y_start = max(0, y_start)
    y_end = min(N, y_end)
    
    # 2. Extract Sub-Region
    aperture_sub = hologram[y_start:y_end, x_start:x_end]
    
    # 3. Generate 2D Hanning Window
    win_w = generate_hanning_window_1d(aperture_sub.shape[1], alpha)
    win_h = generate_hanning_window_1d(aperture_sub.shape[0], alpha)
    
    # Create 2D window by outer product
    window_2d = np.outer(win_h, win_w)
    
    # 4. Apply Apodization
    apodized_sub = aperture_sub * window_2d
    
    # 5. Zero-Pad to original size
    apodized_hologram = np.zeros((N, M), dtype=complex)
    apodized_hologram[y_start:y_end, x_start:x_end] = apodized_sub
    
    return apodized_hologram
```

## Subtopics

### Hanning Window
A window function used to multiply the ramp filter to attenuate high-frequency noise [S3, unofficial]. The Hann variant uses parameter $a = 0.5$, while the Hamming variant uses $a = 0.54$ [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels in the synthetic aperture $O(W \times H)$ for window generation and multiplication. Zero-padding is $O(N \times M)$ [G].
- **Memory Complexity:** $O(N \times M)$ to store the zero-padded result [G].
- **Strengths:** Effectively suppresses side-lobe leakage and aperture diffraction artifacts [S2].
- **Limitations:** Causes a decrease in resolution due to the limited signal area [S2].
- **When to use:** When sharp rectangular aperture edges cause significant diffraction artifacts in the reconstruction [S2].

## Background
The technique was proposed to address aperture diffraction issues in numerical reconstruction of computer-generated holograms [S2]. It builds on the concept of synthetic aperture selection [S1]. The Hanning window function is a standard signal processing tool adapted for this optical context [S2].

## Sources
[S1] Recent Advances in the Processing and Rendering Algorithms for Computer-Generated Holography, https://doi.org/10.3390/electronics8050556, fetched 2026-10-26
[S2] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
[S3] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
