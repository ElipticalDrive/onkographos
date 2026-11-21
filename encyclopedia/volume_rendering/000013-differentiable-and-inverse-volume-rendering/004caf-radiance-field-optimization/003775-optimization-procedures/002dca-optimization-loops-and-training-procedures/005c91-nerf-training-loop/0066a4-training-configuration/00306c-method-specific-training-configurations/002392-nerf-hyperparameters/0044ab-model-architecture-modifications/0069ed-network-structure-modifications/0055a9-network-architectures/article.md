# Network Architectures
Also called: NeRF Network Architectures

## Overview
This article details specific modifications to the standard Multi-Layer Perceptron (MLP) architecture used in Neural Radiance Fields (NeRF) to address constraints such as inference speed, memory footprint, dynamic scene representation, and object manipulation. While the standard NeRF architecture [Parent](11313-model-architecture-modifications.md) employs two MLPs ($\Phi_{pos}$ and $\Phi_{shade}$) to map spatial and directional inputs to density and color, these variants introduce structural changes including multi-input multi-output (MIMO) parallel processing, sparse latent spaces, feature distillation, and residual modeling. These methods aim to improve training convergence, reduce parameter counts, or enable specific capabilities like continual learning and transparent object handling, distinguishing them from the baseline dense MLP approach.

## How it works

### Standard NeRF Baseline
The baseline architecture [S1] employs two MLPs:
1.  **Position MLP ($\Phi_{pos}$)**: Takes spatial position $\mathbf{x}$ and positional encoding $\gamma(\mathbf{x})$ to produce a feature vector $\mathbf{h}$.
2.  **Shading MLP ($\Phi_{shade}$)**: Takes $\mathbf{h}$ and directional encoding $\gamma(\mathbf{d})$ to predict volume density $\sigma$ and view-dependent color $\mathbf{c}$.

### MobileNeRF Architecture
MobileNeRF [S1] modifies the standard architecture to exploit the polygon rasterization pipeline for efficient rendering on mobile devices. It replaces the continuous volumetric integration with a discrete, texture-based approach.

**Data Structures:**
*   **Texture Images**: Two PNG images are used to store 8-dimensional feature vectors. Each PNG has 4 channels; the two images provide 8 channels total.
*   **Alpha Channel Encoding**: The binary alpha (opacity) channel is squeezed into the first feature channel. If the first feature channel is non-zero, alpha is 1; if zero, alpha is 0. This avoids storing a separate alpha texture.
*   **Texture Constraints**: Texture sizes must be powers of 2 and at most $4096 \times 4096$. Larger feature sets are split across multiple texture images.

**Network Components:**
*   **MLP A and F**: These MLPs are designed using the standard NeRF architecture but with hidden layer sizes increased from 256 to 384. They are used only during training to generate the texture features and are not used during inference.
*   **MLP H**: A small MLP identical to the one used in SNeRG, consisting of two hidden layers, each with 16 neurons.

**Steps:**
1.  **Training**: Use MLPs A and F (384 hidden units) to predict features for spatial locations. Encode these features into the two PNG texture images.
2.  **Inference**: Use the rasterization pipeline to sample from the texture images based on the projected geometry. Use MLP H to process the sampled features for final rendering.

### DM-NeRF (Point-NeRF Adaptation)
DM-NeRF [S2, unofficial] adapts Point-NeRF for 3D scene geometry decomposition and manipulation. It uses a neural point cloud representation rather than a dense voxel or grid.

**Inputs:**
*   Spatial locations of neural points.
*   Object codes inferred from a DM-NeRF model.

**Steps:**
1.  **Training Data Generation**: Use ground truth dense point clouds (e.g., 400 million points per scene) instead of MVSNet-generated points to create the neural point cloud, ensuring high-quality scene-level representation [S2, unofficial].
2.  **Inference - Object Code Inference**: Feed the spatial locations of all neural points into the DM-NeRF model to infer corresponding object codes [S2, unofficial].
3.  **Manipulation**: Identify points to be edited based on manipulation information. Transform the corresponding neural points to desired new locations [S2, unofficial].
4.  **Rendering**: Render the image with object manipulation from a given view using point-based volume rendering in Point-NeRF with the modified neural point cloud [S2, unofficial].

### Pyramid NeRF
Pyramid NeRF [S3] accelerates training and inference through an image pyramid and frequency optimization strategy.

**Steps:**
1.  **Frequency Optimization**: Utilize a "low-frequency-first, high-frequency-second" approach.
2.  **Network Segmentation**: Substitute large MLPs with thousands of smaller MLPs via progressive segmentation.
3.  **Depth-Aware Sampling**: Introduce a sampling approach aware of depth to enhance rendering quality and hasten training convergence [S3].

### Residual-NeRF-S
Residual-NeRF-S [S4] is an ablation variant of Residual-NeRF where background and residual NeRFs are merged by simply adding their outputs before activation, instead of using a Mixnet.

