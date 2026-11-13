# Gaussian Splatting Photometric Stereo

Also called: GS-PS

## Overview
Gaussian Splatting Photometric Stereo (GS-PS) is a calibrated photometric stereo method that estimates surface normals and albedo by optimizing 3D Gaussian Splatting parameters to minimize a combined photometric and normal alignment loss [S1, unofficial]. It differs from deep learning-based photometric stereo by optimizing an explicit physical image formation model rather than neural network weights, and from inverse rendering methods using small networks by using Gaussian parameters with a direct geometric interpretation [S1, unofficial].

## How it works
**Inputs:** A set of $n$ calibrated images of an object taken under different known light directions. The method assumes Lambertian materials for the baseline evaluation [S1, unofficial].
**Outputs:** Optimized 3D Gaussian parameters (position, covariance, color, opacity) representing the scene, from which surface normals and albedo maps are derived.

**Data Structures:**
The scene is represented as a collection of 3D Gaussians. Each Gaussian $i$ is defined by:
*   Position $\mu_i \in \mathbb{R}^3$
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (or its rotation-scaling decomposition)
*   Color $c_i$ (represented using spherical harmonics to allow view-dependent color variation) [S1, unofficial]
*   Opacity $\alpha_i \in [0, 1]$

**Rendering Model:**
Standard Gaussian Splatting does not account for lighting, meaning a render from a fixed viewpoint is identical regardless of light position [S1, unofficial]. To handle photometric stereo, the method uses a lighting model (referred to as Equation 1 in the source) to analyze surface alignment with light directions to calculate self-occlusion shadows [S1, unofficial].

The composite image $c(x, y)$ for a given view is calculated as the maximum of $n$ input images:
$$ c(x, y) = \max\{I_1 (x, y), I_2 (x, y), \dots , I_n (x, y)\} $$
This composite serves as the target or reference for photometric consistency [S1, unofficial].

**Loss Function:**
The total loss $L$ is defined as:
$$ L = L_c + \lambda L_n $$
where:
*   $L_c$ is the photometric loss, calculated using an L1 loss function to measure the difference between the rendered image and the real input image [S1, unofficial].
*   $L_n$ is the normal regularization loss. A normal map is computed from the depth map gradient and compared with the normal map obtained via splatting to ensure Gaussians are aligned with the true object surface [S1, unofficial].
*   $\lambda$ is a weighting parameter for the normal term [S1, unofficial].

**Optimization Steps:**
1.  **Initialization:** Initialize a set of 3D Gaussians to represent the scene.
2.  **Rendering:** Render the normal map and albedo map. Use the lighting model to determine if surfaces are aligned with light directions to calculate self-occlusion shadows [S1, unofficial].
3.  **Loss Calculation:**
    *   Compute $L_c$ between the rendered image and the input image using L1 loss [S1, unofficial].
    *   Compute $L_n$ by comparing the normal map derived from the depth map gradient with the normal map from splatting [S1, unofficial].
    *   Compute total loss $L = L_c + \lambda L_n$ [S1, unofficial].
4.  **Gradient Descent:** Optimize Gaussian parameters (position, covariance matrix, color, opacity) using gradient descent. The rendering engine's exclusive use of Gaussians allows for efficient gradient calculation [S1, unofficial].
5.  **Adaptive Evolution:** Evolve the number of Gaussians during optimization:
    *   Remove Gaussians that are too large or have too low opacity.
    *   Clone or split Gaussians when more detail is needed.
    *   (Details of this process are described in source [5], which is not provided in the citation list, so specific thresholds are not given) [S1, unofficial].

**Pitfalls:**
*   **Cast Shadows:** The current model does not account for cast shadows (ombres portées), causing high angular estimation errors. Errors are concentrated in regions where cast shadows are present on many input images [S1, unofficial].
*   **Lambertian Assumption:** The method's performance on the DiliGenT dataset was evaluated on objects with Lambertian materials; deviations from Lambertian reflectance may affect accuracy [S1, unofficial].

## Operating characteristics
**Performance:**
On the DiliGenT dataset (Bear, Cat, Pot1 objects with Lambertian materials), measured by mean angular error in degrees [S1, unofficial]:
*   **Cat:** GS-PS: 8.13° (Baseline: 8.41°, DANINet: 4.73°, Uni MS-PS: 3.45°)
*   **Bear:** GS-PS: 10.11° (Baseline: 8.39°, DANINet: 4.11°, Uni MS-PS: 3.14°)
*   **Pot1:** GS-PS: 4.12° (Baseline: 8.89°, DANINet: 6.41°, Uni MS-PS: 4.12°) [S1, unofficial]

**Strengths:**
*   More explainable than deep neural network methods because it optimizes an explicit physical image formation model [S1, unofficial].
*   More explainable than inverse rendering methods using small networks because the optimized Gaussian parameters have a geometric interpretation [S1, unofficial].
*   Efficient gradient calculation due to the exclusive use of Gaussians in the rendering engine [S1, unofficial].

**Limitations:**
*   High angular estimation errors in regions with cast shadows [S1, unofficial].
*   Performance on complex objects (Cat, Bear) is lower than specialized deep learning methods (DANINet, Uni MS-PS) [S1, unofficial].

**When to use:**
*   When interpretability of the reconstruction is prioritized over state-of-the-art accuracy on complex lighting conditions.
*   For calibrated photometric stereo with Lambertian surfaces.

**When not to use:**
*   When cast shadows are prominent and cannot be mitigated.
*   When higher accuracy is required than what GS-PS provides on non-trivial objects (e.g., Cat, Bear).

## Background
GS-PS is a method for calibrated photometric stereo using 3D Gaussian Splatting [S1, unofficial]. It builds on the Adaptive Spatio-Temporal 3D Gaussian Splatting method [9535-adaptive-spatio-temporal-3d-gaussian-splatting.md] by incorporating photometric and normal regularization losses. The method was presented in a source dated 2026 (unofficial) [S1, unofficial].

## Sources
[S1] Stéréophotométrie par Gaussian Splatting et rendu inverse, https://hal.science/hal-05151330v1, fetched 2026-10-26 (unofficial)
