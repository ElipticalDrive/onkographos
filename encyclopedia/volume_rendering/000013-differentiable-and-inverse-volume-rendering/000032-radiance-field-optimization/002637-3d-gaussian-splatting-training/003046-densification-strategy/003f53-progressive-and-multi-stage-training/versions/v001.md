# Progressive and Multi-stage Training

Also called: Progressive Propagation Strategy

## Overview
Progressive and Multi-stage Training addresses the instability and suboptimal convergence of single-stage densification in 3D Gaussian Splatting by structuring the optimization into distinct phases or propagating the Gaussian cloud gradually [S1, unofficial]. It sets itself apart from standard single-stage training by decoupling coarse initialization from fine-grained refinement, thereby improving rendering quality and coverage stability [S1, unofficial].

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
A training strategy involving multiple stages of refinement for the point cloud, as opposed to training the point cloud in a single stage. It yields better rendering quality than single-stage training by progressively refining the geometric representation. See [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md).

### [Fine-tuning](9138-fine-tuning.md)
Fine-tuning in 3D Gaussian Splatting (3DGS) is a post-initialization optimization phase that refines pre-generated Gaussian primitives to achieve photorealism while reducing memory consumption. It sets itself apart from end-to-end training by leveraging pretrained priors to accelerate convergence and by explicitly enabling the growing and pruning of primitives during the optimization process to eliminate redundancy. See [Fine-tuning](9138-fine-tuning.md).

### Progressive Propagation Strategy
Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage [S1, unofficial].

## Background
The concept of progressive expansion in Gaussian Splatting was introduced in GaussianPro [77], which proposed expanding the Gaussian cloud in a fronto-parallel manner to ensure smooth expansion and better coverage [S1, unofficial]. This approach addresses the limitations of static initialization by allowing the density to adapt gradually to the scene complexity.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
