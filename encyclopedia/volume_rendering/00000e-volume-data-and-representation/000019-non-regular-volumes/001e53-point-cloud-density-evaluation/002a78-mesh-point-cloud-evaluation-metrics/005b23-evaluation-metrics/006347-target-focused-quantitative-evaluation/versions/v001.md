# Target-Focused Quantitative Evaluation

Also called: Region-of-Interest PSNR/SSIM/LPIPS

## Overview
Target-Focused Quantitative Evaluation isolates specific semantic objects or foreground targets within a rendered image to assess reconstruction quality independently of the background. It solves the problem where background regions, which may be less critical to the user or contain large uniform areas, dilute the perceived quality of the primary subject. It differs from standard [Evaluation Metrics](9899-evaluation-metrics.md) by restricting the PSNR, SSIM, and LPIPS calculation to a user-defined bounding box enclosing the target, rather than the entire image frame.

## How it works
The method evaluates rendering fidelity by applying standard image quality metrics to a specific spatial subset of the image.

**Inputs and Outputs:**
*   **Inputs:** Reference image $I_{ref}$, rendered image $I_{render}$, and a rectangular region definition $(x, y, w, h)$ specifying the target's location and size.
*   **Outputs:** Scalar values for PSNR, SSIM, and LPIPS computed only over the pixels within the defined region.

**Data Structures:**
*   **Region Definition:** A tuple or struct containing:
    *   $x, y$: Top-left pixel coordinates (integer).
    *   $w, h$: Width and height in pixels (integer).
*   **Image Data:** 2D arrays of pixel values (RGB or grayscale) for $I_{ref}$ and $I_{render}$.

**Steps:**
1.  **Select Views:** Choose representative test views where the target is visible and significant. [S1]
2.  **Define Region:** Identify the rectangular bounding box that encloses the primary target (e.g., a sculpture) in the reference image. [S1]
3.  **Apply Region:** Extract the sub-images $I_{ref}^{ROI}$ and $I_{render}^{ROI}$ using the coordinates $(x, y, w, h)$ from both the reference and rendered images. [S1]
4.  **Calculate Metrics:** Compute PSNR, SSIM, and LPIPS on $I_{ref}^{ROI}$ and $I_{render}^{ROI}$ using the standard full-image evaluation procedures. [S1]

**Formulas:**
The metrics are calculated using the same definitions as the parent [Evaluation Metrics](9899-evaluation-metrics.md), but restricted to the ROI pixels.
*   **PSNR:**
    $$ \text{PSNR} = 10 \cdot \log_{10} \left( \frac{MAX_I^2}{\text{MSE}} \right) $$
    Where $MAX_I$ is the maximum possible pixel value (e.g., 1.0 or 255), and MSE is the Mean Squared Error between the ROI pixels of the reference and rendered images.
*   **SSIM:**
    $$ \text{SSIM}(x, y) = \frac{(2\mu_x\mu_y + C_1)(2\sigma_{xy} + C_2)}{(\mu_x^2 + \mu_y^2 + C_1)(\sigma_x^2 + \sigma_y^2 + C_2)} $$
    The final SSIM score is the mean of the local SSIM maps over the ROI. The source uses an 11×11 Gaussian window with $\sigma=1.5$, data range 1.0, and population covariance normalization. [S1]
*   **LPIPS:**
    Computed using learned perceptual image patch similarity, restricted to the ROI patches. [S1]

**Parameters:**
*   **Window Size:** 11×11 pixels for SSIM. [S1]
*   **Standard Deviation:** 1.5 for the SSIM Gaussian window. [S1]
*   **Data Range:** 1.0. [S1]

**Edge Cases and Pitfalls:**
*   **Boundary Alignment:** The bounding box must be applied identically to both reference and rendered images to ensure pixel-wise correspondence. [S1]
*   **Target Occlusion:** If the target is partially occluded or outside the frame in the rendered view, the metric may become undefined or misleading. [G]
*   **Background Influence:** This method explicitly excludes background noise, which may hide global rendering artifacts but accurately reflects local target fidelity. [S1]

## Subtopics

### ROI-Level Quantitative Evaluation
A generalization of the target-focused approach where multiple representative regions of interest (ROIs) are defined across various scene types (e.g., forest, bridge, pavement) to assess local features like vegetation crowns or surface textures. It uses the same bounding box application and metric calculation as the target-focused method but applies it to multiple, semantically diverse ROIs rather than a single foreground object. [S1]

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels in the ROI. Significantly faster than full-image evaluation if the ROI is small.
*   **Memory Complexity:** Linear with respect to the ROI size.
*   **Strengths:** Isolates quality of specific objects; prevents background uniformity from skewing metrics; useful for assessing fine details like textures or structural features. [S1]
*   **Limitations:** Not an unbiased statistical sample of the whole image; results depend heavily on the subjective selection of the ROI. [S1]
*   **When to Use:** When the primary interest is the reconstruction quality of specific foreground objects or detailed textures, rather than overall scene coherence. [S1]

## Background
This method was introduced in the context of evaluating 3D Gaussian Splatting for real-world scene reconstruction. It was developed to complement whole-image metrics by examining the visibility and reconstruction quality of local scene features, such as object boundaries and fine surface features, which might be averaged out in global assessments. [S1]

## Sources
[S1] Practical Evaluation of Standard 3D Gaussian Splatting for Real-World Scene Reconstruction., https://doi.org/10.3390/s26185754, fetched 2026-10-26
