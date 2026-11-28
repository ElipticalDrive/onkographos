# NeRF with Time Parameter

Also called: NeRF + time, NeRF with Time Parameter Baseline, Time-NeRF

## Overview
This method adapts the static Neural Radiance Field (NeRF) architecture [Parent](3065-nerf-hyperparameters.md) to represent dynamic scenes by introducing a temporal dimension. It solves the problem of "ghosting" and blurry predictions in static NeRFs when applied to video sequences by allowing the scene representation to vary over time. It differs from sibling methods like [ST-NeRF](18515-st-nerf.md) or [KFD-NeRF](15551-kfd-nerf.md) by using a minimal modification: simply concatenating a positional encoding of time to the spatial input, rather than employing complex warping fields, latent codes, or separate temporal modules.

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, sampled at $N$ points $\mathbf{x}_i$ along the ray, and a scalar time parameter $t$.
*   **Output:** Volume density $\sigma$ and view-dependent color $\mathbf{c}$ for each sample point.

**Data Structures:**
The model is a Multi-Layer Perceptron (MLP) $F_\Theta$. The input vector to the first layer is constructed as follows:
1.  Positional encoding of spatial coordinates: $\gamma_{\mathbf{x}}(\mathbf{x}) \in \mathbb{R}^{2L_{\mathbf{x}}+1}$, where $L_{\mathbf{x}}$ is the number of frequency bands.
2.  Positional encoding of time: $\gamma_{t}(t) \in \mathbb{R}^{2L_{t}+1}$, where $L_{t}$ is the number of frequency bands for time.
3.  Concatenated input: $\mathbf{v} = [\gamma_{\mathbf{x}}(\mathbf{x}); \gamma_{t}(t)]$.
4.  The viewing direction $\mathbf{d}$ is typically encoded separately as $\gamma_{\mathbf{d}}(\mathbf{d})$ and concatenated later in the network (e.g., after the first hidden layer), as per the standard NeRF architecture [Parent](3065-nerf-hyperparameters.md).

**Steps:**
1.  **Encode Time:** Apply the same positional encoding function $\gamma$ used for spatial coordinates to the scalar time $t$.
    $$ \gamma_{t}(t) = [\sin(2^0 \pi t), \cos(2^0 \pi t), \dots, \sin(2^{L_t-1} \pi t), \cos(2^{L_t-1} \pi t)] $$
2.  **Concatenate:** Append $\gamma_{t}(t)$ to the spatial positional encoding $\gamma_{\mathbf{x}}(\mathbf{x})$.
    $$ \mathbf{v}_{\text{spatial-time}} = \text{concat}(\gamma_{\mathbf{x}}(\mathbf{x}), \gamma_{t}(t)) $$
3.  **Forward Pass:** Pass $\mathbf{v}_{\text{spatial-time}}$ through the MLP's spatial feature layers to obtain a feature vector $\mathbf{f}$.
4.  **Direction Conditioning:** Concatenate the view direction encoding $\gamma_{\mathbf{d}}(\mathbf{d})$ to $\mathbf{f}$ and pass through the remaining MLP layers to predict $\sigma$ and $\mathbf{c}$.
5.  **Volume Rendering:** Integrate $\sigma$ and $\mathbf{c}$ along the ray using the standard differentiable volume rendering equation [Parent](3065-nerf-hyperparameters.md) to produce the pixel color.
6.  **Optimization:** Minimize the mean squared error between rendered colors and ground truth images across all frames in the training set.

**Parameters:**
*   **Training Configuration:** Use the same hyperparameters as the standard NeRF real forward-facing experiments (e.g., learning rate, number of samples per ray, network depth/width) [S1].
*   **Time Encoding Bands ($L_t$):** Not explicitly specified in sources, but typically matches or is lower than spatial bands.

**Edge Cases and Pitfalls:**
*   **Temporal Aliasing:** If the time sampling frequency is too low relative to scene motion, high-frequency temporal details may be lost.
*   **Blurriness:** The method suffers from blurry predictions and lacks fine detail compared to more complex dynamic NeRF variants [S1].
*   **Lack of Dynamics Reasoning:** It cannot explicitly model or predict scene dynamics (e.g., object trajectories), only interpolating appearance changes [S1].

## Subtopics

### Temporal Compression of NeRF Models
A compression strategy for sequences of NeRF models that exploits temporal redundancy. It treats the first model as a key-frame and compresses only the deltas between successive models [S2].
*   **Definition:** Exploits similarity between neighboring NeRF models by storing only weight differences [S2].
*   **Steps:**
    1.  Train the first NeRF from scratch [S2].
    2.  Initialize subsequent NeRFs using the weights of the previous frame's trained NeRF to encourage temporal redundancy [S2].
    3.  Calculate the delta between the current frame's weights and the previous frame's weights [S2].
    4.  Apply exponent-based quantization and Huffman coding to the deltas [S2].
    5.  Restore a model by accumulating decoded deltas to the reference model [S2].
*   **When to choose:** When storage or transmission bandwidth is limited for dynamic NeRF sequences [S2].

## Operating characteristics
*   **Time Complexity:** Training time is comparable to static NeRF, though convergence may be slower due to the increased capacity required to model temporal variations [S1].
*   **Memory Complexity:** Slightly higher than static NeRF due to the additional time encoding dimension, but negligible compared to the MLP size.
*   **Performance:** Improves quality over vanilla NeRF on dynamic scenes but yields lower PSNR/SSIM than specialized dynamic methods (e.g., Neural Scene Graphs) [S1].
*   **Strengths:** Simplicity of implementation; minimal architectural change.
*   **Limitations:** Blurry predictions; inability to reason about scene dynamics; significant ghosting if motion is complex [S1].
*   **When to use:** As a baseline for dynamic NeRF research; simple dynamic scenes with slow motion.
*   **When not to use:** High-speed dynamics; scenes requiring explicit motion modeling; applications demanding high temporal fidelity.

## Background
Originated as a baseline modification to evaluate the limits of implicit representations for dynamic scenes. The approach was introduced in "Neural Scene Graphs for Dynamic Scenes" (CVPR 2021) [S1] to demonstrate that simply adding a time input is insufficient for complex dynamic scene reconstruction, motivating the development of more advanced methods like scene graphs. It is widely used as a reference point in dynamic NeRF literature.

## Sources
[S1] Neural Scene Graphs for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.00288, fetched 2026-10-26
[S2] Volumetric Video Compression Through Neural-based Representation, https://doi.org/10.1145/3652212.3652220, fetched 2026-11-05
