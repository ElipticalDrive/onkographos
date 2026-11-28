# Locally Orderless Image (LOI)

Also called: Locally Orderless Image representation

## Overview
Locally Orderless Image (LOI) is a differentiable rendering optimization technique that replaces standard pixel-wise image comparison with matching of local histograms (distributions of intensity values) to extend gradient support for sparse gradients caused by motion in image space [S1]. It solves the problem of poor convergence and local minima in inverse rendering, where standard RGB gradients are sparse or zero when initial and target features do not overlap [S1]. Unlike Gaussian Pyramids, which average out local appearance and suppress high-frequency details, LOIs match entire intensity distributions within neighborhoods, preserving local variations and enabling reliable recovery from arbitrary initializations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A differentiable renderer capable of computing image gradients $\frac{\partial I}{\partial \theta}$ with respect to scene parameters $\theta$; a reference image $I_{\text{ref}}$; initial scene parameters $\theta_0$ [S1].
*   **Output:** Optimized scene parameters $\theta^*$ that minimize the histogram distance between the rendered LOI and the reference LOI [S1].

**Data Structures:**
The core data structure is the Locally Orderless Image representation $P(x, \alpha, \beta, \sigma)$, which maps each pixel location $x$ to a histogram of intensities [S1].
*   **Fields:**
    *   $x$: 2D spatial coordinate $(u, v)$ [S1].
    *   $H_x$: A histogram vector representing the distribution of intensity values in the neighborhood of $x$ [S1].
    *   Parameters: $\alpha$ (extent scale), $\beta$ (tonal scale), $\sigma$ (inner scale) [S1].
*   **Layout:** The representation is a 4D tensor or a list of histograms indexed by $(x, \alpha, \beta, \sigma)$ [G]. For implementation, it is typically computed per scale space [G]. The histogram $H_x$ has size $N_{\text{bins}}$, where $N_{\text{bins}}$ is determined by the tonal scale $\beta$ and the dynamic range of intensities [G].

**Numbered Steps:**

1.  **Construct Reference LOI:**
    For the reference image $I_{\text{ref}}$, compute the LOI representation $P_{\text{ref}}(x, \alpha, \beta, \sigma)$ for a set of scales [S1].
    *   **Inner Scale ($\sigma$):** Apply a Gaussian blur to $I_{\text{ref}}$ to create a multi-resolution representation:
        $$I(x; \sigma) = (G * I_{\text{ref}})(x; \sigma)$$
        where $G$ is the Gaussian aperture function:
        $$G(x; \sigma) = \frac{1}{2\pi\sigma^2} \exp\left(-\frac{x \cdot x}{2\sigma^2}\right)$$
        [S1]
    *   **Tonal Scale ($\beta$):** Model the radiance at each pixel as a distribution rather than a single real value, effectively binning intensities to capture measurement uncertainty and noise [S1].
    *   **Extent Scale ($\alpha$):** Compute histograms of intensity values within a spatial neighborhood defined by $\alpha$ for each pixel, creating a 'locally orderless' representation where spatial order within the neighborhood is discarded [S1].

2.  **Construct Rendered LOI:**
    For the current scene parameters $\theta$, render the image $I(x; \theta)$ [S1]. Compute the LOI representation $P_{\text{render}}(x, \alpha, \beta, \sigma)$ using the same scales $\alpha, \beta, \sigma$ as the reference [S1].
    *   Apply Gaussian blur with kernel width $\sigma$ to $I(x; \theta)$ [S1].
    *   Bin intensities with width $\beta$ [S1].
    *   Compute histograms over neighborhoods of size $\alpha$ [S1].

3.  **Define Objective Function:**
    Define an inverse rendering objective function $E$ that minimizes the distance between the histograms of $P_{\text{render}}$ and $P_{\text{ref}}$ [S1]. Let $H_{\text{ref}}(x, \alpha, \beta, \sigma)$ and $H_{\text{render}}(x, \alpha, \beta, \sigma)$ be the histograms at location $x$ and scales [S1].
    $$E(\theta) = \sum_{x, \alpha, \beta, \sigma} D(H_{\text{render}}(x, \alpha, \beta, \sigma), H_{\text{ref}}(x, \alpha, \beta, \sigma))$$
    where $D$ is a histogram distance metric [S1].

