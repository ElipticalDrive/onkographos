# Coarse-to-fine multi-resolution densification

Also called: energy density resolution scheduler, energy-aware multi-resolution scheduling

## Overview
Coarse-to-fine multi-resolution densification is a training strategy for 3D Gaussian Splatting that dynamically allocates optimization iterations across different image resolutions based on the image's frequency energy density [S1, unofficial]. It solves the problem of premature convergence to local optima and inefficient spatial diffusion caused by applying full-resolution supervision during the early global spread phase [S1, unofficial]. Unlike [Budget-Centered Training](9682-budget-centered-training.md), which manages Gaussian count via budget constraints, this method manages the *resolution* of supervision to decouple global coverage (split operations) from local refinement (clone operations) [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of training images $I$, total densification iterations $T_{\text{densify}}$, and a maximum scale factor $K$.
*   **Output:** A schedule of iteration counts $T_r$ for each resolution scale $r \in \{1, \dots, K\}$, and the resulting optimized Gaussian primitives.

**Data Structures**
*   **Scale Factor Set:** $A = \{1, 2, \dots, K\}$, where $r=1$ is full resolution and $r=K$ is the coarsest resolution [S1, unofficial].
*   **Iteration Schedule:** An array or map mapping each $r \in A$ to an integer $T_r$, representing the number of iterations spent at that resolution [S1, unofficial].

**Steps**
1.  **Compute Energy Density:** For each scale factor $r \in A$, compute the energy density $E_r$ of the training images.
    *   Downscale the input image $I$ to resolution $I_r$ using bilinear interpolation with anti-aliasing:
        $$I_r = D_r(I) = \text{Bilinear}(I, \text{scale} = 1/r)$$
    *   Compute the 2D Fast Fourier Transform (FFT) of $I_r$: $F(I_r)$ [S1, unofficial].
    *   Calculate the energy spectrum $E(I_r)$:
        $$E(I_r) = \sqrt{\Re(F(I_r))^2 + \Im(F(I_r))^2}$$
        where $\Re$ and $\Im$ are the real and imaginary parts [S1, unofficial].
    *   Compute the normalized energy density $E_r$:
        $$E_r = \|E(I_r)\|_1 \cdot r^2$$
        The term $r^2$ normalizes the energy values across different resolutions [S1, unofficial].
2.  **Allocate Iterations:** Calculate the number of iterations $T_r$ for each resolution $r$ based on the ratio of its energy to the full-resolution energy $E_1$:
    $$T_r = \text{Round}\left(\frac{E_r}{E_1} \cdot T_{\text{densify}}\right)$$
    where $T_{\text{densify}}$ is the total number of densification iterations [S1, unofficial].
3.  **Execute Training Schedule:** Train the Gaussian primitives in reversed order of resolution, from coarsest ($r=K$) to finest ($r=1$) [S1, unofficial].
    *   For a specific scale factor $r$, training starts at iteration $T_{r+1}$ and ends at iteration $T_r$. Note: $T_{K+1}$ is typically 0 or the start of training.
    *   **Global Spread Phase ($r > 1$):** Use downsampled images for supervision. Keep the positional learning rate constant. Suppress clone operations to prevent premature local clustering and encourage spatial diffusion via split operations [S1, unofficial].
    *   **Local Refinement Phase ($r = 1$):** Switch to full-resolution supervision. Reduce the positional learning rate. Reintroduce clone operations to refine high-frequency details [S1, unofficial].

**Parameters**
*   **$T_{\text{densify}}$:** Total densification iterations. Default value: 25,000 [S1, unofficial].
*   **$K$:** Maximum scale factor. Default value: 8 [S1, unofficial].
*   **Positional Learning Rate:** Constant during coarse resolutions ($r > 1$), reduced after restoring full resolution ($r=1$) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Gradient Cancellation:** Using full-resolution images during the global spread phase can cause large Gaussians to project to many pixels. The accumulated gradient vectors may cancel each other out due to opposing directions, resulting in small gradient errors below the split threshold and preventing further split operations [S1, unofficial].
*   **Premature Convergence:** Full-resolution supervision encourages Gaussians to converge prematurely to local optima by overfitting to individual pixels, limiting spatial expansion [S1, unofficial].

## Subtopics

### Energy-guided coarse-to-fine multi-resolution training
A training framework that gradually increases resolution based on energy density in 2D images to cooperate with the global-to-local densification strategy [S1, unofficial]. It starts training at lower resolution to facilitate global spread via split operations, gradually increases resolution based on energy density, and transitions to full-resolution training to allow clone operations for local refinement [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method accelerates training by approximately 2x compared to baseline implementations [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Achieves superior reconstruction performance with fewer Gaussian primitives on datasets such as MipNeRF-360, Deep Blending, and Tanks & Temples [S1, unofficial].
*   **Strengths:** Promotes efficient spatial diffusion of Gaussian primitives, prevents premature local clustering, and balances global scene coverage with detail reconstruction [S1, unofficial].
*   **Limitations:** Requires computation of FFT and energy density for scheduling. Relies on the assumption that energy density correlates with the required training effort at each resolution [S1, unofficial].
*   **When to use:** When training time is a constraint and efficient global coverage is needed before local refinement.
*   **When not to use:** If the scene has very low frequency content where energy-based scheduling provides no benefit, or if computational overhead of FFT is prohibitive for the target hardware.

## Background
The method was introduced in the paper "Decomposing Densification in Gaussian Splatting for Faster 3D Scene Reconstruction" [S1, unofficial]. It builds on the observation that split operations are responsible for global diffusion while clone operations govern local refinement [S1, unofficial]. The approach draws inspiration from frequency analysis techniques in 2D image processing and DashGaussian [S1, unofficial]. It was developed to address the inefficiencies of standard 3D Gaussian Splatting densification, which often leads to unnecessary computational consumption and suboptimal behavior when using full-resolution supervision too early [S1, unofficial].

## Sources
[S1] Decomposing Densification in Gaussian Splatting for Faster 3D Scene Reconstruction, https://doi.org/10.48550/arXiv.2507.20239, fetched 2026-10-30 (unofficial)
