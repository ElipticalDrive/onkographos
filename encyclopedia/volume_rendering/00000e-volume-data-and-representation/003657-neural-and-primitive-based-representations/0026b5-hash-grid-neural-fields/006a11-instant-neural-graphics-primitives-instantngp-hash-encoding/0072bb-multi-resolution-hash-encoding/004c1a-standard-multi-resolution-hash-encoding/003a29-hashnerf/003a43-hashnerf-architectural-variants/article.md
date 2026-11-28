# HashNeRF Architectural Variants

Also called: HashNeRF Network Architecture Variants

## Overview
HashNeRF Architectural Variants are specific modifications to the shallow neural network decoder used in HashNeRF, designed to optimize the trade-off between rendering accuracy (PSNR) and training time [S1, unofficial]. While the standard HashNeRF uses a simple multi-layer perceptron (MLP) with linear layers, these variants explore convolutional layers, residual connections, and deeper network structures to improve feature extraction and color prediction [S1, unofficial]. They differ from sibling methods like Zip-NeRF by focusing on architectural ablation of the MLP decoder rather than changing the underlying hash encoding or density estimation mechanism [S1, unofficial].

## How it works
The variants modify the "Shallow Neural Network" stage of HashNeRF, which takes the hashed feature vector as input and outputs density ($\sigma$) and color ($C$) [S1, unofficial]. The input to these networks is the concatenated feature vector from the multi-resolution hash encoding [S1, unofficial].

The following four variants were proposed and evaluated on the synthetic Lego scene (100 training views, 200 test views) with data downsampled by a factor of 4. While the main text notes training for 2,500 iterations, the reported PSNR values in the comparison table were generated over 1,000 iterations [S1, unofficial].

### Variant 1: Convolutional Layers
This variant replaces the linear layers of the standard MLP with 1D convolutional layers to reduce trainable parameters and potentially speed up training [S1, unofficial].

**Pipeline 1 (Density/Color Network):**
1. `Conv1d(48, 32, kernel_size=3)`
2. `Conv1d(32, 16, kernel_size=3)`
3. `Conv1d(16, 8, kernel_size=3)`
4. `Conv1d(8, 4, kernel_size=3)`

The reported PSNR is 15.831 with a training time of 363.415 seconds [S1, unofficial].

### Variant 2: Residual Connections from Initial Input
This variant adds residual connections from the initial input to each subsequent layer in the sigma and color MLP layers to preserve spatial information from the first layer [S1, unofficial].

**Pipeline 1 (Sigma Network):**
1. `Linear(32, 64, bias=False)`
2. `ReLU`
3. `Concat(input_views, output_1)`
4. `Linear(79, 16, bias=False)`
5. `ReLU`
6. `Concat(input_views, output_2)`

**Pipeline 2 (Color Network):**
1. `Linear(31, 64, bias=False)`
2. `ReLU`
3. `Concat(input_views, output_3)`
4. `Linear(79, 64, bias=False)`
5. `ReLU`
6. `Concat(input_views, output_4)`
7. `Linear(79, 3, bias=False)`

The reported PSNR is 25.873 with a training time of 306.751 seconds [S1, unofficial].

### Variant 3: Residual Connections from Alternate Layers
This variant improves upon Variant 2 by adding residual connections from 2 layers before each layer, preserving information from previous layers while allowing effective parameter tuning [S1, unofficial].

**Pipeline 1:**
1. `Linear(32, 64, bias=False)`
2. `ReLU`
3. `Linear(79, 16, bias=False)`
4. `ReLU`
5. `Concat(input_views, output_1)`

**Pipeline 2:**
1. `Linear(31, 64, bias=False)`
2. `ReLU`
3. `Concat(output_1, output_2)`
4. `Linear(79, 64, bias=False)`
5. `ReLU`
6. `Concat(output_2, output_3)`
7. `Linear(79, 3, bias=False)`

The reported PSNR is 26.547 with a training time of 305.208 seconds [S1, unofficial].

### Variant 4: Deeper Network with Residuals
This variant makes the network deeper by adding 3 linear layers along with residuals from the sigma layers to increase the number of parameters and improve color output fidelity [S1, unofficial].

**Pipeline 1:**
1. `Linear(32, 64, bias=False)`
2. `ReLU`
3. `Linear(64, 16, bias=False)`
4. `ReLU`
5. `Concat(input_views, output_1)`

