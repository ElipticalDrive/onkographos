# PlenOctree Fine-tuning

Also called: Accelerated NeRF Training

## Overview
PlenOctree Fine-tuning is a method to improve the image quality of a pre-converted PlenOctree by optimizing its leaf node parameters against original training images using standard NeRF loss and SGD, while keeping the octree structure fixed [S1, unofficial]. It distinguishes itself from [PlenOctree Conversion](1359-plenoctree-conversion.md) by performing a second optimization phase on the tree structure itself rather than just baking the initial NeRF weights, and from [Analytic Derivatives of PlenOctree Rendering](1360-analytic-derivatives-of-plenoctree-rendering.md) by relying on numerical gradient descent rather than closed-form derivatives. This approach accelerates NeRF-based rendering training by 5 orders of magnitude without loss in image quality [S1, unofficial].

## How it works
**Inputs:** A pre-converted PlenOctree (structure and initial leaf parameters derived from a coarse NeRF), original training images with camera poses, and a target NeRF loss function (e.g., MSE).
**Outputs:** An optimized PlenOctree with refined leaf node coefficients.

**Data Structures:**
The PlenOctree is a hierarchical grid. Each leaf node stores spherical harmonics (SH) coefficients for color and a density value. The structure is fixed during this phase; only the values within the leaves are updated.

**Steps:**
1.  **Initialize:** Load the PlenOctree structure and initial leaf parameters from the conversion step [S1, unofficial].
2.  **Sample Rays:** For each training image, cast rays through each pixel.
3.  **Volume Rendering:** For each ray, traverse the octree. At each leaf node intersected, evaluate the SH color and density. Compute the accumulated color and transmittance using standard volume rendering integration [S1, unofficial].
4.  **Compute Loss:** Compare the rendered pixel color with the ground truth training image pixel. Calculate the loss (typically Mean Squared Error).
5.  **Backpropagate:** Compute gradients of the loss with respect to the leaf node parameters (SH coefficients and densities) using the analytic derivatives of the rendering equation or automatic differentiation [S1, S1, unofficial].
6.  **Update:** Update the leaf node parameters using an optimizer (e.g., SGD or Adam) [S1, unofficial].
7.  **Iterate:** Repeat steps 2-6 for a fixed number of epochs or until convergence.

**Formulas:**
The volume rendering equation for a ray $\mathbf{r}$ is:
$$ C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i c_i $$
where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance, $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity, $\sigma_i$ is the density, $\delta_i$ is the segment length, and $c_i$ is the color from SH evaluation [S1, unofficial].

**Parameters:**
*   Learning rate: Not given in the sources.
*   Optimizer: SGD or Adam [S1, unofficial].
*   Epochs: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** SH evaluation can suffer from ringing artifacts if high-order SH are used with insufficient sampling [S1, unofficial].
*   **Boundaries:** The octree structure is fixed, so if the initial conversion missed fine details, fine-tuning may not recover them if the leaf resolution is too coarse [S1, unofficial].
*   **Degenerate Input:** If the initial NeRF was poorly trained, the PlenOctree may have incorrect topology or missing regions, which fine-tuning cannot fully correct [S1, unofficial].

## Subtopics

### EfficientNeRF
EfficientNeRF is an improvement on PlenOctree that uses spherical harmonics and caches the trained scene in a tree, with specific optimizations for training speed [S2]. It uses a momentum density voxel grid to store predicted density via exponential weighted average updates, discarding zero-density points during coarse sampling [S2]. During fine sampling, it employs a pivot system where pivot points are defined as points $x_i$ for which $T_i \alpha_i > \epsilon$, where $\epsilon$ is a predefined threshold, and $T_i$ and $\alpha_i$ are transmittance and alpha values [S2]. Only points near pivot points are considered during fine sampling [S2]. This speeds up training time by a factor of 8 over baseline NeRF and results in rendering speeds comparable to FastNeRF and exceeding baseline NeRF by thousands-fold [S2].

## Operating characteristics
*   **Time Complexity:** Training is significantly faster than full NeRF optimization because it avoids neural network forward/backward passes during the fine-tuning phase, relying instead on direct SH evaluation and volume rendering [S1, unofficial].
*   **Memory Complexity:** Depends on the PlenOctree resolution and SH order. The momentum density voxel grid in EfficientNeRF adds overhead but reduces sampling cost [S2].
*   **Measured Performance:** Accelerates NeRF-based rendering by 5 orders of magnitude without loss in image quality [S1, unofficial]. EfficientNeRF variant speeds up training by a factor of 8 over baseline NeRF and achieves rendering speeds thousands-fold faster than baseline NeRF [S2].
*   **Strengths:** High rendering speed, good image quality, practical for real-time applications [S1, unofficial].
*   **Limitations:** Requires a pre-converted PlenOctree; quality depends on the initial conversion [S1, unofficial].

## Background
PlenOctree Fine-tuning was introduced in the paper "PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs)" [S1, unofficial]. It builds on the PlenOctree representation, which converts a trained NeRF into a hierarchical grid of spherical harmonics. The method addresses the need for faster training and rendering of NeRFs by decoupling the heavy neural network training from the final optimization phase. EfficientNeRF [S2] further refined this approach with momentum density grids and pivot systems.

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
