# PSF Modeling

Also called: Fonction d’étalement du point, Gaussian Gain Calculation, PSF Estimation, Point Spread Function (PSF) Modeling

## Overview
PSF Modeling characterizes the spatial response of an imaging or rendering system to a point source (Dirac pulse) to quantify image sharpness and reconstruction precision [S1, unofficial]. It differs from sibling methods like the [Narrow beam distribution method](9069-narrow-beam-distribution-method.md) by explicitly fitting a parametric function to the blurred point source profile rather than analyzing beam attenuation or distribution directly.

## How it works
**Inputs:** Reconstructed images containing a point source (Dirac pulse) at known coordinates $(t_x, t_y)$.
**Outputs:** PSF parameters ($F_{max}, \sigma_x, \sigma_y, d_x, d_y, B$) and Gaussian Gain $G(x,y)$.

**Data Structures:**
*   **Image:** 2D array of pixel intensities.
*   **PSF Model:** A separable function combining Gaussian decay and cosine oscillation.

**Steps:**
1.  **Acquire Point Source Data:** Capture or reconstruct an image of a Dirac pulse (a single non-zero intensity pixel or idealized point) at position $(t_x, t_y)$ [S1, unofficial].
2.  **Estimate Initial Standard Deviations:** Measure the Full Width at Half Maximum (FWHM) $H_x$ and $H_y$ along the x and y axes from the reconstructed image profile [S1, unofficial]. Calculate initial $\sigma$ estimates:
    $$ \sigma_x = \frac{H_x}{2\sqrt{2\ln(2)}} $$
    $$ \sigma_y = \frac{H_y}{2\sqrt{2\ln(2)}} $$
3.  **Refine Parameters via Non-Linear Least Squares:** Use a Newton-Gauss iterative algorithm with separated data to fit the PSF model to the measured pixel intensities [S1, unofficial]. The model to fit is:
    $$ PSF(x, y) = F_{max} e^{-\frac{(x-t_x)^2}{2\sigma_x^2}} \cos(d_x(x-t_x)) e^{-\frac{(y-t_y)^2}{2\sigma_y^2}} \cos(d_y(y-t_y)) + B $$
    Where:
    *   $(t_x, t_y)$: Coordinates of the Dirac pulse center.
    *   $F_{max}$: Maximum intensity $f(t_x, t_y)$.
    *   $\sigma_x, \sigma_y$: Spread shape parameters (standard deviations).
    *   $B$: Background intensity.
    *   $d_x, d_y$: Frequency coefficients for positive/negative transitions [S1, unofficial].
