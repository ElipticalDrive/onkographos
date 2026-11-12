# Locally Orderless Image (LOI)

Also called: Locally Orderless Image representation

## Overview
Locally Orderless Image (LOI) is a differentiable rendering optimization technique that replaces standard pixel-wise image comparison with matching of local histograms (distributions of intensity values) to extend gradient support for sparse gradients caused by motion in image space [S1]. It solves the problem of poor convergence and local minima in inverse rendering, where standard RGB gradients are sparse or zero when initial and target features do not overlap [S1]. Unlike Gaussian Pyramids, which average out local appearance and suppress high-frequency details, LOIs match entire intensity distributions within neighborhoods, preserving local variations and enabling reliable recovery from arbitrary initializations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A differentiable renderer capable of computing image gradients $\frac{\partial I}{\partial \theta}$ with respect to scene parameters $\theta$; a reference image $I_{ref}$; initial scene parameters $\theta_0$.
*   **Output:** Optimized scene parameters $\theta^*$ that minimize the histogram distance between the rendered LOI and the reference LOI.

**Data Structures:**
The core data structure is the Locally Orderless Image representation $P(x, \alpha, \beta, \sigma)$, which maps each pixel location $x$ to a histogram of intensities.
*   **Fields:**
    *   $x$: 2D spatial coordinate $(u, v)$.
    *   $H_x$: A histogram vector representing the distribution of intensity values in the neighborhood of $x$.
    *   Parameters: $\alpha$ (extent scale), $\beta$ (tonal scale), $\sigma$ (inner scale).
*   **Layout:** The representation is a 4D tensor or a list of histograms indexed by $(x, \alpha, \beta, \sigma)$. For implementation, it is typically computed per scale space. The histogram $H_x$ has size $N_{bins}$, where $N_{bins}$ is determined by the tonal scale $\beta$ and the dynamic range of intensities.

**Numbered Steps:**

1.  **Construct Reference LOI:**
    For the reference image $I_{ref}$, compute the LOI representation $P_{ref}(x, \alpha, \beta, \sigma)$ for a set of scales.
    *   **Inner Scale ($\sigma$):** Apply a Gaussian blur to $I_{ref}$ to create a multi-resolution representation:
        $$I(x; \sigma) = (G * I_{ref})(x; \sigma)$$
        where $G$ is the Gaussian aperture function:
        $$G(x; \sigma) = \frac{1}{2\pi\sigma^2} \exp\left(-\frac{x \cdot x}{2\sigma^2}\right)$$
        [S1]
    *   **Tonal Scale ($\beta$):** Model the radiance at each pixel as a distribution. Bin the intensity values with bin width $\beta$ to capture measurement uncertainty and noise. This transforms the real-valued intensity into a probability distribution or histogram. [S1]
    *   **Extent Scale ($\alpha$):** For each pixel $x$, compute the histogram of intensity values within a spatial neighborhood defined by $\alpha$. This discards spatial order within the neighborhood, creating the "locally orderless" property. [S1]

2.  **Construct Rendered LOI:**
    For the current scene parameters $\theta$, render the image $I(x; \theta)$. Compute the LOI representation $P_{render}(x, \alpha, \beta, \sigma)$ using the same scales $\alpha, \beta, \sigma$ as the reference.
    *   Apply Gaussian blur with kernel width $\sigma$ to $I(x; \theta)$.
    *   Bin intensities with width $\beta$.
    *   Compute histograms over neighborhoods of size $\alpha$. [S1]