**Formulas:**
The rendered color $\hat{\mathbf{C}}(\mathbf{r})$ along a ray $\mathbf{r}$ is computed as:
$$
\hat{\mathbf{C}}(\mathbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-(\sigma_{bg} + \sigma_{res})_i \delta_i)) (\mathbf{c}_{bg} + \mathbf{c}_{res})_i
$$
where:
*   $T_i = \exp(-\sum_{j=1}^{i-1} (\sigma_{bg} + \sigma_{res})_j \delta_j)$ is the transmittance.
*   $\delta_i$ is the distance between sample points.
*   $(\mathbf{c}'_{bg}, \sigma_{bg}) = \Phi_{bg}(h(\mathbf{x}, \theta, \phi))$ are the background outputs.
*   $(\mathbf{c}'_{res}, \sigma_{res}) = \Phi_{res}(h(\mathbf{x}, \theta, \phi))$ are the residual outputs.
*   The combined color $\mathbf{c}_{bg+res}$ is calculated as:
    $$
    \mathbf{c}_{bg+res} = S(\mathbf{c}'_{bg} + \mathbf{c}'_{res})
    $$
    where $S(x) = \frac{1}{1 + e^{-x}}$ is the sigmoid activation function [S4].

**Comparison:**
Residual-NeRF-S produces worse depth and view reconstruction compared to Residual-NeRF with Mixnet, especially for partially opaque objects [S4].

## Subtopics

### [MIMO-DONeRF](16217-mimo-donerf.md)
MIMO-DONeRF applies the Multi-Input Multi-Output (MIMO) neural rendering concept to the DONeRF architecture. It modifies the shading network to accept multiple spatial inputs and view directions simultaneously, producing multiple color and density outputs per forward pass. This improves inference speed compared to increasing the number of samples in standard DONeRF, while maintaining or improving image quality by allowing the network to accumulate neighbor information through grouped samples. It differs from standard DONeRF by processing $N_p$ samples in parallel rather than sequentially, and from MIMO-NeRF by being specifically adapted to the two-network structure (depth oracle and shading) of DONeRF.

### [Sparse Latent Space for 4D Novel View Synthesis (SLS4D)](18112-sparse-latent-space-for-4d-novel-view-synthesis-sls4d.md)
SLS4D represents dynamic radiance fields using a learnable sparse latent space, reducing network parameters compared to dense grid discretizations while maintaining high-frequency detail via attention-based interpolation. It differs from dense grid methods (like TiNeuVox) by avoiding the $O(N^3TF)$ space complexity of 4D grids, and from per-image latent code methods by offering better generalization across time through continuous temporal interpolation.

### [Neural Feature Fusion Fields (N3F)](18114-neural-feature-fusion-fields-n3f.md)
Neural Feature Fusion Fields (N3F) distills dense 2D image features from a pre-trained teacher network into a 3D neural field (student) via differentiable volumetric rendering, enabling 3D-aware feature representation. Unlike Semantic NeRF or Panoptic NeRF which fuse discrete semantic labels, N3F fuses continuous dense feature vectors, allowing for open-world applications without manual labels. It differs from standard feature distillation because the student operates in 3D world space (ensuring spatial consistency) while the teacher operates in 2D image space.

### [CLNeRF Architecture with Embeddings](18159-clnerf-architecture-with-embeddings.md)
CLNeRF is a model architecture modification for NeRF that enables continual learning by incorporating scene changes (appearance and geometry) over time within a single model, preventing model size growth as new scans are added. It sets itself apart from sibling methods like MIMO-DONeRF and SLS4D by using explicit, trainable per-scan embedding vectors rather than latent space compression or multi-instance networks, allowing for efficient updates and low overhead for generative replay.

### Point-NeRF Adaptation for Object Manipulation
This variant, detailed under DM-NeRF [S2, unofficial], adapts Point-NeRF for object manipulation by using ground truth dense point clouds during training and inferring object codes for neural points during inference. It enables geometry decomposition and manipulation by transforming neural points based on manipulation information and rendering via point-based volume rendering.

### Pyramid NeRF
Pyramid NeRF [S3] accomplishes rapid NeRF training and inference through the application of an image pyramid and a strategy based on frequency optimization. It utilizes a "low-frequency-first, high-frequency-second" approach, substitutes large MLPs with thousands of smaller ones via progressive segmentation, and introduces depth-aware sampling. It enhances training speed by 15x and inference speed by 805x relative to the original NeRF [S3].

### Residual-NeRF-S
Residual-NeRF-S [S4] is an ablation variant of Residual-NeRF where background and residual NeRFs are merged by simply adding their outputs before activation, instead of using a Mixnet. It uses a sigmoid function to combine the color outputs. It produces worse depth and view reconstruction compared to Residual-NeRF with Mixnet, especially for partially opaque objects [S4].

## Operating characteristics
*   **MobileNeRF**: Optimized for mobile hardware constraints (texture size limits, power of 2 dimensions). Training time is increased due to larger MLPs (384 hidden units), but inference is accelerated by leveraging rasterization pipelines [S1].
*   **Pyramid NeRF**: Offers significant speedups: 15x faster training and 805x faster inference compared to original NeRF [S3].
*   **Residual-NeRF-S**: Simpler architecture than Residual-NeRF with Mixnet, but results in inferior depth and view reconstruction quality [S4].
*   **DM-NeRF**: Requires high-quality input data (ground truth dense point clouds) for training to ensure quality, which may be difficult to obtain for real-world scenes [S2, unofficial].

## Background
NeRF architectures have evolved from the original two-MLP design [Parent](11313-model-architecture-modifications.md) to address specific limitations in speed, memory, and functionality. MobileNeRF [S1] was introduced to enable neural field rendering on mobile devices by exploiting rasterization. Pyramid NeRF [S3] was proposed to accelerate training and inference through frequency optimization and network segmentation. Residual-NeRF [S4] was developed to improve depth perception and training speed for scenes with transparent objects. DM-NeRF [S2, unofficial] extends Point-NeRF for scene decomposition and manipulation.

## Sources
[S1] MobileNeRF: Exploiting the Polygon Rasterization Pipeline for Efficient Neural Field Rendering on Mobile Architectures, https://doi.org/10.1109/cvpr52729.2023.01590, fetched 2026-10-26
[S2] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
[S3] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S4] Residual-NeRF: Learning Residual NeRFs for Transparent Object Manipulation, https://doi.org/10.1109/icra57147.2024.10611224, fetched 2026-10-26