4.  **Calculate Gaussian Gain:** For any point $(x, y)$, compute the sharpness gain:
    $$ G(x, y) = \frac{1}{\sigma_x(x, y)\sigma_y(x, y)} $$
    Higher gain indicates smaller standard deviations, less point spread, and higher sharpness [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.optimize import curve_fit

def psf_model(xy, F_max, sigma_x, d_x, sigma_y, d_y, B, tx, ty):
    """
    Separable PSF model: Gaussian envelope modulated by cosine, shifted by (tx, ty).
    xy is a tuple (x_coords, y_coords) or meshgrid.
    """
    x, y = xy
    # Gaussian terms
    gauss_x = np.exp(-((x - tx)**2) / (2 * sigma_x**2))
    gauss_y = np.exp(-((y - ty)**2) / (2 * sigma_y**2))
    # Cosine modulation terms
    cos_x = np.cos(d_x * (x - tx))
    cos_y = np.cos(d_y * (y - ty))
    
    return F_max * gauss_x * cos_x * gauss_y * cos_y + B

def estimate_psf_parameters(image, tx, ty):
    """
    Estimate PSF parameters from a reconstructed image containing a point source.
    image: 2D numpy array.
    tx, ty: Integer coordinates of the point source center.
    """
    # 1. Extract profile around the point source
    # Define a region of interest (ROI) large enough to capture the spread
    roi = image[int(tx)-50:int(tx)+51, int(ty)-50:int(ty)+51]
    
    # 2. Initial estimation of sigma using FWHM
    # Find max intensity in ROI
    max_val = np.max(roi)
    half_max = max_val / 2.0
    
    # Find FWHM along x and y axes (simplified 1D extraction)
    # Note: In practice, this requires finding the width at half-max on the profile
    # This is a placeholder for the specific FWHM calculation logic
    H_x = 10.0 # Placeholder: Measure FWHM from x-profile
    H_y = 10.0 # Placeholder: Measure FWHM from y-profile
    
    sigma_x_init = H_x / (2 * np.sqrt(2 * np.log(2)))
    sigma_y_init = H_y / (2 * np.sqrt(2 * np.log(2)))
    
    # 3. Non-linear least squares fitting
    # Create meshgrid for the ROI
    y_indices, x_indices = np.mgrid[0:roi.shape[0], 0:roi.shape[1]]
    # Adjust coordinates to be relative to (tx, ty) if needed, or absolute
    # Assuming ROI is centered at (tx, ty) in the original image, 
    # we map ROI indices to image coordinates.
    x_coords = x_indices + (int(tx) - 50)
    y_coords = y_indices + (int(ty) - 50)
    
    # Flatten for curve_fit
    x_flat = x_coords.flatten()
    y_flat = y_coords.flatten()
    z_flat = roi.flatten()
    
    # Initial guess: [F_max, sigma_x, d_x, sigma_y, d_y, B]
    # tx, ty are fixed knowns
    p0 = [max_val, sigma_x_init, 0.1, sigma_y_init, 0.1, np.min(roi)]
    
    try:
        popt, _ = curve_fit(psf_model, (x_flat, y_flat), z_flat, p0=p0)
        F_max, sigma_x, d_x, sigma_y, d_y, B = popt
    except RuntimeError:
        # Fallback to initial estimates if fitting fails
        F_max, sigma_x, d_x, sigma_y, d_y, B = max_val, sigma_x_init, 0.1, sigma_y_init, 0.1, np.min(roi)

    return {
        'F_max': F_max,
        'sigma_x': sigma_x,
        'sigma_y': sigma_y,
        'd_x': d_x,
        'd_y': d_y,
        'B': B,
        'tx': tx,
        'ty': ty
    }

def calculate_gaussian_gain(sigma_x, sigma_y):
    """
    Calculate Gaussian Gain G(x,y) = 1 / (sigma_x * sigma_y).
    """
    return 1.0 / (sigma_x * sigma_y)
```

**Parameters & Typical Values:**
*   **Image Size:** 512x512 pixels [S1, unofficial].
*   **Gaussian Gain (Center):** 5.01 for Back Projection ($R^{-1}$); 8.39 for Fourier Reconstruction ($F_R$) [S1, unofficial].
*   **Step Size for PSF Variation Study:** 5 pixels [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Spatial Variability:** The PSF is not constant across the image. Gain decreases near the edges of the circle of interest, becoming equivalent to Back Projection gain at the extremities [S1, unofficial]. This variability makes global deconvolution difficult.
*   **Precision:** The model assumes a specific separable form (Gaussian $\times$ Cosine). Deviations from this shape (e.g., non-separable artifacts) may reduce fit accuracy.

## Operating characteristics
*   **Complexity:** The fitting step involves non-linear least squares (Newton-Gauss), which is computationally more expensive than simple FWHM measurement but necessary for accurate parameter extraction.
*   **Strengths:** Provides a quantitative metric (Gaussian Gain) for comparing reconstruction algorithms (e.g., Fourier vs. Back Projection) and assessing local sharpness.
*   **Limitations:** The spatial non-stationarity of the PSF limits its utility for simple global deconvolution. It is primarily an analytical tool for quality assessment rather than a real-time rendering kernel itself.

## Background
The Point Spread Function (PSF) is a fundamental concept in imaging systems, describing the response to a point source (Dirac pulse) [S1, unofficial]. In the context of tomographic reconstruction, it is used to estimate the sharpness and precision of computed images. The specific parametric model combining Gaussian spread and cosine modulation was detailed in studies comparing Radon Back Projection ($R^{-1}$) and Fourier Reconstruction ($F_R$) [S1, unofficial].

## Sources
[S1] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