3.  **Define Objective Function:**
    Define an inverse rendering objective function $E$ that minimizes the distance between the histograms of $P_{render}$ and $P_{ref}$. Let $H_{ref}(x, \alpha, \beta, \sigma)$ and $H_{render}(x, \alpha, \beta, \sigma)$ be the histograms at location $x$ and scales.
    $$E(\theta) = \sum_{x, \alpha, \beta, \sigma} D(H_{render}(x, \alpha, \beta, \sigma), H_{ref}(x, \alpha, \beta, \sigma))$$
    where $D$ is a histogram distance metric (e.g., Chi-squared, Earth Mover's Distance, or KL divergence). [S1]

4.  **Compute Gradients:**
    Compute gradients $\frac{\partial E}{\partial \theta}$ using the chain rule. The gradient depends on the standard RGB gradients from the differentiable renderer $\frac{\partial I}{\partial \theta}$ and the derivative of the histogram distance metric with respect to the rendered intensities.
    $$\frac{\partial E}{\partial \theta} = \sum_{x, \alpha, \beta, \sigma} \frac{\partial D}{\partial H_{render}} \cdot \frac{\partial H_{render}}{\partial I} \cdot \frac{\partial I}{\partial \theta}$$
    [S1]

5.  **Update Parameters:**
    Update parameters $\theta$ via gradient descent:
    $$\theta_{new} = \theta_{old} - \eta \frac{\partial E}{\partial \theta}$$
    where $\eta$ is the learning rate. [S1]

**Parameters:**
*   $\sigma$ (sigma): Inner scale parameter controlling effective resolution via Gaussian kernel width. Typical values: $\sigma=0$ (stationary), $\sigma>0$ (coarser scales). [S1]
*   $\alpha$ (alpha): Extent scale parameter defining the spatial extent (neighborhood size) for histogram computation. [S1]
*   $\beta$ (beta): Tonal scale parameter quantifying imprecision in measured radiance (bin width). [S1]

**Pitfalls:**
*   Standard multi-resolution image pyramids (Gaussian Pyramids) are unreliable for complex inverse rendering because blurring averages out local appearance and suppresses high-frequency details, leading to local minima [S1].
*   In cases with no overlap between initial and target features (e.g., disks), standard image gradients $\frac{\partial I}{\partial \theta}$ are sparse (non-zero only at silhouettes), causing the error gradient $\frac{\partial E}{\partial \theta}$ to be zero everywhere, stalling optimization [S1].
*   LOI requires careful selection of $\alpha$ and $\beta$ to balance noise robustness and detail preservation. [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.ndimage import gaussian_filter
from scipy.stats import entropy

def gaussian_kernel(sigma):
    """Create a 2D Gaussian kernel."""
    size = int(np.ceil(3 * sigma)) * 2 + 1
    x, y = np.meshgrid(np.linspace(-size, size, size), np.linspace(-size, size, size))
    kernel = np.exp(-(x**2 + y**2) / (2 * sigma**2))
    return kernel / np.sum(kernel)

def compute_histogram(intensity_image, bin_width, num_bins):
    """Compute histogram of intensity values."""
    # Clip intensities to valid range
    intensities = intensity_image.flatten()
    # Create histogram
    hist, _ = np.histogram(intensities, bins=num_bins, range=(0, num_bins * bin_width))
    return hist.astype(float)

def compute_loi(image, sigma, alpha, beta, num_bins):
    """
    Compute Locally Orderless Image representation.
    image: 2D numpy array of intensities.
    sigma: Inner scale (Gaussian blur width).
    alpha: Extent scale (neighborhood size for histogram).
    beta: Tonal scale (bin width).
    num_bins: Number of histogram bins.
    """
    # Inner Scale: Gaussian Blur
    if sigma > 0:
        blurred_image = gaussian_filter(image, sigma=sigma)
    else:
        blurred_image = image.copy()
    
    # Extent Scale: Compute histograms in neighborhoods
    # For each pixel, compute histogram of intensities in alpha x alpha neighborhood
    h, w = blurred_image.shape
    lois = []
    
    # Precompute kernel for extent scale if needed, but here we just use a box filter concept
    # Note: The source describes computing histograms within a spatial neighborhood.
    # This is computationally expensive. A common approximation is to use a sliding window.
    
    half_alpha = alpha // 2
    
    for i in range(h):
        for j in range(w):
            # Define neighborhood
            i_start = max(0, i - half_alpha)
            i_end = min(h, i + half_alpha + 1)
            j_start = max(0, j - half_alpha)
            j_end = min(w, j + half_alpha + 1)
            
            neighborhood = blurred_image[i_start:i_end, j_start:j_end]
            
            # Tonal Scale: Bin intensities
            # Flatten neighborhood and compute histogram
            hist, _ = np.histogram(neighborhood, bins=num_bins, range=(0, num_bins * beta))
            lois.append(hist)
            
    return np.array(lois)

def histogram_distance(hist1, hist2):
    """Compute Chi-squared distance between two histograms."""
    # Avoid division by zero
    hist1 = np.maximum(hist1, 1e-10)
    hist2 = np.maximum(hist2, 1e-10)
    return np.sum((hist1 - hist2)**2 / (hist1 + hist2))

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
    
    # Precompute Reference LOI
    ref_loi = compute_loi(reference_image, sigma, alpha, beta, num_bins)
    
    for i in range(num_iterations):
        # Render current image
        current_image = render_fn(theta)
        
        # Compute Rendered LOI
        render_loi = compute_loi(current_image, sigma, alpha, beta, num_bins)
        
        # Compute Objective
        error = 0
        for h_ref, h_render in zip(ref_loi, render_loi):
            error += histogram_distance(h_ref, h_render)
        
        # Compute Gradients
        # Note: The source states that gradients are computed using standard RGB gradients
        # from the differentiable renderer and the histogram distance metric.
        # This requires the differentiable renderer to provide dI/dtheta.
        # Here we assume a placeholder for the gradient computation.
        # In practice, this involves backpropagating through the histogram distance.
        
        # Placeholder for gradient computation
        # grad = compute_gradient(render_fn, theta, reference_image, sigma, alpha, beta, num_bins)
        # For the purpose of this pseudocode, we assume a generic gradient function exists.
        grad = compute_gradient(render_fn, theta, reference_image, sigma, alpha, beta, num_bins)
        
        # Update Parameters
        theta = theta - learning_rate * grad
        
    return theta
```

## Subtopics

### [Variational Optimization with Locally Orderless Images (LOI)](11021-variational-optimization-with-locally-orderless-images-loi.md)
Variational Optimization with Locally Orderless Images (LOI) is a differentiable rendering optimization technique that combines variational derivatives with multi-scale local histogram matching to recover scene parameters (e.g., geometry, camera pose, material properties) from images where standard RGB gradients are unstable, sparse, or insufficient due to complex light transport or high variance. Unlike variational optimization using stationary image matching, which fails due to strict alignment sensitivity and noise, LOI matching does not require perfect alignment between the variational estimate and reference images, allowing reliable recovery from arbitrary initializations.

## Operating characteristics
*   **Time Complexity:** Computing LOI involves Gaussian blurring and histogram computation over neighborhoods. The complexity is $O(N \cdot \alpha^2)$ per pixel for histogram computation, where $N$ is the number of pixels and $\alpha$ is the neighborhood size. This is significantly more expensive than standard pixel-wise comparison.
*   **Memory Complexity:** Storing LOI representations for multiple scales and pixels requires $O(N \cdot \alpha^2 \cdot N_{scales})$ memory, where $N_{scales}$ is the number of scale levels.
*   **Measured Performance:** Validated on inverse vectorization, path tracing, and rasterization problems using both synthetic and real data [S1].
*   **Scaling:** LOI scales with the number of scales and neighborhood size. Larger $\alpha$ and $\sigma$ improve gradient support but increase computational cost.
*   **Strengths:** Extends gradient support for sparse gradients; robust to noise; works with arbitrary geometry representations and complex light transport; does not require perfect alignment [S1].
*   **Limitations:** Computationally expensive due to histogram computation; sensitive to parameter selection ($\alpha, \beta, \sigma$); may smooth out fine details if scales are too large [S1].
*   **When to use:** Inverse rendering problems with sparse gradients, large initial errors, or complex light transport where standard RGB gradients fail [S1].
*   **When not to use:** Real-time applications requiring fast convergence; problems where fine detail preservation is critical and computational resources are limited [S1].

## Background
LOI was originally proposed by Griffin [11] and later extended by Koenderink and van Doorn [15] for reasoning about image topology [S1]. It has been used in various vision tasks, including image retrieval [17], object tracking [25], and non-linear filtering [33, 34] [S1]. The application of LOI to differentiable rendering optimization was introduced by Chandraker and Ramamoorthi in 2025 [S1].

## Sources
[S1] Locally Orderless Images for Optimization in Differentiable Rendering, https://doi.org/10.1109/cvpr52734.2025.00541, fetched 2026-10-26
