# MVSplat

Also called: Feed-forward 3DGS, Single-pass 3DGS

## Overview
MVSplat is a feed-forward deep neural network that directly regresses 3D Gaussian Splatting (3DGS) parameters from sparse multi-view images in a single pass, bypassing the per-scene gradient descent optimization required by standard 3DGS [S1][S2, unofficial]. It distinguishes itself from its parent [Neural Network and Feed-Forward Approaches](9526-neural-network-and-feed-forward-approaches.md) by building upon PixelSplat, combining multi-view feature extraction through transformers and cost volumes, which are then processed by a U-Net [S1]. Unlike per-scene optimization methods, MVSplat learns a prior that generalizes to unseen scenes, enabling immediate reconstruction without iterative training [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ sparse multi-view images $\{I_1, \dots, I_N\}$ and their corresponding camera parameters (extrinsics and intrinsics).
*   **Output:** A set of 3D Gaussian primitives $\{\mathbf{\mu}_i, \mathbf{R}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_i\}$, where $\mathbf{\mu}_i$ is the center, $\mathbf{R}_i$ is the rotation, $\mathbf{s}_i$ is the scale, $\alpha_i$ is the opacity, and $\mathbf{c}_i$ is the color (typically represented as spherical harmonics coefficients).

**Data Structures:**
The network architecture typically involves:
1.  **Feature Extractor:** A backbone (e.g., ResNet or Transformer) extracts 2D features from each input image.
2.  **Cost Volume:** A 3D cost volume is constructed by matching features across views, often using epipolar constraints or plane-sweeping, to encode geometric depth information.
3.  **U-Net Decoder:** The cost volume and multi-view features are processed by a U-Net to predict Gaussian parameters. The output is a dense or sparse grid of Gaussian properties.

**Steps:**
1.  **Multi-view Feature Extraction:** Extract features $F_i$ from each input image $I_i$ using a shared encoder network.
2.  **Cost Volume Construction:** Construct a cost volume $C$ by aggregating features from all views. This involves projecting features from reference views to target views using known camera poses and computing similarity metrics (e.g., dot product, correlation) at discrete depth levels.
3.  **Depth/Feature Refinement:** Process the cost volume $C$ using a U-Net or similar 3D convolutional network to refine depth estimates and feature representations. This step integrates multi-view context to resolve ambiguities.
4.  **Gaussian Parameter Regression:** Decode the refined features from the U-Net to predict the 3D Gaussian parameters. This can be done by:
    *   Predicting a probability distribution over depth for each pixel, from which Gaussians are sampled [S1].
    *   Directly regressing Gaussian centers, scales, rotations, opacities, and colors for a fixed or adaptive set of primitives.
5.  **Differentiable Rendering (Training Loss):** During training, the predicted Gaussians are rasterized using the standard 3D Gaussian Splatting rasterizer [S1]. The rendered image is compared to the ground truth input images using a loss function (e.g., L1 loss, SSIM, or perceptual loss) to update the network weights.

**Formulas:**
*   **Cost Volume Construction:** For a reference view $I_r$ and a target view $I_t$, the cost volume element $C(x, y, d)$ at pixel $(x, y)$ and depth $d$ is computed as:
    $$ C(x, y, d) = \text{Similarity}(F_r(x, y), F_t(x', y')) $$
    where $(x', y')$ is the projected pixel location in $I_t$ corresponding to depth $d$ in $I_r$.
*   **Gaussian Sampling (if probabilistic):** Gaussians are sampled from a predicted probability distribution $P(x, y, z)$ over 3D space:
    $$ \{\mathbf{\mu}_i\} \sim P(x, y, z) $$
    Other parameters ($\mathbf{R}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_i$) are predicted conditioned on the sampled centers.

**Parameters:**
*   **Number of Views:** Typically 2-4 views are used for training and inference, though the method can generalize to sparse inputs.
*   **Depth Discretization:** The number of discrete depth levels in the cost volume affects memory usage and depth resolution.
*   **Network Depth:** The U-Net depth and feature dimensions determine the model's capacity.

**Edge Cases and Pitfalls:**
*   **Occlusions:** Sparse views may lead to occlusions, causing missing geometry. The cost volume and U-Net must handle occluded regions gracefully, often by predicting low opacity or relying on learned priors.
*   **Scale Ambiguity:** Without known camera intrinsics or a reference scale, the reconstructed scene may suffer from scale ambiguity. MVSplat assumes known camera parameters [S1].
*   **Generalization Gap:** Performance may degrade on scenes significantly different from the training distribution.

## Subtopics

### Sparse Input Reconstruction via Generalizability
Methods that use learned priors to reconstruct 3D scenes, often allowing reconstruction from a single input image after training on multiple views [S3].
*   **Step:** Synthesize additional views through generative models to augment input data [S3]. Use a feed-forward model to directly generate Gaussian properties from input images [S3].
*   **Pitfall:** Synthesizing additional views via generative models is computationally intensive and bounded by the capabilities of the generative model [S3]. Pixel-aligned Gaussians are distributed nearly evenly in space, struggling to represent high-frequency details and smoother regions with an appropriate number of Gaussians [S3].
*   **Comparison:** Uses learned priors for fast feed-forward inference, favoring speed and cross-scene generalization, but sometimes at the expense of geometric fidelity compared to regularization-based methods [S3].
*   **Usage:** PixelSplat samples Gaussians from dense probability distributions [S3]. Splatter Image uses a 2D image-to-image network to map an input image to a 3D Gaussian per pixel [S3].

## Operating characteristics
*   **Time Complexity:** Inference is $O(1)$ with respect to the number of optimization iterations, as it is a single forward pass through the neural network. The complexity is dominated by the feature extraction and U-Net processing, which is linear in the number of input pixels and views.
*   **Memory Complexity:** Memory usage is determined by the neural network size and the cost volume dimensions. The cost volume can be large, requiring significant VRAM, especially for high-resolution images and many depth levels.
*   **Measured Performance:** MVSplat enables real-time rendering speeds (900+ FPS) due to the absence of per-scene optimization [S2, unofficial]. Quality is generally lower than optimized 3DGS but significantly faster.
*   **Scaling:** Performance scales with the number of input views and image resolution. Larger networks and cost volumes improve accuracy but increase memory and compute requirements.
*   **Strengths:** Fast inference, no per-scene optimization required, generalizes to unseen scenes.
*   **Limitations:** Lower geometric fidelity compared to optimized methods, sensitive to camera pose accuracy, potential scale ambiguity.
*   **When to use:** Applications requiring real-time reconstruction from sparse views, such as mobile AR/VR or rapid scene capture.
*   **When not to use:** Applications requiring high geometric accuracy or dense reconstruction from very sparse inputs where learned priors are insufficient.

## Background
MVSplat was introduced as a method to accelerate 3D Gaussian Splatting reconstruction by replacing the iterative optimization loop with a feed-forward neural network [S1][S2, unofficial]. It builds upon PixelSplat, which introduced epipolar transformers for multi-view feature extraction [S1]. MVSplat extends this by incorporating a U-Net to process cost volumes, improving the accuracy of Gaussian parameter regression [S1]. The method is part of a broader trend towards generalizable 3D reconstruction, where neural networks learn priors over scenes to enable fast inference on new data [S3].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
