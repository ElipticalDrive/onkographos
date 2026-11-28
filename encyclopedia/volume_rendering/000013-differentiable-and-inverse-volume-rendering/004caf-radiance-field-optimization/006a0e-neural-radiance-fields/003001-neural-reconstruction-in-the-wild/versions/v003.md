# Neural Reconstruction in the Wild


## Overview
Neural Reconstruction in the Wild is a neural radiance field method for reconstructing large-scale, real-world scenes [S1, unofficial]. It addresses the low geometric reconstruction quality and coarse details of standard NeRF in expansive environments by using surface-guided sampling to improve reconstruction accuracy [S1, unofficial].

## How it works
**Inputs:** Not given in the sources.
**Outputs:** Not given in the sources.
**Data Structures:** Not given in the sources.

**Steps:**
1. Apply surface-guided sampling to improve reconstruction accuracy for large-scale scenes [S1, unofficial].
2. Evaluate the neural radiance field and render images using the standard volume rendering equation [Parent].

**Parameters:** Not given in the sources.

**Edge Cases and Pitfalls:** Not given in the sources.

## Subtopics

### Inverse Projection for Single-View Reconstruction
A technique to perform single-view reconstruction by optimizing the periodic activation parameters (frequencies and phase shifts) of a frozen implicit radiance field generator to match a target image, without modifying the network weights [S2]. This method freezes the MLP weights and optimizes the frequencies $\gamma_i$ and phase shifts $\beta_i$ for each layer $i$ [S2].

**Steps:**
1. Freeze the parameters of the implicit representation (the MLP weights) [S2].
2. Identify the learnable parameters: frequencies $\gamma_i$ and phase shifts $\beta_i$ for each MLP layer $i$ [S2].
3. Initialize $\gamma_i$ and $\beta_i$ to $\bar{\gamma}_i$ and $\bar{\beta}_i$, the center of mass of frequencies and phase shifts for each layer [S2].
4. Calculate $\bar{\gamma}_i$ and $\bar{\beta}_i$ by averaging the frequencies and phase shifts of 10,000 random noise vector inputs [S2].
5. Run gradient descent to minimize the mean-squared-error (MSE) image reconstruction loss [S2].
6. Apply an L2 penalty with weight 0.1 to prevent $\gamma_i$ and $\beta_i$ from straying too far from $\bar{\gamma}_i$ and $\bar{\beta}_i$ [S2].
7. Optimize using the Adam optimizer over 700 iterations [S2].
8. Initialize the learning rate to 0.01, decaying by a factor of 0.5 every 200 iterations [S2].

**Parameters:**
* L2 penalty weight: 0.1 [S2].
* Optimization iterations: 700 [S2].
* Initial learning rate: 0.01 [S2].
* Learning rate decay factor: 0.5 every 200 iterations [S2].
* Number of random noise vectors for initialization: 10,000 [S2].

## Operating characteristics
* **Performance:** Improves reconstruction accuracy for large-scale scenes compared to standard NeRF [S1, unofficial].
* **Strengths:** Better geometric detail in large-scale environments [S1, unofficial].
* **Limitations:** Relies on surface-guided sampling [S1, unofficial].
* **When to use:** Large-scale, real-world scene reconstruction [S1, unofficial].

## Background
The method was introduced in "Neural Reconstruction in the Wild" at SIGGRAPH 2022 [S1, unofficial]. It addresses the limitations of NeRF in large-scale scenes, such as coarse geometry, by leveraging surface-guided sampling to enhance neural implicit surface reconstruction quality [S1, unofficial].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis, https://doi.org/10.1109/cvpr46437.2021.00574, fetched 2026-10-26
