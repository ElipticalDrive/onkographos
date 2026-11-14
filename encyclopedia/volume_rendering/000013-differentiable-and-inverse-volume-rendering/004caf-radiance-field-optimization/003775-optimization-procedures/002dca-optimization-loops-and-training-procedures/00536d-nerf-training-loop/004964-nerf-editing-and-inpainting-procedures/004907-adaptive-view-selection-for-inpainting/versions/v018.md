# Adaptive View Selection for Inpainting

Also called: 3D Inpainting Component, 3D Inpainting Component with Adaptive View Selection, Adaptive Weight Inpainting, Gradient-Based View Selection

## Overview
This method addresses the inconsistency of single-view 2D inpainting in 3D scenes by introducing a training view selection mechanism based on regional average gradients [S1]. It differs from standard NeRF editing by dynamically weighting the reconstruction loss between ground-truth pixels and 2D inpainting priors based on texture richness, ensuring multi-view consistency for occluded regions [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained NeRF $f_\theta$, a set of input views with known camera poses, a binary mask $M$ identifying the occluded object in at least one view, and a pretrained 2D image inpainter (specifically LaMa [S1]).
*   **Outputs:** An optimized NeRF where the occluded object is removed and the background is inpainted consistently across all views.

**Data Structures:**
*   **NeRF Representation:** Standard MLP mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$ [S1].
*   **Mask:** Binary mask $M$ or predicted mask $\hat{P}(r)$ indicating occlusion status for ray $r$ [S1].
*   **Sub-regions:** The occluded area in each view is divided into multiple sub-regions (patches) for gradient evaluation [S1].

**Numbered Steps:**

1.  **Object Segmentation and Mask Generation:**
    Use an object segmentation network to infer masks $M = \{M_s | s = 1...K\}$ for all input views, constrained by depth information, to identify the occluded object in each view [S1].

2.  **2D Inpainting Prior Generation:**
    Apply the pretrained 2D inpainter (LaMa) to the input views containing the occluded object to generate appearance priors $C_l$ for the occluded regions [S1].

3.  **Sub-region Division and Gradient Evaluation:**
    Divide the occluded area in each view into multiple sub-regions. For each sub-region, calculate the average gradient of the rendered result $\nabla \hat{g}$ and the average gradient of the 2D inpainting result $\nabla g$ [S1].

4.  **Adaptive Weight Calculation:**
    Compute the adaptive weight $\lambda$ for each sub-region based on the texture richness (gradient magnitude) [S1]:
    $$
    \lambda = \begin{cases} 
    1 & \text{if } \nabla \hat{g} > \tau \cdot \nabla g \\
    0 & \text{if } \nabla \hat{g} \le \tau \cdot \nabla g 
    \end{cases}
    $$
    where $\tau$ is a threshold parameter [S1]. If $\nabla \hat{g}$ is high, the NeRF rendering is considered reliable (texture-rich), so $\lambda=1$ (trust NeRF). If $\nabla \hat{g}$ is low, the NeRF rendering is unreliable (smooth/missing texture), so $\lambda=0$ (trust 2D prior) [S1].

5.  **Loss Computation:**
    Compute the total reconstruction loss $L$ as a weighted sum of non-occluded loss $L_{NonOcc}$ and occluded loss $L_{Occ}$ [S1]:
    $$
    L = \lambda L_{NonOcc} + (1 - \lambda) L_{Occ}
    $$
    
    *   **Non-occluded Loss ($L_{NonOcc}$):**
        $$
        L_{NonOcc} = \sum_{r \in R} (1 - \hat{P}(r)) \left[ ||\hat{C}_c(r) - C(r)||_2^2 + ||\hat{C}_f(r) - C(r)||_2^2 \right]
        $$
        where $\hat{P}(r)$ is the predicted mask value, $\hat{C}_c(r)$ and $\hat{C}_f(r)$ are coarse and fine network outputs, and $C(r)$ is the ground truth color [S1].
        
    *   **Occluded Loss ($L_{Occ}$):**
        $$
        L_{Occ} = \sum_{r \in R} \hat{P}(r) \left[ ||\hat{C}_c(r) - C_l(r)||_2^2 + ||\hat{C}_f(r) - C_l(r)||_2^2 \right]
        $$
        where $C_l(r)$ is the color of the corresponding pixel in the 2D inpainting image [S1].

6.  **Optimization:**
    Minimize $L$ with respect to NeRF parameters $\theta$ using gradient descent [S1]. The segmentation network is also trained jointly using region encoding, RGB, and depth losses to ensure multi-view consistency [S1].

**Parameters:**
*   $\tau$: Threshold for gradient comparison. Not given in the sources.
*   $\lambda_1, \lambda_2, \lambda_3$: Weights for segmentation loss components ($L_{region-encoding}, L_{rgb}, L_{depth}$). Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Uniform Strategy Misjudgment:** Applying a single strategy to the entire occluded area may fail if some sub-regions have rich texture (reliable NeRF) and others are smooth (unreliable NeRF). The sub-region division mitigates this [S1].
*   **2D Inpainter Inconsistency:** Single 2D inpainting does not consider 3D consistency. The method relies on multi-view correlation to correct this, but if no other views provide supplementary information, it falls back to the 2D prior, which may still exhibit 3D inconsistencies [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of views, sub-regions, and NeRF training iterations. The additional cost is the 2D inpainting inference and gradient calculation per sub-region.
*   **Memory Complexity:** Requires storing masks for all views and intermediate 2D inpainting results.
*   **Strengths:** Improves fidelity of de-occluded results by leveraging multi-view correlation and avoiding the disturbing effects of 2D inpainting in texture-rich areas [S1].
*   **Limitations:** Relies on the quality of the 2D inpainter (LaMa) and the accuracy of the segmentation network [S1].

## Background
This method was proposed in the paper "3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting" [S1]. It builds upon NeRF [21] and 2D inpainting techniques [25], addressing the limitation that single 2D image inpainting does not consider consistency in the real 3D scene [S1].

## Sources
[S1] 3D Scene De-occlusion in Neural Radiance Fields: A Framework for Obstacle Removal and Realistic Inpainting, https://doi.org/10.1145/3664647.3681429, fetched 2026-11-01
