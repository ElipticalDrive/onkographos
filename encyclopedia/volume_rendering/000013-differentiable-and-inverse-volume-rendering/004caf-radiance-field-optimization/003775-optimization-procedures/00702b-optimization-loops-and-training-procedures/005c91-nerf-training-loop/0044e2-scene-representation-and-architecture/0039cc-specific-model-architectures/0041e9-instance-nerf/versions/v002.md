# Instance-NeRF

Also called: Instance Neural Radiance Field

## Overview
Instance-NeRF is a learning-based 3D instance segmentation pipeline that extends a pre-trained NeRF with an instance field to detect objects, producing bounding boxes, continuous 3D masks, and class labels [S1]. It distinguishes itself from semantic NeRF variants by resolving multi-view inconsistency in instance IDs without relying on 3D bounding volume priors, using a 2D mask matching procedure to supervise the continuous instance field [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A pre-trained NeRF model $F_{\Theta_\sigma, \Theta_c}$ parameterized by density net $\Theta_\sigma$ and color branch $\Theta_c$, trained on posed RGB images [S1].
*   **Output:** A continuous 3D instance segmentation represented by the instance field $G_{\Theta_\sigma, \Theta_i}$, which outputs an $L$-dimensional instance label vector $i$ at any 3D position, where the first dimension represents the background [S1].

**Data Structures:**
The model consists of three components:
1.  **Density Net ($\Theta_\sigma$):** Part of the pre-trained NeRF, outputs volume density $\sigma$ [S1].
2.  **Color Branch ($\Theta_c$):** Part of the pre-trained NeRF, outputs RGB radiance $c$ [S1].
3.  **Instance Branch ($\Theta_i$):** The new component, outputs the instance label vector $i$ [S1].

**Numbered Steps:**
1.  **Ray Formulation:** For each pixel, formulate a ray $r(t) = o + td$ emitted from the camera center $o$ in direction $d$ [S1].
2.  **Sampling:** Select $K$ quadrature points $\{t_k\}_{k=1}^K$ on the ray $r$ within the traceable volume [S1].
3.  **NeRF-RCNN Prediction:** Use NeRF-RCNN (extended from NeRF-RPN) on the pre-trained NeRF's radiance and density fields to output 3D bounding boxes, class labels, and discrete 3D masks for detected objects [S1].
4.  **2D Segmentation Generation:** Generate multi-view 2D panoptic segmentation maps of the scene using existing methods (e.g., Mask2Former) on rendered images [S1].
5.  **Mask Matching and Refinement:** Project the discrete 3D masks from NeRF-RCNN back to 2D image space using the same camera poses. Use Mask2Former and CascadePSP to match the same instance across different views and refine the resulting masks, resolving multi-view inconsistency [S1].
6.  **Instance Field Training:** Train the instance field component $G_{\Theta_\sigma, \Theta_i}$ using the multi-view consistent 2D segmentation masks as supervision to produce continuous 3D segmentation [S1].

**Formulas:**
The pre-trained NeRF is formulated as [S1]:
$$ F_{\Theta_\sigma, \Theta_c}(x, d) = (\sigma, c) $$
where $x$ is a 3D position and $d$ is a viewing direction.

The Instance Field is formulated as [S1]:
$$ G_{\Theta_\sigma, \Theta_i}(x) = i $$
where $i$ is an $L$-dimensional vector with its first dimension representing the background.

The numerical quadrature to accumulate expected color $\hat{C}(r)$, expected depth $\hat{D}(r)$, and expected instance logits $\hat{I}(r)$ for a ray $r$ are [S1]:

$$ \hat{C}(r) = \sum_{k=1}^{K} \hat{T}(t_k) \alpha(\sigma(t_k)\delta_k) c(t_k) $$

$$ \hat{D}(r) = \sum_{k=1}^{K} \hat{T}(t_k) \alpha(\sigma(t_k)\delta_k) t_k $$

$$ \hat{I}(r) = \sum_{k=1}^{K} \hat{T}(t_k) \alpha(\sigma(t_k)\delta_k) i(t_k) $$

where:
*   $\hat{T}(t_k)$ is the transmittance at point $t_k$.
*   $\alpha$ is the opacity function.
*   $\sigma(t_k)$ is the density at point $t_k$.
*   $\delta_k$ is the distance interval.
*   $c(t_k)$ is the radiance at point $t_k$.
*   $i(t_k)$ is the instance label vector at point $t_k$.
*   $t_k$ is the position along the ray.

**Parameters:**
*   $K$: Number of quadrature points. Not given in the sources.
*   $L$: Number of instance labels including background. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Multi-view Inconsistency:** 2D instance masks are generally multi-view inconsistent, as the same object may be assigned different instance IDs in different views. This can impair performance if not resolved [S1].
*   **Ambiguity in Overlapping Volumes:** Unlike methods using 3D bounding volumes as priors (e.g., Panoptic NeRF), Instance-NeRF does not rely on such priors, avoiding ambiguity when objects of the same class have overlapping bounding volumes [S1].

## Operating characteristics
Not given in the sources.

## Background
Instance-NeRF is one of the first learning-based approaches to perform 3D instance segmentation in NeRF without ground-truth segmentation input [S1]. It was proposed by Yichen Liu, Benran Hu, Junkai Huang, Yu-Wing Tai, and Chi-Keung Tang from The Hong Kong University of Science and Technology, Carnegie Mellon University, and Dartmouth College [S1]. The method was published in ICCV 2023 [S1].

## Sources
[S1] Instance Neural Radiance Field, https://doi.org/10.1109/ICCV51070.2023.00079, fetched 2026-10-29
