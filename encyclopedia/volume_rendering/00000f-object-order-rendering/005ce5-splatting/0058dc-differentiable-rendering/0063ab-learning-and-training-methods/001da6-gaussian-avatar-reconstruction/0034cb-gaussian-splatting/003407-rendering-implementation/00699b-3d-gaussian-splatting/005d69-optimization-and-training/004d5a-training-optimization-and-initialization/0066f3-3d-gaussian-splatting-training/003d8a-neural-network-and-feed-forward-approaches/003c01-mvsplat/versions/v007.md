# MVSplat

Also called: Amortized 3D Reconstruction, Feed-forward 3DGS, Feedforward 3D Reconstruction, Single-pass 3DGS

## Overview
MVSplat is a feed-forward deep neural network that directly regresses 3D Gaussian Splatting (3DGS) parameters from sparse multi-view images in a single pass, bypassing the per-scene gradient descent optimization loop required by standard 3DGS [S1][S2, unofficial]. It builds upon PixelSplat by combining multi-view feature extraction through transformers and cost volumes, which are then processed by a U-Net [S1]. Unlike iterative methods that optimize primitives for a specific scene, MVSplat provides a universal reconstruction model trained on diverse object types, offering useful 3D priors for subsequent 3D optimizations [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ sparse multi-view images of a scene.
*   **Output:** A set of 3D Gaussian primitives, defined by their position, covariance (scale and rotation), opacity, and spherical harmonics coefficients for color.

**Data Structures:**
The network architecture processes image features into a cost volume, which is decoded into Gaussian parameters. The specific internal tensor layouts for the transformer tokens and U-Net feature maps are not detailed in the sources [G: Standard transformer encoder/decoder and U-Net convolutional block structures are assumed].

**Algorithm Steps:**
1.  **Multi-view Feature Extraction:** Extract features from the input images using a transformer-based backbone. This step generalizes features between views, similar to the epipolar transformer used in PixelSplat [S1].
2.  **Cost Volume Construction:** Construct a cost volume representing the similarity or correlation of features across the input views. This volume encodes depth information [S1].
3.  **Depth and Gaussian Regression:** Process the cost volume through a U-Net architecture. The U-Net decodes the cost volume to predict depth maps and directly regresses the 3D Gaussian parameters (position, scale, rotation, opacity, color) aligned along camera-to-pixel rays [S1].
4.  **Rendering (Optional for Training):** The predicted 3D Gaussians are rendered into novel views using the differentiable 3D Gaussian Splatting rasterizer to compute photometric losses [S2, unofficial].

**Formulas:**
The sources do not provide explicit mathematical formulas for the feature extraction or U-Net decoding steps. The training objective minimizes the Mean Squared Error (MSE) between the rendered novel views and the ground truth novel views [S3].

**Parameters:**
*   **Training Views:** 20 source images are used as inputs during training [S3].
*   **Evaluation Views:** 5 novel views are used for calculating MSE losses during training evaluation [S3].
*   **Finetuning Inputs:** 80 views are used as inputs to the feed-forward model for initialization in the finetuning regime [S3].

**Edge Cases and Pitfalls:**
*   **Sparse Views:** The method is designed for sparse inputs, but performance may degrade if the input views have insufficient overlap or extreme baseline differences not covered in the training distribution [S1].
*   **Scale Ambiguity:** Unlike methods that use known camera poses, MVSplat relies on the cost volume and transformer generalization. If the input views lack sufficient geometric cues, scale ambiguity may arise, though the sources note that NoPoSplat addresses this specifically by eliminating known camera pose requirements [S1].

## Subtopics

### [Sparse Input Reconstruction via Generalizability](11788-sparse-input-reconstruction-via-generalizability.md)
Addresses reconstructing 3D scenes from a limited number of input views (sparse views) or even a single image, bypassing the iterative per-scene optimization required by standard 3D Gaussian Splatting. It achieves this by employing learned priors within a feed-forward deep neural network that directly regresses 3D Gaussian parameters from input images in a single pass. This approach favors inference speed and cross-scene generalization over the high geometric fidelity typically achieved by regularization-based methods that rely on per-scene gradient descent.

### [VolSplat Architecture and Training Pipeline](13066-volsplat-architecture-and-training-pipeline.md)
VolSplat is a feed-forward deep neural network that directly regresses 3D Gaussian Splatting (3DGS) parameters from sparse multi-view images in a single pass, bypassing the per-scene gradient descent optimization required by standard 3DGS [Parent](9213-mvsplat.md). It distinguishes itself by lifting 2D features into a sparse 3D voxel grid via plane-sweep depth prediction and predicting one Gaussian primitive per occupied voxel, ensuring scene-adaptive primitive allocation while maintaining stable sparse 3D decoding.

## Operating characteristics
*   **Time Complexity:** The method performs inference in a single forward pass, making it significantly faster than iterative optimization methods that require hundreds or thousands of gradient descent steps per scene [S2, unofficial].
*   **Memory Complexity:** Memory usage is determined by the neural network's parameter count and the intermediate feature maps during the forward pass. It does not require storing large per-scene optimization states or dense point clouds during the initial reconstruction phase [S3].
*   **Measured Performance:** The model generates compelling reconstructions with just a single forward pass [S3]. After fine-tuning, it is on par with overfitting baselines in color accuracy (PSNR, LPIPS) but largely outperforms hash-based baselines in depth error [S3].
*   **Strengths:** Bypasses per-scene gradient descent; provides useful 3D priors for subsequent 3D optimizations; works on a variety of object types [S3].
*   **Limitations:** As a feed-forward model, it may not achieve the same level of geometric fidelity as iterative methods that overfit to a specific scene's data, particularly in areas with limited viewpoint coverage [S3].

## Background
MVSplat was introduced as a method to combine multi-view feature extraction through transformers and cost volumes, processed by a U-Net, building upon the foundations of PixelSplat [S1]. It is part of a broader trend towards amortized 3D reconstruction, where a neural network is trained to generalize across scenes rather than optimizing for a single instance [S2, unofficial]. The model was trained on all CO3Dv2 categories to create a universal reconstruction model capable of handling various object types [S3].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] Lightplane: Highly-Scalable Components for Neural 3D Fields, https://doi.org/10.1109/3dv66043.2025.00058, fetched 2026-10-29
