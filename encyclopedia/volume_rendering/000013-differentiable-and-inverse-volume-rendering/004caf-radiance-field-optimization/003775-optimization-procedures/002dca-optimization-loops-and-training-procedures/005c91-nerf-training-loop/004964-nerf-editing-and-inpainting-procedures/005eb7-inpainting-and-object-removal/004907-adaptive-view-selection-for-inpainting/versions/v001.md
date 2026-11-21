# Adaptive View Selection for Inpainting

Also called: 3D Inpainting Component, 3D Inpainting Component with Adaptive View Selection, Adaptive Weight Inpainting, Gradient-Based View Selection

## Overview
This method addresses the inconsistency of single-view 2D inpainting in 3D scenes by integrating a 2D appearance prior with multi-view geometric consistency. It differs from standard NeRF training [NeRF Training Loop](970-nerf-training-loop.md) by dynamically switching the supervision signal for occluded regions between a 2D inpainter (LaMa) and other training views based on local texture richness. It builds on [NeRF Training Loop](970-nerf-training-loop.md) and [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md) by adding a gradient-based view selection mechanism to resolve conflicts between 2D priors and 3D geometry.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input images with known camera poses; a binary mask $M$ identifying occluded regions in the target view; a pre-trained 2D image inpainter (LaMa [25]) to generate color priors $C_l$ for occluded areas; a NeRF model (coarse $\hat{C}_c$ and fine $\hat{C}_f$ branches) and a region coding network (segmentation network) predicting occlusion masks $\hat{P}(r)$.
*   **Outputs:** A refined NeRF model and inpainted images for the target view with consistent multi-view geometry.

**Data Structures:**
*   **Ray Set $R$:** Set of rays cast through the target view.
*   **Mask $\hat{P}(r)$:** Predicted probability that ray $r$ passes through an occluded region (output of the region coding network).
*   **Color Priors $C_l(r)$:** Pixel colors from the 2D inpainted image corresponding to ray $r$.
*   **Sub-regions:** The occluded area is divided into multiple sub-regions for gradient evaluation.

**Numbered Steps:**
1.  **2D Inpainting Prior Generation:** Apply the pre-trained 2D inpainter (LaMa) to the target view to generate an inpainted image. Extract the color $C_l(r)$ for rays $r$ passing through occluded regions.
2.  **Region Coding and Masking:** Use the region coding network to predict the occlusion mask $\hat{P}(r)$ for rays in the target view. This network is trained using multi-view consistency constraints (region encoding, RGB, and depth losses) to ensure accurate segmentation across views.
3.  **Sub-region Division:** Divide the occluded area in the target view into multiple sub-regions.
4.  **Gradient Evaluation:** Calculate the average gradient $\nabla \hat{g}$ for the rendered result in each sub-region and the average gradient $\nabla g$ for the 2D inpainting result in the corresponding sub-region.
5.  **Adaptive Weight Calculation:** Determine the adaptive weight $\lambda$ for the reconstruction loss based on the ratio of gradients. If $\nabla \hat{g} > \tau \cdot \nabla g$, set $\lambda = 1$ (prefer NeRF rendering); otherwise, set $\lambda = 0$ (prefer 2D inpainting prior).
6.  **Loss Computation:** Compute the total reconstruction loss $L$ using the adaptive weight $\lambda$ and the specific losses for non-occluded and occluded areas.
7.  **Optimization:** Update the NeRF parameters (and region coding network) by minimizing $L$.

**Formulas:**
The total reconstruction loss $L$ is defined as [S1]:
$$
L = \lambda L_{NonOcc} + (1 - \lambda) L_{Occ}
$$
where $\lambda$ is the adaptive weight [S1]:
$$
\lambda = 
\begin{cases} 
1 & \text{if } \nabla \hat{g} > \tau \cdot \nabla g \\
0 & \text{if } \nabla \hat{g} \le \tau \cdot \nabla g 
\end{cases}
$$
Here, $\nabla \hat{g}$ is the average gradient of the rendered result in a sub-region, $\nabla g$ is the average gradient of the 2D inpainting results in the same sub-region, and $\tau$ is a threshold parameter.

The reconstruction loss for non-occluded areas $L_{NonOcc}$ is [S1]:
$$
L_{NonOcc} = \sum_{r \in R} (1 - \hat{P}(r)) \left[ \|\hat{C}_c(r) - C(r)\|_2^2 + \|\hat{C}_f(r) - C(r)\|_2^2 \right]
$$
where $\hat{C}_c(r)$ and $\hat{C}_f(r)$ are the coarse and fine predicted colors, $C(r)$ is the ground truth color, and $\hat{P}(r)$ is the predicted occlusion mask.

The reconstruction loss for occluded areas $L_{Occ}$ is [S1]:
$$
L_{Occ} = \sum_{r \in R} \hat{P}(r) \left[ \|\hat{C}_c(r) - C_l(r)\|_2^2 + \|\hat{C}_f(r) - C_l(r)\|_2^2 \right]
$$
where $C_l(r)$ is the color of the corresponding pixel in the 2D inpainting image.

**Parameters:**
*   $\tau$: Threshold for gradient comparison. Not given in the sources.
*   $\lambda_1, \lambda_2, \lambda_3$: Weights for the region coding network losses ($L_{region-encoding}, L_{rgb}, L_{depth}$). Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Uniform Strategy Misjudgment:** Applying a uniform strategy to the entire occluded area may fail to recover details. The method avoids this by dividing the occluded area into sub-regions and evaluating gradients individually [S1].
*   **2D Inpainter Inconsistency:** Single 2D inpainting does not consider 3D consistency. The method mitigates this by selecting views based on texture richness (gradient) to ensure multi-view consistency [S1].
*   **Lack of Supplementary Information:** If no other view provides supplementary information for an occluded area, the method relies primarily on the 2D inpainting results [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the NeRF training loop and the additional computation of 2D inpainting and gradient evaluation. Not given in the sources.
*   **Memory Complexity:** Requires storing the NeRF model, region coding network, and 2D inpainter outputs. Not given in the sources.
*   **Strengths:** Improves fidelity of de-occluded results by leveraging multi-view correlation and avoiding the disturbing effects of 2D inpainting in 3D editing [S1].
*   **Limitations:** Relies on the quality of the pre-trained 2D inpainter and the accuracy of the region coding network. Not given in the sources.
*   **When to use:** When inpainting occluded areas in NeRF scenes where multi-view consistency is critical and single-view 2D inpainting leads to inconsistencies.
*   **When not to use:** When no multi-view data is available or when the occluded areas are not well-segmented.

## Background
The method was introduced in "3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting" [S1]. It addresses the limitation of single 2D image inpainting in 3D scenes by introducing a training view selection mechanism based on regional average gradients. The method has been evaluated on datasets like LLFF and SPIn-NeRF [S1].

## Sources
[S1] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