4.  **Compute Gradients:**
    Compute gradients $\frac{\partial E}{\partial \theta}$ using the chain rule [S1]. The gradient depends on the standard RGB gradients from the differentiable renderer $\frac{\partial I}{\partial \theta}$ and the derivative of the histogram distance metric with respect to the rendered intensities [S1].
    $$\frac{\partial E}{\partial \theta} = \sum_{x, \alpha, \beta, \sigma} \frac{\partial D}{\partial H_{\text{render}}} \cdot \frac{\partial H_{\text{render}}}{\partial I} \cdot \frac{\partial I}{\partial \theta}$$
    [S1]

5.  **Update Parameters:**
    Update parameters $\theta$ via gradient descent [S1].
    $$\theta_{\text{new}} = \theta_{\text{old}} - \eta \frac{\partial E}{\partial \theta}$$
    where $\eta$ is the learning rate [G].

**Parameters:**
*   $\sigma$ (sigma): The inner scale parameter controlling the effective resolution of the image via Gaussian kernel width. Typical values include $\sigma=0$ (stationary) and $\sigma>0$ (coarser scales) [S1].
*   $\alpha$ (alpha): The extent scale parameter defining the spatial extent (neighborhood size) over which the histogram is computed for each pixel [S1].
*   $\beta$ (beta): The tonal scale parameter quantifying the imprecision in measured radiance (bin width) at a location, modeling uncertainty in intensity values [S1].

