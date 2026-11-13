# Adaptive View Selection for Inpainting

Also called: 3D Inpainting Component, 3D Inpainting Component with Adaptive View Selection, Adaptive Weight Inpainting, Gradient-Based View Selection

## Overview
This method is a component for 3D scene de-occlusion and inpainting that recovers missing geometry and appearance in occluded regions of a Neural Radiance Field (NeRF) [S1]. It addresses the inconsistency of single-view 2D inpainting by introducing a training view selection mechanism based on regional average gradients [S1]. Unlike siblings that may use uniform masking or semantic priors alone, this method adaptively switches between relying on a 2D inpainter's appearance prior and leveraging multi-view geometric consistency based on texture richness [S1].

## How it works
The method operates within a NeRF training loop where specific rays are identified as occluded. It uses a 2D inpainter (specifically LaMa [S1]) to generate initial appearance priors and a region coding network to identify occluded areas [S1]. The core innovation is the adaptive weighting of the reconstruction loss based on whether the rendered NeRF output has sufficient texture detail (gradient) compared to the 2D inpainter's output [S1].

### Inputs and Outputs
**Inputs:**
*   NeRF model parameters $\Theta$ (defining $F_\Theta(\mathbf{x}, \mathbf{v}) \to (\sigma, \mathbf{c})$) [S1].
*   Input views with ground truth colors $C(r)$ and depths $D(r)$ [S1].
*   A binary mask $M$ indicating occluded regions in the input views [S1].
*   A 2D inpainter (e.g., LaMa) to generate inpainted images $I_l$ for occluded regions [S1].

**Outputs:**
*   Updated NeRF parameters $\Theta$ that reconstruct the scene with inpainted occluded regions [S1].
*   Rendered images with consistent appearance in previously occluded areas [S1].

### Data Structures
*   **Ray Set $R$**: The set of all rays sampled during training [S1].
*   **Predicted Mask $\hat{P}(r)$**: A scalar value for each ray $r$, indicating the probability or confidence that the ray passes through an occluded region. This is derived from the object segmentation network [S1].
*   **Sub-regions**: The occluded area in a view is divided into multiple sub-regions for gradient evaluation [S1].
*   **Color Priors $C_l(r)$**: The color value from the 2D inpainted image corresponding to ray $r$ [S1].

### Numbered Steps
1.  **Occlusion Identification**: Use an object segmentation network to infer masks $M = \{M_s | s = 1...K\}$ for input views, identifying occluded areas [S1].
2.  **2D Inpainting**: Apply a 2D inpainter (LaMa) to the occluded regions of the input views to generate appearance priors $C_l(r)$ [S1].
3.  **Sub-region Division**: Divide the occluded area in each view into multiple sub-regions [S1].
4.  **Gradient Evaluation**: Calculate the average gradient for each sub-region in both the rendered NeRF output ($\nabla \hat{g}$) and the 2D inpainted result ($\nabla g$) [S1].
5.  **Adaptive Weight Calculation**: Determine the adaptive weight $\lambda$ for the reconstruction loss based on the gradient comparison (see Formula) [S1].
6.  **Loss Computation**: Compute the total reconstruction loss $L$ using the adaptive weight $\lambda$, the non-occluded loss $L_{NonOcc}$, and the occluded loss $L_{Occ}$ [S1].
7.  **Optimization**: Update NeRF parameters $\Theta$ by minimizing $L$ along with other consistency losses (e.g., depth loss, region encoding loss) [S1].

### Formulas
The total reconstruction loss $L$ is a weighted sum of the non-occluded loss and the occluded loss [S1]:

$$
L = \lambda L_{NonOcc} + (1 - \lambda) L_{Occ}
$$

The adaptive weight $\lambda$ is determined by comparing the average gradient of the rendered result $\nabla \hat{g}$ to the average gradient of the 2D inpainting result $\nabla g$ in a sub-region [S1]:

$$
\lambda = 
\begin{cases} 
1 & \text{if } \nabla \hat{g} > \tau \cdot \nabla g \\
0 & \text{if } \nabla \hat{g} \le \tau \cdot \nabla g 
\end{cases}
$$

Where $\tau$ is a threshold parameter [S1].

The reconstruction loss for non-occluded areas $L_{NonOcc}$ is defined as [S1]:

$$
L_{NonOcc} = \sum_{r \in R} (1 - \hat{P}(r)) \left[ ||\hat{C}_c(r) - C(r)||_2^2 + ||\hat{C}_f(r) - C(r)||_2^2 \right]
$$

Where:
*   $\hat{P}(r)$ is the predicted mask value for ray $r$ [S1].
*   $\hat{C}_c(r)$ and $\hat{C}_f(r)$ are the predicted colors from the coarse and fine NeRF networks, respectively [S1].
*   $C(r)$ is the ground truth color for ray $r$ [S1].

The reconstruction loss for occluded areas $L_{Occ}$ is defined as [S1]:

$$
L_{Occ} = \sum_{r \in R} \hat{P}(r) \left[ ||\hat{C}_c(r) - C_l(r)||_2^2 + ||\hat{C}_f(r) - C_l(r)||_2^2 \right]
$$

Where:
*   $C_l(r)$ is the color of the corresponding pixel in the 2D inpainting image for ray $r$ [S1].

### Parameters
*   $\tau$: Threshold for gradient comparison. Not given in the sources.
*   $\lambda_1, \lambda_2, \lambda_3$: Weights for the joint optimization strategy involving region encoding, RGB, and depth losses. Not given in the sources.

### Edge Cases and Pitfalls
*   **Uniform Strategy Misjudgment**: Applying a uniform treatment strategy to the entire occluded area can lead to misjudgment. The method avoids this by dividing the occluded area into sub-regions and evaluating gradients individually [S1].
*   **2D Inpainter Inconsistency**: Single 2D image inpainting does not consider consistency in the real 3D scene. The method addresses this by selecting training views based on texture richness (gradient) to ensure multi-view consistency [S1].
*   **Lack of Supplementary Information**: If no other views can provide supplementary information for an occluded area, the method relies primarily on the 2D inpainting results [S1].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Strengths**: Effectively reduces the disturbing effect of 2D inpainting on 3D editing by leveraging multi-view correlation [S1]. It precisely identifies target objects in different views and accomplishes realistic inpainting [S1].
*   **Limitations**: Relies on the quality of the 2D inpainter (LaMa) and the accuracy of the object segmentation network [S1]. Performance depends on the availability of multi-view data [S1].
*   **When to use**: When inpainting occluded regions in NeRFs and multi-view consistency is critical [S1].
*   **When not to use**: When single-view inpainting is sufficient or when multi-view data is not available [S1].

## Background
The method was proposed in the paper "3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting" [S1]. It builds upon the NeRF framework [S1] and addresses the limitations of 2D inpainting in 3D scenes by introducing a training view selection mechanism based on regional average gradients [S1]. The method has been evaluated on datasets such as LLFF and SPIn-NeRF [S1].

## Sources
[S1] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
