# Adaptive View Selection for Inpainting

Also called: 3D Inpainting Component, 3D Inpainting Component with Adaptive View Selection, Adaptive Weight Inpainting, Gradient-Based View Selection

## Overview
This method addresses the inconsistency of 2D inpainting priors in 3D scenes by introducing a training view selection mechanism that leverages multi-view correlation [S1]. It distinguishes itself from standard NeRF editing by dynamically weighting the reconstruction loss between ground-truth visible pixels and 2D inpainted priors based on the texture richness (average gradient) of occluded sub-regions [S1].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   A pre-trained NeRF model $f_\theta$ mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ [S1].
    *   A set of input views with known camera poses [S1].
    *   A binary mask $M$ identifying occluded regions in the input views [S1].
    *   A 2D inpainter (specifically LaMa) to generate appearance priors for occluded areas [S1].
    *   A threshold parameter $\tau$ for gradient comparison [S1].
*   **Outputs:**
    *   Updated NeRF weights $\theta$ that reconstruct the occluded areas consistently across views [S1].

### Data Structures
*   **NeRF Model:** Standard MLP taking 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$, outputting density $\sigma$ and color $\mathbf{c}$ [S1].
*   **Predicted Mask $\hat{P}(r)$:** A soft mask value for ray $r$, output by an object segmentation network (or provided externally) indicating the probability of a ray passing through an occluded area [S1].
*   **2D Inpainting Prior $C_l(r)$:** The color value of the pixel corresponding to ray $r$ in the image inpainted by the 2D model (LaMa) [S1].
*   **Sub-regions:** The occluded area in each view is divided into multiple sub-regions (patches) to evaluate local texture richness [S1].

### Numbered Steps

1.  **Initial Recovery:** Use a pretrained 2D image inpainter (e.g., LaMa) to generate initial color priors $C_l$ for the occluded areas in the input views [S1].
2.  **Sub-region Division:** Divide the occluded area in each view into multiple sub-regions (patches) [S1].
3.  **Gradient Evaluation:** For each sub-region, calculate the average gradient of the rendered result $\nabla \hat{g}$ and the average gradient of the 2D inpainting result $\nabla g$ [S1].
4.  **Adaptive Weight Calculation:** Determine the adaptive weight $\lambda$ for the reconstruction loss based on the gradient comparison [S1]:
    *   If $\nabla \hat{g} > \tau \cdot \nabla g$, set $\lambda = 1$. This indicates the rendered result has richer texture than the 2D prior, so the model should rely on multi-view consistency (ground truth from other views).
    *   If $\nabla \hat{g} \le \tau \cdot \nabla g$, set $\lambda = 0$. This indicates the rendered result lacks texture, so the model should rely on the 2D inpainting prior [S1].
5.  **Loss Formulation:** Compute the total reconstruction loss $L$ using the adaptive weight $\lambda$ [S1]:
    $$ L = \lambda L_{NonOcc} + (1 - \lambda) L_{Occ} $$
    where $L_{NonOcc}$ is the loss for non-occluded areas and $L_{Occ}$ is the loss for occluded areas [S1].
6.  **Optimization:** Update the NeRF parameters $\theta$ using gradient descent (e.g., Adam) to minimize $L$ [S1].

### Formulas

**Adaptive Weight $\lambda$ [S1]:**
$$
\lambda =
\begin{cases}
1 & \text{if } \nabla \hat{g} > \tau \cdot \nabla g \\
0 & \text{if } \nabla \hat{g} \le \tau \cdot \nabla g
\end{cases}
$$
Where:
*   $\nabla \hat{g}$ is the average gradient of the rendered result in a sub-region.
*   $\nabla g$ is the average gradient of the 2D inpainting results in the same sub-region.
*   $\tau$ is a threshold parameter.

**Reconstruction Loss for Non-Occluded Areas $L_{NonOcc}$ [S1]:**
$$
L_{NonOcc} = \sum_{r \in R} (1 - \hat{P}(r)) \left[ \| \hat{C}_c(r) - C(r) \|_2^2 + \| \hat{C}_f(r) - C(r) \|_2^2 \right]
$$
Where:
*   $R$ is the set of rays.
*   $\hat{P}(r)$ is the predicted mask value for ray $r$.
*   $\hat{C}_c(r)$ and $\hat{C}_f(r)$ are the predicted colors from the coarse and fine NeRF networks, respectively.
*   $C(r)$ is the ground truth color of the input view for ray $r$.

**Reconstruction Loss for Occluded Areas $L_{Occ}$ [S1]:**
$$
L_{Occ} = \sum_{r \in R} \hat{P}(r) \left[ \| \hat{C}_c(r) - C_l(r) \|_2^2 + \| \hat{C}_f(r) - C_l(r) \|_2^2 \right]
$$
Where:
*   $C_l(r)$ is the color of the corresponding pixel in the 2D inpainting image for ray $r$, serving as an estimated color prior [S1].

### Parameters
*   $\tau$: Threshold for gradient comparison. Not given in the sources.
*   $\lambda_1, \lambda_2, \lambda_3$: Weights for the object segmentation network loss ($L_{region-encoding}, L_{rgb}, L_{depth}$). Not given in the sources.

### Edge Cases and Pitfalls
*   **Single 2D Inpainter Limitation:** A single 2D image inpainter does not consider consistency in the real 3D scene [S1]. The method addresses this by selecting training views based on texture richness (gradient) to ensure multi-view consistency [S1].
*   **Uniform Strategy Misjudgment:** Applying a uniform treatment strategy to the entire occluded area can lead to misjudgment [S1]. The method avoids this by dividing the occluded area into sub-regions and evaluating gradients individually [S1].
*   **Lack of Supplementary Information:** When no other views can provide supplementary information for an occluded area, the method relies primarily on the 2D inpainting results [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Effectively reduces the disturbance of 2D inpainters in 3D editing by leveraging appearance consistency across views [S1]. It identifies target objects precisely in different views and accomplishes realistic inpainting [S1].
*   **Limitations:** Relies on the quality of the 2D inpainter (LaMa) for priors [S1]. Requires multi-view data to be effective [S1].
*   **When to use:** When inpainting occluded areas in NeRFs and multi-view consistency is critical [S1].
*   **When not to use:** When only a single view is available or when the 2D inpainter fails to provide reasonable priors [S1].

## Background
*   **Origin:** The method was proposed in "3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting" [S1].
*   **History:** It builds upon NeRF [S1] and 2D inpainting techniques like LaMa [S1]. It specifically addresses the limitation of 2D inpainting in 3D scenes by introducing a training view selection mechanism based on regional average gradients [S1].
*   **Usage:** Evaluated on datasets including NeRF LLFF and SPIn-NeRF [S1].

## Sources
[S1] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