**Pitfalls:**
*   Standard multi-resolution image pyramids (Gaussian Pyramids) are unreliable for complex inverse rendering because blurring averages out local appearance and suppresses high-frequency details, leading to local minima [S1].
*   In cases with no overlap between initial and target features (e.g., disks), standard image gradients $\frac{\partial I}{\partial \theta}$ are sparse (non-zero only at silhouettes), causing the error gradient $\frac{\partial E}{\partial \theta}$ to be zero everywhere, stalling optimization [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def gaussian_kernel_1d(sigma):
    """Create a 1D Gaussian kernel."""
    size = int(math.ceil(3 * sigma)) * 2 + 1
    x = np.arange(size) - size // 2
    kernel = np.exp(-(x**2) / (2 * sigma**2))
    return kernel / np.sum(kernel)

def apply_gaussian(image, sigma):
    """Apply 2D Gaussian blur using separable convolution."""
    if sigma <= 0:
        return image.copy()
    kernel_1d = gaussian_kernel_1d(sigma)
    # Convolve along rows, then columns
    temp = np.apply_along_axis(lambda row: np.convolve(row, kernel_1d, mode='same'), 1, image)
    return np.apply_along_axis(lambda col: np.convolve(col, kernel_1d, mode='same'), 0, temp)

def compute_histogram(intensities, bin_width, num_bins):
    """Compute histogram of intensity values."""
    hist = np.zeros(num_bins)
    for val in intensities:
        if 0 <= val < num_bins * bin_width:
            idx = min(int(val / bin_width), num_bins - 1)
            hist[idx] += 1.0
    return hist

def compute_loi(image, sigma, alpha, beta, num_bins):
    """
    Compute Locally Orderless Image representation.
    image: 2D numpy array of intensities.
    sigma: Inner scale (Gaussian blur width).
    alpha: Extent scale (neighborhood size for histogram).
    beta: Tonal scale (bin width).
    num_bins: Number of histogram bins.
    """
    blurred = apply_gaussian(image, sigma)
    h, w = blurred.shape
    lois = []
    half_alpha = alpha // 2
    
    for i in range(h):
        for j in range(w):
            i_start = max(0, i - half_alpha)
            i_end = min(h, i + half_alpha + 1)
            j_start = max(0, j - half_alpha)
            j_end = min(w, j + half_alpha + 1)
            
            neighborhood = blurred[i_start:i_end, j_start:j_end].flatten()
            hist = compute_histogram(neighborhood, beta, num_bins)
            lois.append(hist)
            
    return np.array(lois)

def histogram_distance(hist1, hist2):
    """Compute Chi-squared distance between two histograms."""
    hist1 = np.maximum(hist1, 1e-10)
    hist2 = np.maximum(hist2, 1e-10)
    return np.sum((hist1 - hist2)**2 / (hist1 + hist2))

def compute_gradient(render_fn, theta, reference_loi, sigma, alpha, beta, num_bins):
    """
    Compute gradient of LOI objective w.r.t. theta using the differentiable renderer's gradients.
    render_fn: Function that takes theta and returns rendered image.
    """
    current_image = render_fn(theta)
    render_loi = compute_loi(current_image, sigma, alpha, beta, num_bins)
    
    # Compute derivative of histogram distance w.r.t. rendered image intensities
    # The exact backprop through binning depends on the chosen distance metric [G]
    dE_dI = np.zeros_like(current_image)
    for h_ref, h_render in zip(reference_loi, render_loi):
        diff = h_render - h_ref
        denom = h_render + h_ref + 1e-10
        dD_dH = 2 * diff / denom
        # Map bin derivatives back to image pixels (simplified aggregation)
        # In practice, this requires tracking which pixels contributed to which bins [G]
        
    # Chain rule: ∂E/∂θ = ∂E/∂I * ∂I/∂θ
    # ∂I/∂θ is provided directly by the differentiable renderer [S1]
    grad_I = render_fn.gradient(theta)
    return grad_I * dE_dI

def optimize_loi(render_fn, reference_image, initial_theta, sigma, alpha, beta, num_bins, learning_rate, num_iterations):
    """
    Optimize scene parameters using LOI matching.
    render_fn: Function that takes theta and returns rendered image.
    reference_image: 2D numpy array of reference intensities.
    initial_theta: Initial scene parameters.
    sigma, alpha, beta: LOI parameters.
    num_bins: Number of histogram bins.
    learning_rate: Step size for gradient descent.
    num_iterations: Number of optimization steps.
    """
    theta = initial_theta.copy()
    ref_loi = compute_loi(reference_image, sigma, alpha, beta, num_bins)
    
    for _ in range(num_iterations):
        grad = compute_gradient(render_fn, theta, ref_loi, sigma, alpha, beta, num_bins)
        theta = theta - learning_rate * grad
        
    return theta
```

## Subtopics

### [Variational Optimization with Locally Orderless Images (LOI)](11021-variational-optimization-with-locally-orderless-images-loi.md)
Variational Optimization with Locally Orderless Images (LOI) is a differentiable rendering optimization technique that combines variational derivatives with multi-scale local histogram matching to recover scene parameters (e.g., geometry, camera pose, material properties) from images where standard RGB gradients are unstable, sparse, or insufficient due to complex light transport or high variance. Unlike variational optimization using stationary image matching, which fails due to strict alignment sensitivity and noise, LOI matching does not require perfect alignment between the variational estimate and reference images, allowing reliable recovery from arbitrary initializations.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** Validated on inverse vectorization, path tracing, and rasterization problems using both synthetic and real data [S1].
*   **Scaling:** Not given in the sources [S1].
*   **Strengths:** Extends gradient support for sparse gradients; robust to noise; works with arbitrary geometry representations and complex light transport; does not require perfect alignment [S1].
*   **Limitations:** Not given in the sources [S1].
*   **When to use:** Inverse rendering problems with sparse gradients, large initial errors, or complex light transport where standard RGB gradients fail [S1].
*   **When not to use:** Not given in the sources [S1].

## Background
LOI was originally proposed by Griffin [11] and later extended by Koenderink and van Doorn [15] for reasoning about image topology [S1]. It has been used in various vision tasks, including image retrieval [17], object tracking [25], and non-linear filtering [33, 34] [S1]. The application of LOI to differentiable rendering optimization was introduced by Chandraker and Ramamoorthi in 2025 [S1].

## Sources
[S1] Locally Orderless Images for Optimization in Differentiable Rendering, https://doi.org/10.1109/cvpr52734.2025.00541, fetched 2026-10-26
