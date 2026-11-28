# Exposure Correction Strategy

Also called: Nearest-neighbor exposure mimicry

## Overview
This strategy corrects inconsistent exposure across training views caused by the auto-exposure behavior of modern cameras in radiance-field optimization [S1]. It addresses the limitation of prior methods that only correct exposure at training views by generalizing to images rendered at any novel viewpoint [S1]. Unlike approaches that jointly optimize an affine transformation matrix for each training view [6, 17], this method assumes that images taken at nearby locations share similar global lighting conditions and corrects camera exposure at novel views by mimicking the exposure setting of the closest source view [S1].

## How it works
The method operates within an image-based Gaussian Splatting pipeline [S1]. It relies on the spatial proximity of training views to infer exposure parameters for novel viewpoints.

**Inputs:**
*   Set of training images $S = \{I_i\}$.
*   Camera poses for training images.
*   Camera pose for the novel view $V_{novel}$.

**Outputs:**
*   Exposure correction factor applied to the rendered image at $V_{novel}$.

**Data Structures:**
*   Training view positions: 3D coordinates $\mathbf{p}_i$ for each $I_i \in S$.
*   Novel view position: 3D coordinate $\mathbf{p}_{novel}$.

**Steps:**
1.  Identify the nearest neighbor: Calculate the Euclidean distance between the novel view position $\mathbf{p}_{novel}$ and every training view position $\mathbf{p}_i$.
2.  Select the closest source view $I_{nn}$ such that:
    $$ I_{nn} = \arg \min_{I_i \in S} \| \mathbf{p}_{novel} - \mathbf{p}_i \| $$
3.  Retrieve the exposure setting (or implicit exposure scaling factor) associated with $I_{nn}$. In the context of the source, this is treated as a known property of the training data or derived from the auto-exposure metadata [S1].
4.  Apply the exposure correction from $I_{nn}$ to the rendered radiance at $V_{novel}$.

**Parameters:**
*   Distance metric: Euclidean distance in 3D space [S1].
*   Neighbor count: 1 (nearest neighbor only) [S1].

**Edge Cases and Pitfalls:**
*   **Sparse Training Data:** If the nearest neighbor is far away, the assumption of similar global lighting conditions may fail, leading to incorrect exposure correction [S1].
*   **Dynamic Exposure:** When exposure differences are significantly dynamic among neighborhood views, the method may still struggle if the nearest neighbor has a vastly different exposure setting, potentially causing the renderer to misinterpret exposure differences as high-frequency reflectance [S2].

## Subtopics

### White Balance Initialization
This is an initialization strategy to resolve the ambiguity between exposure and white balance, stabilizing optimization by guiding initial solutions to physically plausible ones [S2]. It addresses the issue where the overall scale of white balance becomes extremely small or large during training [S2].

**Steps:**
1.  Calculate the averaged color $(r_a, g_a, b_a) \in \mathbb{R}^3$ of all input images in the set $S$.
2.  Select a reference image $I_{ref}$ which has the closest averaged color value $(r_{ref}, g_{ref}, b_{ref}) \in \mathbb{R}^3$ to $(r_a, g_a, b_a)$.
3.  Fix the white balance of this reference image as regularization.
4.  Estimate the reference color ratio by comparing per-image averaged pixel values to the averaging of each RGB value from the entire image set $S$.
5.  Initialize the white balances $w_{c,i}$ for each camera $c \in \{r, g, b\}$ of each image $I_i$ as:
    $$ w_{c,i} = \frac{\text{mean}_{k \in I_i}(c_k)}{\text{mean}_{j \in S}(c_j)} $$
    where $c_k$ is the pixel value for channel $c$ in image $I_i$, and $c_j$ is the pixel value for channel $c$ in image $I_j$ [S2].

This initialization helps the white balance be learned on the proper scale, ensuring a suitable exposure value and stabilizing early optimization steps, particularly when LDR images from neighborhood views have significantly dynamic exposure differences [S2].

## Operating characteristics
*   **Time Complexity:** The nearest-neighbor search is $O(N)$ per novel view, where $N$ is the number of training views. This is negligible compared to the rendering cost.
*   **Memory Complexity:** $O(N)$ to store training view positions.
*   **Strengths:** Generalizes exposure correction to any viewpoint, not just training views [S1]. Simple to implement.
*   **Limitations:** Relies on the assumption of similar lighting between nearby views. May fail in scenes with rapid lighting changes or sparse training data [S1].
*   **When to use:** When training data is captured with auto-exposure cameras and views are sufficiently dense [S1].
*   **When not to use:** When lighting conditions vary significantly over short distances or when training views are sparse [S1].

## Background
The strategy was introduced in the context of Image-Based Gaussian Splatting (IBGS) to address photometric inconsistencies arising from auto-exposure in modern cameras [S1]. It builds on the observation that prior works [6, 17] only corrected exposure at training views, limiting their applicability to novel viewpoints [S1]. The related White Balance Initialization strategy was proposed in HDR-Plenoxels to resolve scale ambiguity between exposure and white balance, and between SH coefficients and white balance, ensuring stable optimization [S2].

## Sources
[S1] IBGS: Image-Based Gaussian Splatting, https://arxiv.org/abs/2511.14357v1, fetched 2026-10-26
[S2] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
