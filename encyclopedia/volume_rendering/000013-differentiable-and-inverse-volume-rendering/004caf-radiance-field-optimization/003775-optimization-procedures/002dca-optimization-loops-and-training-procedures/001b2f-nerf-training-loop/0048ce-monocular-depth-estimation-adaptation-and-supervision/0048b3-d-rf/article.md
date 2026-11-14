# DäRF
Also called: Monocular Depth Adaptation for boosting Radiance Fields from Sparse Input Views

## Overview
DäRF is a framework for robust few-shot NeRF reconstruction that combines Neural Radiance Fields (NeRF) with Monocular Depth Estimation (MDE) through online complementary training [S1, unofficial]. It solves the problem of poor generalization in sparse-view NeRF by injecting MDE's geometric prior into the NeRF representation at both seen and unseen viewpoints, while simultaneously adapting the MDE network to produce multiview-consistent geometry aligned with the NeRF [S1, unofficial]. Unlike methods that only distill depth from pretrained MDE on seen views, DäRF jointly optimizes both networks to remove MDE ambiguity via patch-wise scale-shift fitting and confidence modeling [S1, unofficial].

## How it works
**Inputs:** A set of $N$ input images $S = \{I_i\}$ with known camera poses, and a pretrained MDE network $M_{\phi}$ [S1, unofficial].
**Outputs:** A NeRF representation $F_{\theta}$ and an adapted MDE network $M_{\phi'}$ [S1, unofficial].

**Data Structures:**
*   $F_{\theta}$: MLP mapping $(x, d) \to (\sigma, c)$ where $x \in \mathbb{R}^3$ is position, $d \in \mathbb{R}^2$ is viewing direction, $\sigma$ is density, $c$ is color [S1, unofficial].
*   $M_{\phi}$: MDE network outputting relative depth map $D^*$ [S1, unofficial].
*   $\bar{D}$: Rendered absolute depth from NeRF [S1, unofficial].

**Steps:**
1.  **Ray Sampling:** For each pixel in input images, sample points along rays using NeRF's sampling strategy [S1, unofficial].
2.  **Depth Rendering:** Render absolute depth $\bar{D}$ from NeRF by accumulating sampled depths weighted by transmittance and density [S1, unofficial].
3.  **Monocular Depth Estimation:** Pass input images through $M_{\phi}$ to obtain relative depth maps $D^*$ [S1, unofficial].
4.  **Patch-wise Scale-Shift Fitting:** Divide images into patches $P$. For each patch, compute local scale $w$ and shift $q$ by minimizing the least squares error between aligned MDE depth and NeRF depth:
    $$ \min_{w, q} \sum_{p \in P} \| (w \cdot D^*(p) + q) - \bar{D}(p) \|^2 $$
    This is solved analytically using normal equations [S1, unofficial]. Global fitting is avoided because MDE performs poorly on depth differences between multiple objects [S1, unofficial].
5.  **Confidence Modeling:** Verify accurate depth information to handle ambiguity (e.g., relative depths not being multiview-consistent or convexity problems) [S1, unofficial]. [G] Typically implemented by weighting the depth loss based on the variance of depth estimates or consistency across views, though specific confidence metric formula is not detailed in the provided text.
6.  **Joint Optimization:** Update NeRF parameters $\theta$ and MDE parameters $\phi$ jointly. The MDE is fine-tuned to align with the canonical geometry captured by NeRF, directly removing ambiguity present in the pretrained MDE [S1, unofficial].
7.  **Loss Computation:** Combine photometric loss with depth consistency loss derived from the patch-wise aligned depths [S1, unofficial].

**Parameters:**
*   Patch size: Not given in the sources.
*   Learning rates for $\theta$ and $\phi$: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **MDE Ambiguity:** MDE outputs relative depth lacking absolute scale/shift, causing artifacts if not properly aligned [S1, unofficial].
*   **Convexity Problems:** MDE may produce incorrect depth for convex surfaces; confidence modeling helps mitigate this [S1, unofficial].
*   **Multiview Inconsistency:** MDE depths may not be consistent across views; joint optimization enforces consistency [S1, unofficial].

## Subtopics

### Patch-wise Scale-Shift Fitting
A local alignment method that computes scale and shift per image patch to handle depth differences between objects better than global fitting. Choose when MDE has poor performance on depth differences between multiple objects. See [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md).

### Confidence Modeling
A technique to verify accurate depth information and handle ambiguity such as multiview inconsistency and convexity problems. Choose when MDE outputs are unreliable or ambiguous. See [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md).

### Joint Optimization
Simultaneously updating NeRF and MDE parameters to enforce multiview consistency and adapt MDE to the scene. Choose when aiming for robust few-shot reconstruction with sparse inputs. See [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md).

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Achieves state-of-the-art results quantitatively and qualitatively on real-world indoor and outdoor scene datasets for few-shot NeRF optimization [S1, unofficial].
*   **Strengths:** Robust geometry reconstruction and artifact removal in both unseen and seen viewpoints; removes MDE ambiguity by finetuning with NeRF geometry; exploits MDE at both seen and unseen views [S1, unofficial].
*   **Limitations:** Relies on the quality of the pretrained MDE network; joint optimization may be computationally intensive [S1, unofficial].
*   **When to Use:** Few-shot NeRF reconstruction with sparse input views where standard NeRF fails due to ambiguity or lack of geometry priors [S1, unofficial].
*   **When Not to Use:** When abundant multi-view images are available, as standard NeRF may suffice; when computational resources for joint optimization are limited [S1, unofficial].

## Background
DäRF was proposed in 2023 by authors of the paper "DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation" [S1, unofficial]. It addresses limitations of previous methods like SCADE [50], which only distill depths from pretrained MDE on seen views and use probabilistic modeling for uncertainty. DäRF improves upon this by jointly optimizing NeRF and MDE, distilling monocular depth prior to NeRF at both seen and unseen views, and directly removing MDE ambiguity through finetuning with canonical geometry captured by NeRF [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
