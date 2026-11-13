# Aperture Apodization

Also called: Hanning window apodization, Suppression of aperture diffraction, Synthetic Aperture Selection & Apodization

## Overview
Aperture Apodization is a technique used in numerical holographic reconstruction to suppress side-lobe leakage and diffraction artifacts caused by the sharp intensity edges of a rectangular synthetic aperture [S2]. It works by smoothly tapering the aperture edges to zero, thereby reducing strong diffraction effects at the borders that would otherwise occur during propagation [S1, S2]. It is applied after selecting a sub-region of the hologram (synthetic aperture) to define a specific viewpoint [S1].

## How it works
**Inputs:**
- Complex hologram matrix $H$ of size $N \times M$.
- Aperture definition parameters: center coordinates $(x_c, y_c)$, reconstruction distance $d_r$, and aperture dimension angle $\psi$ [S1].
- Window parameter: Full Width at Half Maximum $\alpha$ [S2].

**Outputs:**
- Apodized synthetic aperture matrix $A_{apo}$ of size $N \times M$ (zero-padded to original hologram size) [S2].

**Data Structures:**
- The synthetic aperture is a 2D sub-region of the hologram.
- The Hanning window is a 2D mask applied element-wise to the aperture.

**Steps:**
1. **Define Synthetic Aperture:**
   Calculate the horizontal and vertical aperture angles $\theta$ and $\phi$ using the center coordinates $(x_c, y_c)$ and reconstruction distance $d_r$ [S1]:
   $$ \theta = \arctan\left(\frac{x_c}{d_r}\right) $$
   $$ \phi = \arctan\left(\frac{y_c}{d_r}\right) $$
   The aperture dimension is controlled by the angle $\psi$ [S1]. The specific pixel coordinates defining the rectangular sub-region are determined by these angles and the hologram resolution [S1].

2. **Extract Sub-Region:**
   Extract the rectangular region from the hologram $H$ defined by the calculated boundaries. Let this sub-region be $A_{sub}$ with dimensions $W \times H$ [S2].

3. **Generate Hanning Window:**
   Create a 1D Hanning window function $h_n(x)$ [S2]:
   $$ h_n(x) = \cos^2\left(\frac{\pi x}{2\alpha}\right) $$
   where $\alpha$ is the Full Width at Half Maximum (FWHM) [S2].
   *Note:* The source defines a 1D function but applies a "bidimensional Hanning window" [S1]. The 2D window $W_{2D}(u, v)$ is constructed by applying the 1D function along both axes. For a window of width $W$, the coordinate $x$ typically ranges from $-W/2$ to $W/2$ or $0$ to $W$ depending on implementation, centered at the aperture [S2].

4. **Apply Apodization:**
   Multiply the extracted sub-region $A_{sub}$ element-wise by the 2D Hanning window $W_{2D}$:
   $$ A_{apo\_sub} = A_{sub} \odot W_{2D} $$
   This smoothly brings the aperture edges to zero [S2].

5. **Zero-Pad:**
   Zero-pad the apodized sub-region $A_{apo\_sub}$ to the original hologram size $N \times M$ to create the final apodized aperture matrix $A_{apo}$ [S2].

**Parameters:**
- $\alpha$: Full Width at Half Maximum of the Hanning window. Not given in the sources.
- $\psi$: Aperture dimension angle. Not given in the sources.
- $d_r$: Reconstruction distance. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Resolution Loss:** Apodization reduces resolution due to the limited signal area [S2].
- **Precision:** Ensure the window function is correctly normalized or scaled if amplitude preservation is critical, though the source does not specify normalization [S2].
- **Boundary Alignment:** The window must be correctly aligned with the extracted sub-region to ensure smooth tapering at the correct edges [S2].

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
    # Coordinate system centered at 0
    x = np.linspace(-length / 2, length / 2, length)
    # Formula from [S2]: h_n(x) = cos^2(pi * x / (2 * alpha))
    window_1d = np.cos(np.pi * x / (2 * alpha)) ** 2
    return window_1d

def aperture_apodization(hologram, center_x, center_y, aperture_width, aperture_height, alpha, reconstruction_distance):
    """
    Applies aperture apodization to a hologram.
    :param hologram: 2D complex numpy array (N, M).
    :param center_x: x-coordinate of aperture center.
    :param center_y: y-coordinate of aperture center.
    :param aperture_width: Width of the synthetic aperture in pixels.
    :param aperture_height: Height of the synthetic aperture in pixels.
    :param alpha: FWHM for the Hanning window.
    :param reconstruction_distance: Distance dr for angle calculation (used for positioning if needed, 
                                    but here we assume coordinates are given directly).
    :return: Apodized hologram (N, M) complex array.
    """
    N, M = hologram.shape
    
    # 1. Define Synthetic Aperture boundaries
    # Assuming center is in pixel coordinates
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
    # Note: Source [S1] uses angles to define position, but [S2] implies direct extraction.
    # We assume the user provides the pixel dimensions derived from angles.
    aperture_sub = hologram[y_start:y_end, x_start:x_end]
    
    # 3. Generate 2D Hanning Window
    # Source [S2] defines 1D window. Source [S1] mentions bidimensional.
    # We construct 2D by outer product of 1D windows.
    win_w = generate_hanning_window_1d(aperture_sub.shape[1], alpha)
    win_h = generate_hanning_window_1d(aperture_sub.shape[0], alpha)
    
    # Create 2D window
    window_2d = np.outer(win_h, win_w)
    
    # 4. Apply Apodization
    apodized_sub = aperture_sub * window_2d
    
    # 5. Zero-Pad to original size
    apodized_hologram = np.zeros((N, M), dtype=complex)
    apodized_hologram[y_start:y_end, x_start:x_end] = apodized_sub
    
    return apodized_hologram
```

## Subtopics

### Synthetic Aperture Selection
Defines the sub-region of the hologram to simulate a limited optical aperture. It controls the viewpoint. [Link](52-synthetic-aperture-selection.md)

### Numerical Reconstruction
The subsequent step that propagates the apodized aperture to reconstruct the image. [Link](53-numerical-reconstruction.md)

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels in the synthetic aperture $O(W \times H)$ for window generation and multiplication. Zero-padding is $O(N \times M)$.
- **Memory Complexity:** $O(N \times M)$ to store the zero-padded result.
- **Strengths:** Effectively suppresses side-lobe leakage and aperture diffraction artifacts [S2].
- **Limitations:** Causes a decrease in resolution due to the limited signal area [S2].
- **When to use:** When sharp rectangular aperture edges cause significant diffraction artifacts in the reconstruction [S2].
- **When not to use:** When maximum resolution is required and diffraction artifacts are acceptable or handled by other means [S2].

## Background
The technique was proposed to address aperture diffraction issues in numerical reconstruction of computer-generated holograms [S2]. It builds on the concept of synthetic aperture selection [S1]. The Hanning window function is a standard signal processing tool adapted for this optical context [S2]. The method is part of a framework for rendering 3D holograms from point clouds on light field displays [S2].

## Sources
[S1] Recent Advances in the Processing and Rendering Algorithms for Computer-Generated Holography, https://doi.org/10.3390/electronics8050556, fetched 2026-10-26
[S2] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