**Pipeline 2:**
1. `Linear(20, 64, bias=False)`
2. `ReLU`
3. `Linear(64, 64, bias=False)`
4. `ReLU`
5. `Concat(input_views, output_2)`

**Pipeline 3:**
1. `Linear(50, 128, bias=False)`
2. `ReLU`
3. `Linear(128, 64, bias=False)`
4. `ReLU`
5. `Linear(64, 3, bias=False)`

The reported PSNR is 27.119 with a training time of 319.669 seconds [S1, unofficial].

## Subtopics

### [Switch-NeRF++ Forward Pass and Loss](17395-switch-nerf-forward-pass-and-loss.md)
Switch-NeRF++ is a large-scale neural radiance field method that employs a Mixture of Experts (MoE) architecture to efficiently render heterogeneous urban scenes. It solves the scalability problem of HashGrid Neural Fields by routing 3D points to specialized hash experts based on local scene complexity, thereby balancing memory usage and rendering accuracy. It differs from standard HashNeRF by introducing a learned gating mechanism that selects a single expert per point, and from Switch-NeRF by upgrading the expert capacity and resolution scaling for higher fidelity in massive environments [17395-switch-nerf-forward-pass-and-loss].

### [Switch-NeRF++ Gating](17411-switch-nerf-gating.md)
Switch-NeRF++ Gating is a mechanism for selecting a single expert from a collection of heterogeneous hash experts based on the input 3D point. It solves the problem of efficiently routing spatial data to specialized representations, enabling joint optimization of experts and the gating network through direct gradient-based training. It is distinguished from its sibling [Switch-NeRF++ Forward Pass and Loss](17395-switch-nerf-forward-pass-and-loss.md) by its specific use of hash encoding within the gating network itself, rather than a pure MLP, which improves efficiency and consistency with the hash experts.

### [GP-NeRF](17415-gp-nerf.md)
GP-NeRF is a unified learning framework that embeds a Neural Radiance Field (NeRF) and a 2D segmentation module to perform context-aware 3D scene perception. It jointly constructs radiance and semantic-embedding fields, facilitating joint volumetric rendering to ensure geometry distribution consistency between the two fields. Unlike prior methods that render semantic labels per pixel independently, GP-NeRF renders rich-semantic features and feeds them into a 2D segmentation module for context-aware perception.

## Operating characteristics
The variants were evaluated on the synthetic Lego scene with 100 training views and 200 test views, with data downsampled by a factor of 4 [S1, unofficial].
*   **Variant 1 (Conv1d):** PSNR 15.831, Training Time 363.415s [S1, unofficial]. This variant performed poorly in terms of accuracy compared to linear variants, despite the goal of reducing parameters [S1, unofficial].
*   **Variant 2 (Residual from Input):** PSNR 25.873, Training Time 306.751s [S1, unofficial]. This variant showed significant improvement over Variant 1 and the baseline, with faster training than Variant 1 [S1, unofficial].
*   **Variant 3 (Residual from Alternate Layers):** PSNR 26.547, Training Time 305.208s [S1, unofficial]. This variant improved accuracy over Variant 2 with similar training time [S1, unofficial].
*   **Variant 4 (Deeper Network):** PSNR 27.119, Training Time 319.669s [S1, unofficial]. This variant achieved the highest PSNR among the variants, indicating that increased depth and parameters can improve color output, at the cost of slightly longer training time [S1, unofficial].

Under iso-time evaluation, none of the extensions conclusively outperformed published baselines, but they characterize which extensions transfer to constrained settings [S1, unofficial].

## Background
These architectural variants were proposed in the context of evaluating low-cost neural radiance fields, specifically abalating the feature-decoding MLP of TensoRF and studying HashNeRF designs [S1, unofficial]. The work aims to understand how architectural changes in the shallow decoder affect performance in resource-constrained environments (reduced view counts, downsampled data) [S1, unofficial]. The multi-level hash encoding model has applications in gigapixel approximation, signed distance function generation, and neural radiance caching, but was customized here for NeRF generation [S1, unofficial].

## Sources
[S1] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S2] What Is a Radiance Field? A Complete Guide, https://radiancefields.com/what-is-a-radiance-field, fetched 2026-10-29 (unofficial)
