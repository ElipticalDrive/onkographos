# Densification Strategy
Also called: Gaussian Densification Strategies, 3DGS Densification

## Overview
Densification strategies manage the population of Gaussian primitives during the training of 3D Gaussian Splatting (3DGS) models, addressing the issues of unpredictable model size, redundant primitives, and excessive training duration [S1]. Unlike the original 3DGS approach which relies on heuristic splitting and cloning based on position gradients and opacity, this family includes methods that use score-based constructive growth, temporal sensitivity pruning, and groupwise motion distillation to control model complexity and improve rendering efficiency [S1][S3, unofficial].

## Shared principles
The core mechanism of densification in 3D Gaussian Splatting involves dynamically adjusting the number and distribution of Gaussian primitives $\mathcal{G}_i$ during the optimization process. The primary goal is to allocate more primitives in regions of high geometric or photometric complexity while removing or merging those that contribute little to the reconstruction [S1].

Common principles include:
1.  **Evaluation Metric**: A score or metric is computed for each Gaussian to determine its contribution to the reconstruction quality. This metric often combines gradient information, saliency, and primitive properties [S1].
2.  **Population Control**: Strategies either add new Gaussians (densification), remove existing ones (pruning), or both, to reach a target model size or budget [S1][S3, unofficial].
3.  **Temporal Aggregation**: In dynamic scenes, metrics may be aggregated over multiple timesteps and poses to assess the stability and importance of a Gaussian over time [S3, unofficial].

## Subtopics

### [Gaussian Densification](4314-gaussian-densification.md)
[Link to [Gaussian Densification](4314-gaussian-densification.md)]

### [Optimization and Densification Strategy](4777-optimization-and-densification-strategy.md)
[Link to [Optimization and Densification Strategy](4777-optimization-and-densification-strategy.md)]

### [GroupFlow](6351-groupflow.md)
[Link to [GroupFlow](6351-groupflow.md)]

### Score-based Densification
**Definition**
A purely constructive densification process that steers the addition of new Gaussians toward those that raise reconstruction quality, ensuring model size increases in a controlled manner towards an exact user-defined budget without requiring pruning or culling during training [S1].

**Steps**
1.  Sample training views and compute per-pixel saliency to identify important regions [S1].
2.  Compute a per-Gaussian score $S_g$ by combining gradient, saliency, and primitive properties into a scoring function $F$ [S1].
3.  Densify new Gaussians in the vicinity of the top-scoring Gaussians according to a pre-defined budget [S1].
4.  Follow a growth curve that mimics 3DGS’ behavior and can be fitted to yield any desired model size after training [S1].

**Comparison**
Unlike previous work that requires substantial pruning or culling of Gaussians during training, this method uses a purely constructive schedule, avoiding unnecessary peaks in optimization that could violate hardware or budget constraints [S1].

### Pixel-GS
**Definition**
Scales the densification gradient by the projected area in pixels, so a Gaussian that hurts many pixels is split first [S2, unofficial].

### Temporal Sensitivity Pruning
**Definition**
TSP is a pruning method that removes low-impact Gaussians by calculating a temporal sensitivity score approximating the second-order sensitivity to the L2 loss across all training views [S3, unofficial].

**Steps**
1.  For each Gaussian $\mathcal{G}_i$, calculate the gradient of the rendered image $I_{\mathcal{G}_t}(\phi)$ with respect to the projected Gaussian value $g_i$ for all training poses $\phi$ and timesteps $t$ in $\mathcal{P}_{gt}$ [S3, unofficial].
2.  Square these gradients and sum them over all poses and timesteps to compute the cumulative sensitivity score $\tilde{U}_{\mathcal{G}_i}$ [S3, unofficial].
3.  Periodically prune Gaussians with low sensitivity scores during training to reduce neural inference load [S3, unofficial].

**Formula**
$$
\tilde{U}_{\mathcal{G}_i} \approx \nabla_{g_i}^2 L_2 \approx \sum_{\phi, t \in \mathcal{P}_{gt}} \left( \nabla_{g_i} I_{\mathcal{G}_t}(\phi) \right)^2
$$
where $\tilde{U}_{\mathcal{G}_i}$ is the TSP score for Gaussian $\mathcal{G}_i$, $g_i$ is the value of $\mathcal{G}_i$ projected onto image space, $\mathcal{P}_{gt}$ is the set of training poses and timesteps, and $I_{\mathcal{G}_t}(\phi)$ is the rendered image at pose $\phi$ and timestep $t$ [S3, unofficial].

**Parameters**
$\mathcal{P}_{gt}$: The set of training poses and timesteps used for sensitivity aggregation [S3, unofficial].

## Background
3D Gaussian Splatting (3DGS) was introduced by Kerbl et al. in 2023 [S1]. The original method used a heuristic densification process involving splitting and cloning Gaussians based on position gradients and opacity thresholds. However, this approach led to unpredictable model sizes and training durations, making it difficult to control for specific hardware or budget constraints [S1].

Subsequent works have proposed various densification strategies to address these limitations. "Taming 3DGS" introduced a score-based constructive densification method to control model size [S1]. "SpeeDe3DGS" introduced Temporal Sensitivity Pruning (TSP) for dynamic scenes to remove redundant primitives and improve rendering speed [S3, unofficial]. Other variants like Pixel-GS and GaussianPro have also been proposed to improve densification efficiency and quality [S2, unofficial].

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
