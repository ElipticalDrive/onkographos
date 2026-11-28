# NeRF with Time Parameter

Also called: NeRF + time, NeRF with Time Parameter Baseline, Time-NeRF

## Overview
This method adapts the static Neural Radiance Field (NeRF) architecture to model dynamic scenes by explicitly conditioning the network on a temporal coordinate $t$ [S1]. It solves the problem of reconstructing time-varying geometry and appearance, which vanilla NeRF cannot handle due to its static assumption [S1]. It differs from deformation-based siblings like [Gear-NeRF](13848-gear-nerf.md) by avoiding explicit spatial warping; instead, it learns the temporal variation directly in the latent space, making it a simpler baseline that often suffers from blurriness and ghosting compared to methods that reason about scene dynamics [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D spatial coordinate $\mathbf{x} \in \mathbb{R}^3$, a viewing direction $\mathbf{d} \in \mathbb{R}^3$, and a scalar time parameter $t$.
*   **Output:** Volume density $\sigma$ and view-dependent color $\mathbf{c}$.

**Data Structures:**
The model consists of two Multi-Layer Perceptrons (MLPs), identical in structure to the standard NeRF [S1]:
1.  **Density MLP:** Takes encoded spatial coordinates and outputs density $\sigma$.
2.  **Color MLP:** Takes encoded spatial coordinates, viewing direction, and outputs color $\mathbf{c}$.

**Steps:**
1.  **Positional Encoding of Time:** Apply the same positional encoding function $\gamma_t$ used for spatial coordinates to the time parameter $t$.
    $$ \gamma_t(t) = \text{PE}(t) $$
2.  **Positional Encoding of Space:** Apply positional encoding $\gamma_x$ to the spatial coordinate $\mathbf{x}$.
    $$ \gamma_x(\mathbf{x}) = \text{PE}(\mathbf{x}) $$
3.  **Concatenation:** Concatenate the encoded time and encoded spatial coordinates to form the input vector for the density MLP.
    $$ \mathbf{v}_{\sigma} = [\gamma_x(\mathbf{x}), \gamma_t(t)] $$
4.  **Density Prediction:** Pass $\mathbf{v}_{\sigma}$ through the density MLP to obtain $\sigma$.
    $$ \sigma = \text{MLP}_{\sigma}(\mathbf{v}_{\sigma}) $$
5.  **Feature Extraction:** Pass $\gamma_x(\mathbf{x})$ through the intermediate layers of the density MLP (or a shared feature network) to obtain a feature vector $\mathbf{f}$. [G: In standard NeRF, this is the output of the hidden layers before the density head].
6.  **Color Input Preparation:** Concatenate the feature vector $\mathbf{f}$, the viewing direction $\mathbf{d}$, and the encoded time $\gamma_t(t)$.
    $$ \mathbf{v}_{c} = [\mathbf{f}, \mathbf{d}, \gamma_t(t)] $$
7.  **Color Prediction:** Pass $\mathbf{v}_{c}$ through the color MLP to obtain $\mathbf{c}$.
    $$ \mathbf{c} = \text{MLP}_{c}(\mathbf{v}_{c}) $$
8.  **Volume Rendering:** Render the image using the standard volume rendering integral [S1], integrating over the ray $\mathbf{r}(s) = \mathbf{o} + s\mathbf{d}$ for the specific time $t$ of the input image.

**Parameters:**
*   **Training Configuration:** Use the configuration reported for NeRF's real forward-facing experiments [S1]. Specific hyperparameters (learning rate, number of samples per ray, etc.) are not detailed in the sources beyond this reference [S1].

**Edge Cases and Pitfalls:**
*   **Blurriness:** The method lacks detail and suffers from blurry predictions because it does not explicitly model the motion or deformation of objects [S1].
*   **Ghosting:** Even small changes in camera pose lead to slightly different viewing directions for spatial points across frames, causing severe ghosting artifacts in static regions [S1].
*   **Dynamic Reasoning:** The model fails to reason about scene dynamics, meaning it cannot accurately reconstruct novel scene compositions or complex temporal interactions [S1].

## Subtopics

### Temporal Compression of NeRF Models
A compression strategy for sequences of NeRF models that exploits temporal redundancy by treating the first model as a key-frame and compressing only the deltas between successive models [S2]. It is chosen when storage or transmission bandwidth for a sequence of dynamic NeRFs is constrained [S2].
*   **Initialization:** Train the NeRF for the first frame from scratch [S2].
*   **Subsequent Frames:** Initialize the NeRF for frame $i$ using the weights of the trained NeRF from frame $i-1$ to encourage temporal redundancy and save training time [S2].
*   **Delta Calculation:** Calculate the delta $\Delta W$ between the current frame's NeRF weights $W_i$ and the previous frame's weights $W_{i-1}$: $\Delta W = W_i - W_{i-1}$ [S2].
*   **Compression:** Apply exponent-based quantization and Huffman coding to the delta weights $\Delta W$ [S2].
*   **Restoration:** To restore a model, accumulate the decoded deltas to the reference model (the previous frame's model) [S2].

## Operating characteristics
*   **Time Complexity:** Training time is comparable to standard NeRF per frame, but slightly reduced for subsequent frames if initialization from the previous frame is used (in the context of compression) [S2]. Rendering time is identical to standard NeRF [S1].
*   **Memory Complexity:** Memory usage is identical to standard NeRF for a single time step [S1]. For a sequence, the compression variant reduces storage by storing only deltas [S2].
*   **Performance:** On KITTI dynamic scenes, "NeRF + time" achieves a PSNR of 24.18, SSIM of 0.677, and LPIPS of 0.425, which is better than vanilla NeRF (PSNR 23.34, SSIM 0.662, LPIPS 0.415) but significantly worse than dynamic scene graph methods (PSNR 26.66, SSIM 0.806, LPIPS 0.186) [S1].
*   **Strengths:** Simple to implement; requires no modification to the core NeRF architecture other than input concatenation [S1].
*   **Limitations:** Produces blurry results; fails to handle complex dynamics; suffers from ghosting artifacts [S1].
*   **When to use:** As a baseline for dynamic scene reconstruction; when computational resources for complex deformation fields are unavailable [S1].
*   **When not to use:** When high-fidelity detail is required; when accurate novel view synthesis of dynamic objects is needed [S1].

## Background
The method was introduced as a baseline modification to demonstrate the limitations of simple time-conditioning in implicit neural representations for dynamic scenes [S1]. It was proposed in the context of comparing against "Neural Scene Graphs for Dynamic Scenes" [S1]. The approach builds on the original NeRF architecture [S1].

## Sources
[S1] Neural Scene Graphs for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.00288, fetched 2026-10-26
[S2] Volumetric Video Compression Through Neural-based Representation, https://doi.org/10.1145/3652212.3652220, fetched 2026-11-05
