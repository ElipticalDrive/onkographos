# EndoNeRF

Also called: Endoscopic Neural Radiance Field

## Overview
EndoNeRF is a method for reconstructing deformable surgical scenes using a canonical radiance field and a time-dependent displacement field, optimized with stereo depth-cueing and supervision [S1]. It addresses the challenge of non-rigid tissue reconstruction in endoscopic video by mapping input space-time coordinates to a static canonical space via a learned deformation, allowing standard NeRF rendering to operate on the displaced points [S1]. It sets itself apart from siblings like [Neural Volumes](912-neural-volumes.md) by explicitly modeling temporal deformation through a displacement field rather than a static 3D volume or explicit voxel grid, and differs from [Fluid Motion Reconstruction Framework](10672-fluid-motion-reconstruction-framework.md) by focusing on endoscopic stereo cues rather than fluid dynamics priors.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of endoscopic images (stereo pairs) with known camera poses and timestamps $\tau$.
*   **Outputs:** A rendered image $\hat{C}(\mathbf{r})$ for a given ray $\mathbf{r}$ and time $\tau$, and the underlying scene representation (canonical field and deformation field).

**Data Structures:**
1.  **Canonical Radiance Field ($F_\Theta$):** An MLP that maps a 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$.
    *   Architecture: 8 fully connected ReLU layers, each with 256 channels [S1].
    *   Input encoding: Positional encoding (Fourier features) for input coordinates $\mathbf{x}$ and time $\tau$ [S1].
2.  **Time-Dependent Displacement Field ($G_\Phi$):** An MLP that maps a 3D position $\mathbf{x}$ and time $\tau$ to a 3D displacement vector $\Delta \mathbf{x}$.
    *   Input encoding: Positional encoding for $\mathbf{x}$ and $\tau$ [S1].

**Method Steps:**
1.  **Ray Sampling:** For a camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ at time $\tau$, sample $N$ points along the ray at depths $t_i$ [S1].
2.  **Deformation Mapping:** For each sample point $\mathbf{x}_i = \mathbf{r}(t_i)$, query the displacement field to find the corresponding location in the canonical space:
    $$ \mathbf{x}'_i = \mathbf{x}_i + G_\Phi(\mathbf{x}_i, \tau) $$
    where $G_\Phi(\mathbf{x}_i, \tau)$ is the displacement between point $\mathbf{x}_i$ at time $\tau$ and its corresponding point in the canonical field [S1].
3.  **Field Query:** Query the canonical radiance field $F_\Theta$ at the displaced location $\mathbf{x}'_i$ with viewing direction $\mathbf{d}$ to obtain density $\sigma_i$ and color $\mathbf{c}_i$ [S1]:
    $$ (\sigma_i, \mathbf{c}_i) = F_\Theta(\mathbf{x}'_i, \mathbf{d}) $$
4.  **Volume Rendering:** Compute the final color $\hat{C}(\mathbf{r})$ using the standard differentiable volume rendering equation [44-radiance-field-optimization.md]:
    $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    where $\delta_i$ is the step size and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S1, 44-radiance-field-optimization.md].
5.  **Optimization:** Minimize the L2 photometric reconstruction loss between rendered images and ground truth stereo images [S1]. The optimization is supervised by stereo depth-cueing, which provides depth constraints to regularize the deformation field and improve reconstruction quality [S1].

**Parameters:**
*   MLP Layers: 8 fully connected layers [S1].
*   MLP Channels: 256 per layer [S1].
*   Activation: ReLU [S1].
*   Loss: L2 photometric reconstruction loss [S1].

**Edge Cases and Pitfalls:**
*   **Resource Consumption:** EndoNeRF optimizes an entire spatial-temporal field, resulting in significant time and resource consumption [S1].
*   **Topology Changes:** The displacement field approach assumes topology preservation; it may struggle with significant changes in scene topology (e.g., tissue tearing) [S1].

## Subtopics

### Canonical Volume + Deformation Field Framework
A framework for spatio-temporal reconstruction of objects from few X-ray projections, consisting of a canonical volume assigning density to every point in 3D space and a deformation field returning a displacement for every point as a function of time [S2, unofficial]. It allows querying the neural radiance field at continuous temporal and spatial locations [S2, unofficial]. The deformation field is posed as a B-spline field whose time-dependent weights are obtained from a neural network, using cubic B-splines with fixed control points and time-varying weights to impose a smooth efficient prior [S2, unofficial]. The canonical volume is trained using a hybrid method where projection loss is supplemented by volumetric loss based on high-fidelity X-CT data typically available at initial and final timesteps [S2, unofficial]. The deformation field is trained jointly for all timesteps and with all available data [S2, unofficial]. It is preferred to train the canonical volume on the final step of deformation where the maximum amount of material is present in the field of view to map material seen in projections to the canonical volume [S2, unofficial]. This framework assumes deformations preserve topology [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** High. Optimizing an entire spatial-temporal field results in significant time consumption [S1].
*   **Memory Complexity:** High. The 8-layer, 256-channel MLP for both the canonical field and displacement field requires substantial memory [S1].
*   **Strengths:** Yields high-quality non-rigid reconstruction in surgical scenes [S1]. Can model complex tissue deformations [S1].
*   **Limitations:** Significant time and resource consumption [S1]. Struggles with changes in scene topology [S1]. Inference speed may be insufficient for clinical demands compared to factorized methods like Lerplane [S1].
*   **When to Use:** When high-quality reconstruction of deformable tissue is required and computational resources are available.
*   **When Not to Use:** When real-time inference is required or when scene topology changes significantly.

## Background
EndoNeRF emerged as a promising solution for deformable surgical scene reconstruction, utilizing NeRF with tool-guided ray casting, stereo depth-cueing ray marching, and stereo depth-supervised optimization [S1]. It was introduced to address the limitations of previous methods in handling non-rigid tissue deformation in endoscopic video [S1]. The method builds on the principles of differentiable volume rendering and radiance field optimization [44-radiance-field-optimization.md].

## Sources
[S1] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
[S2] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
