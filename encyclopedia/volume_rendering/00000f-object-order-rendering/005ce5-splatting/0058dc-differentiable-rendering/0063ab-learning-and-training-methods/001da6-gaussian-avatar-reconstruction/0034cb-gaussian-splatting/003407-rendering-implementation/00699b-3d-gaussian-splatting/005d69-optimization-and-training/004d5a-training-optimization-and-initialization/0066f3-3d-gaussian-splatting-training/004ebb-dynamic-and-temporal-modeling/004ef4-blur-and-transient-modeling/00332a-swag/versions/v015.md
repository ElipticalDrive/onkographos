# SWAG

Also called: Unsupervised Transient Gaussian Splatting, Unsupervised Transient Gaussians

## Overview
SWAG is a method for unsupervised transient modeling in 3D Gaussian Splatting that extends the standard representation by conditioning Gaussians on learned appearance embeddings [S1][S2, unofficial]. It solves the problem of transient occluders (e.g., pedestrians, moving vehicles) in casual photo captures by training dedicated transient Gaussians that are explicitly optimized to ignore occlusion, yielding state-of-the-art results on diverse photo scenes [S1][S2, unofficial]. It differs from sibling methods like Wild-GS, which aligns pixel appearance features from image triplanes to 3D Gaussians using depth regularization and visibility maps, by focusing on unsupervised transient Gaussian training rather than triplane alignment [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of unposed or loosely posed images of a scene containing transient objects.
*   **Outputs:** A set of 3D Gaussians representing the static scene geometry and appearance, plus a set of transient Gaussians representing occluders.

**Data Structures:**
The method maintains two sets of Gaussians:
1.  **Static Gaussians:** Standard 3DGS parameters (position, covariance, opacity, spherical harmonics coefficients) conditioned on learned appearance embeddings.
2.  **Transient Gaussians:** A separate set of Gaussians optimized to model transient distractors.

**Method Steps:**
1.  **Initialization:** Initialize a set of 3D Gaussians from the input images.
2.  **Appearance Conditioning:** Learn appearance embeddings for the Gaussians to handle view-dependent effects and static appearance variations [S1][S2, unofficial].
3.  **Transient Gaussian Training:** Introduce transient Gaussians into the scene representation. These are trained unsupervised to model pixels that are occluded or inconsistent with the static scene model [S1][S2, unofficial].
4.  **Occlusion Ignorance:** The transient Gaussians are optimized to "ignore" occluders, meaning they absorb the error caused by transient objects, allowing the static Gaussians to reconstruct the underlying static scene geometry without being corrupted by moving elements [S1][S2, unofficial].
5.  **Rendering:** Render the scene by compositing contributions from both static and transient Gaussians. The static Gaussians provide the consistent background, while transient Gaussians capture the dynamic elements.

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   Not given in the sources.

## Subtopics

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Yields state-of-the-art results on diverse photo scenes [S1][S2, unofficial].
*   **Strengths:** Robustness to transient occluders in casual captures; unsupervised training (no need for explicit transient labels).
*   **Limitations:** Not given in the sources.
*   **When to use:** When reconstructing scenes from casual photo captures with transient objects (e.g., pedestrians, vehicles) and unsupervised learning is preferred.
*   **When not to use:** When explicit transient modeling is not required or when posed data with known camera parameters is available and transient objects are not a concern.

## Background
SWAG was introduced as an extension to 3D Gaussian Splatting to handle transient objects in unstructured photo collections. It builds on the parent's mechanisms for temporal parameterization, canonical space mapping, and dynamic rendering [Parent](13851-blur-and-transient-modeling.md). It is related to Wild-GS, which uses triplane features and depth regularization, and SpotlessSplats, which uses pre-trained feature spaces for outlier detection [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
