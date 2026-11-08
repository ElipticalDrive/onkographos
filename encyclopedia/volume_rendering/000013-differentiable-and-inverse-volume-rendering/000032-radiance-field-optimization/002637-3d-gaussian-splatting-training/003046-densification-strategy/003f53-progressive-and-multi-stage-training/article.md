# Progressive and Multi-stage Training

Also called: Multi-stage Optimization, Coarse-to-Fine Training, Progressive Densification

## Overview
Progressive and Multi-stage Training methods address the instability and poor convergence of single-stage 3D Gaussian Splatting optimization by dividing the training process into distinct phases or stages [S1, unofficial]. Unlike standard approaches that apply densification uniformly from the start, these methods progressively refine the Gaussian representation, balancing global structure capture with local detail refinement to achieve higher rendering quality [7173-hybrid-coarse-to-fine-strategy].

## Shared principles
All methods in this family operate within the differentiable training loop of 3D Gaussian Splatting. They share the following core mechanisms:

1.  **Trigger Condition:** Densification is not performed at every optimization step. It is triggered periodically (e.g., every `refineEvery` steps) after a warm-up period and before a stopping threshold (e.g., `stopSplitAt`) to allow the optimizer to stabilize [Parent][S5].
2.  **Metric Computation:** A score is computed for each Gaussian $\mathcal{G}_i$ to determine its contribution to the reconstruction error. Common metrics include:
    *   **Gradient Norm:** The magnitude of the gradient of the loss with respect to the Gaussian's 2D mean position ($\nabla_{\mu_{2D}}$) [Parent][S5].
    *   **Screen Size:** The projected area of the Gaussian in pixel space. Gaussians that are too small may not capture detail, while those that are too large may oversmooth [Parent].
    *   **Opacity:** The current opacity $\alpha_i$ of the Gaussian. Low opacity often indicates redundancy or "floaters" [Parent].
    *   **Saliency/Loss Contribution:** Aggregated loss or gradient magnitude over the pixels covered by the Gaussian [S1, unofficial].
3.  **Action (Split/Clone/Prune):**
    *   **Split:** A Gaussian is replaced by two new Gaussians. The new means are derived from the original mean plus/minus a perturbation along the principal axes of the covariance matrix. The new scales are typically reduced by a factor (e.g., 1.6) [Parent][S1, unofficial].
    *   **Clone:** A copy of the Gaussian is created with a small random perturbation to its mean to fill gaps without reducing scale [Parent][S1, unofficial].
    *   **Prune:** Gaussians failing specific criteria (e.g., low opacity, low significance) are removed from the active set [Parent].
4.  **Opacity Management:** To prevent over-densification and stabilize training, opacities of newly created Gaussians are initialized to a low value (e.g., logit -1.386) [Parent][S1, unofficial]. Some methods use opacity splitting formulas to maintain transmittance consistency during cloning [S5].
5.  **Data Structure Management:** The set of Gaussians is stored in GPU buffers. Densification requires managing buffer capacity (often pre-allocated to be at least 3x the initial count) and performing compaction to remove pruned primitives [Parent][S4].

## Subtopics

### [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)
The Hybrid Coarse-to-Fine Strategy is a training methodology for point-based radiance field representations that employs multiple stages of refinement rather than a single-stage optimization [7173-hybrid-coarse-to-fine-strategy]. It addresses the problem of insufficient geometric detail and poor reconstruction quality in sparse initial point clouds by progressively densifying the representation based on error metrics [7173-hybrid-coarse-to-fine-strategy]. Unlike single-stage training, this approach yields significantly higher rendering quality by balancing global structure capture with local detail refinement [7173-hybrid-coarse-to-fine-strategy]. It differs from siblings by using a mesh-based or point-based representation with Delaunay triangulation and scoring based on DSSIM and pixel coverage [7173-hybrid-coarse-to-fine-strategy].

### [Training Objective and Densification](7448-training-objective-and-densification.md)
This method defines the specific loss functions and densification strategies for **MeshGS** (Adaptive Mesh-Aligned Gaussian Splatting), a variant of 3D Gaussian Splatting that integrates explicit mesh geometry into the training loop [7448-training-objective-and-densification]. It solves the problem of Gaussian primitives drifting away from or failing to align with the underlying surface geometry during optimization [7448-training-objective-and-densification]. It differs from standard 3DGS [Parent] by introducing geometry-aware losses (normal consistency, projection, and scale regularization) for "tightly-bound" splats and by initializing new primitives via mesh surface sampling rather than random perturbation [7448-training-objective-and-densification].

### [Fine-tuning](9138-fine-tuning.md)
Fine-tuning is a post-initialization optimization phase in 3D Gaussian Splatting where a pre-generated set of Gaussian primitives is refined to achieve photorealistic reconstruction. It solves the problem of initializing with redundant or low-fidelity primitives by enabling the growing and pruning of 3D primitives during the optimization process [9138-fine-tuning]. It sets itself apart from pure initialization methods by actively managing the Gaussian population size and density through densification, leveraging pretrained priors to accelerate convergence and reduce memory consumption compared to training from scratch [9138-fine-tuning].

### Progressive Propagation Strategy
- [S1, unofficial] Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage.

### Interleaved Densification
- [S2] Alternate between spatial and temporal densification every 2 epochs to avoid Gaussians failing to reconstruct either motion or appearance detail.

## Background
Progressive training strategies emerged as researchers identified that single-stage optimization of 3D Gaussian Splatting often leads to suboptimal convergence, particularly in complex scenes or when starting from sparse initializations. The Hybrid Coarse-to-Fine approach [7173-hybrid-coarse-to-fine-strategy] was developed to improve geometric detail by iteratively refining mesh-based representations. MeshGS [7448-training-objective-and-densification] integrated these concepts with explicit mesh alignment to handle geometry-aware scenes. Fine-tuning approaches [9138-fine-tuning] leveraged feed-forward initialization to reduce training time and memory usage. Progressive propagation [S1, unofficial] and interleaved densification [S2] further refined these strategies for specific domains like video and dynamic scenes.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
