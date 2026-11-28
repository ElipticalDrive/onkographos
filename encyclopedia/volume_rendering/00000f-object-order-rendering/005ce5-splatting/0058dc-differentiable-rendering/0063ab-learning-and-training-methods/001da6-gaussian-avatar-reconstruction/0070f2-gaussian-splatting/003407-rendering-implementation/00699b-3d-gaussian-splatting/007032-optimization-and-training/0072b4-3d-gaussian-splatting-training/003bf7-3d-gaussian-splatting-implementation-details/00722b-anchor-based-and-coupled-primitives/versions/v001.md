# Anchor-based and Coupled Primitives

Also called: Anchor-based Gaussian Splatting, Coupled Gaussian Primitives, Anchor3DGS, SOGS

## Overview
This family replaces the standard pixel-aligned or dense point-cloud initialization of 3D Gaussian Splatting with a compact, global set of "anchors" that predict or define the final Gaussian primitives. By decoupling the number of Gaussians from the input resolution or view count, these methods reduce memory footprint and enable feed-forward or sparse-view reconstruction. They distinguish themselves from standard 3DGS by introducing a hierarchical or statistical dependency between a small set of anchors and the larger set of rendered primitives, allowing for better generalization in sparse-view settings and reduced parameter counts in dense settings.

## Shared principles
All methods in this family rely on a fixed or limited budget of $N$ anchor entities, distinct from the final $M$ Gaussian primitives ($M \gg N$ or $M$ is derived from $N$).

1.  **Anchor Initialization:** Anchors are initialized from a sparse point cloud, a geometry backbone prediction, or a fixed grid, rather than directly from every pixel or dense point.
2.  **Prediction/Decoupling:** A network or statistical model maps anchor features to Gaussian attributes (position, scale, rotation, color, opacity). This step may involve:
    *   **Coupling:** Predicting residual offsets or features for coupled primitives based on anchor primitives [20018-coupled-gaussian-primitives-rendering.md].
    *   **Statistical Augmentation:** Using second-order statistics (covariance) of anchor features to augment feature vectors before prediction [13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md].
    *   **Contextual Aggregation:** Using cross-anchor attention or occlusion-aware multi-view aggregation to refine anchor features before decoding [21327-anchor3dgs.md].
3.  **Rendering:** The final set of Gaussians is rendered using standard differentiable rasterization. The rendering pipeline often includes a visibility filter for the anchors to prune invisible regions before primitive prediction [20018-coupled-gaussian-primitives-rendering.md].

## Subtopics

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
SOGS improves rendering quality and reduces model size by introducing second-order statistics into the anchor feature representation. Unlike standard anchor methods that treat feature dimensions independently, SOGS computes a covariance matrix across anchor features to capture co-variation patterns. It selects the top $M$ eigenvectors of this covariance matrix to augment the anchor features, allowing for smaller feature vectors without sacrificing detail [13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md]. It also employs a selective gradient loss based on Sobel operators to focus optimization on hard-to-render regions [13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md]. Choose this method when high fidelity is required with reduced feature dimensions.

### [Coupled Gaussian Primitives Rendering](20018-coupled-gaussian-primitives-rendering.md)
This method decomposes Gaussian primitives into anchor primitives and coupled primitives. Each anchor primitive predicts multiple coupled primitives, creating a hierarchical structure. The coupled primitives' features are stored as residuals relative to the anchors, significantly reducing storage and enabling compression. The rendering pipeline explicitly filters visible anchors before predicting the full set of primitives, optimizing computation [20018-coupled-gaussian-primitives-rendering.md]. Choose this method when memory efficiency and compression are primary goals.

### Anchor3DGS
Anchor3DGS is a pose-free, feed-forward framework that replaces pixel-aligned generation with a global anchor structure. It uses a geometry backbone (e.g., VGGT) to predict depth and poses, then samples anchors using entropy-guided strategies to concentrate capacity on complex regions. It employs occlusion-aware multi-view aggregation and cross-anchor self-attention to propagate context to poorly observed areas, mitigating holes in sparse-view settings. The final rendering is refined by an enhancement UNet [21327-anchor3dgs.md]. Choose this method for sparse-view novel view synthesis where pose estimation is unreliable or unavailable.

## Background
The concept of anchor-based representations in neural rendering originated from the need to scale Gaussian Splatting to large scenes and sparse views. Standard 3DGS scales linearly with scene complexity and view count, leading to memory issues. Anchor-based methods, such as Scaffold-GS and Anchor3DGS, were developed to decouple representation size from input resolution. Coupled primitives extend this by introducing hierarchical dependencies to further compress the representation. These methods are primarily used in novel view synthesis, particularly in scenarios with limited input views or constrained computational resources.

