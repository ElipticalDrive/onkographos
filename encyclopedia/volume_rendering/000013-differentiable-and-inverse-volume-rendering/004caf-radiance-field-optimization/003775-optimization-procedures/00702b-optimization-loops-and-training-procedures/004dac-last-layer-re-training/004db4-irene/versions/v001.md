# IReNe

Also called: Instant Recoloring of Neural Radiance Fields

## Overview
IReNe is a method for instant 360-degree recoloring of pre-trained Neural Radiance Fields (NeRFs) using a single user-edited image [S1]. It addresses the limitations of prior methods by preventing color bleeding at object boundaries and ensuring multi-view consistency for view-dependent effects, such as specularities [S1]. Unlike ICE-NeRF, which suffers from color bleeding, or Palette-NeRF, which requires hours to train, IReNe achieves high-quality edits in under 5 seconds by selectively fine-tuning only the diffuse neurons in the last layer of the color MLP [S1].

## How it works
**Inputs:**
*   A pre-trained NeRF model (specifically Instant-NGP [S1]), consisting of a density MLP $\sigma$, multiresolution hash features $f$, and a color MLP $c$.
*   A single training image $I_{edit}$ containing user-applied color edits [S1].

**Outputs:**
*   An edited image $I_{render}$ generated via volumetric rendering [S1].

**Data Structures:**
*   **Multiresolution Hash Grid Features ($f_x$):** The feature vector for a 3D point $x$ is a concatenation of features from $L$ levels of a multiresolution hash grid [S1].
    $$f_x = \{f_{x,0} \oplus f_{x,1} \oplus \dots \oplus f_{x,L}\}$$
    where $\oplus$ denotes concatenation [S1].
*   **Density MLP ($MLP_\sigma$):** Takes feature $f_x$ and outputs density $\sigma_x$ and hidden feature $h_x$ [S1].
    $$MLP_\sigma(f_x, \phi_\sigma) = \sigma_x, h_x$$
*   **Color MLP ($MLP_c$):** Takes hidden feature $h_x$ and encoded view direction $\gamma(\theta)$ to output color $c_{x,\theta}$ [S1].
    $$MLP_c(h_x, \gamma(\theta), \phi_c) = c_{x,\theta}$$
    Instant-NGP uses spherical harmonics for the view direction encoding $\gamma(\cdot)$ [S1].
*   **Segmentation MLP ($MLP_s$):** A lightweight, trainable MLP ruled by features $f$ to estimate a volumetric soft-segmentation $\alpha$ of the edited region [S1].
*   **Neuron Classification:** Neurons in the last layer of $MLP_c$ are classified as either "diffuse" or "view-dependent" [S1].

**Algorithm Steps:**
1.  **Initialize:** Load the pre-trained Instant-NGP NeRF [S1].
2.  **Detect Diffuse Neurons:** Automatically detect diffuse neurons in the last layer of the color MLP $c$. This is done by analyzing the activations of the last layer neurons across different view directions. Diffuse neurons exhibit color consistency across views, while view-dependent neurons (e.g., specularities) change color with the viewing angle [S1].
3.  **Train Segmentation:** Train the segmentation MLP $MLP_s$ using the multiresolution hash features $f$ to estimate the volumetric soft-segmentation mask $\alpha$ for the edited region. The optimization minimizes the RGB loss between the rendered image and the user-edited image $I_{edit}$ [S1].
4.  **Fine-tune Diffuse Neurons:** Fine-tune only the weights corresponding to the identified diffuse neurons in the last layer of $MLP_c$. The view-dependent neurons remain frozen to preserve view-specific effects [S1].
5.  **Compute Blended Color:** For any 3D point $x$ and view direction $\theta$, compute the final color $C_{final}$ as a linear combination of the color predicted by the frozen weights ($c_{x,\theta}$) and the color predicted by the retrained weights ($c'_{x,\theta}$), blended using the segmentation mask $\alpha_x$ [S1]:
    $$C_{final} = (1 - \alpha_x) c_{x,\theta} + \alpha_x c'_{x,\theta}$$
    where $c_{x,\theta}$ is the output of the original frozen $MLP_c$ and $c'_{x,\theta}$ is the output of the $MLP_c$ with the fine-tuned last layer [S1].
6.  **Volumetric Rendering:** Perform standard volumetric rendering using the density $\sigma_x$ and the blended color $C_{final}$ to obtain the edited image $I_{render}$ [S1].

**Optimization:**
The segmentation MLP $MLP_s$ and the trainable last-layer diffuse neurons are optimized simultaneously. The loss function is the standard RGB loss between the rendered image $I_{render}$ and the user-edited image $I_{edit}$ [S1]:
$$L = \sum \| C_{ij} - C'_{ij} \|^2$$
where $C_{ij}$ is the pixel value in $I_{edit}$ and $C'_{ij}$ is the rendered pixel value [S1].

**Parameters:**
*   **Optimization Time:** Approximately 5 seconds [S1].
*   **Input Resolution:** Depends on the input image $I_{edit}$ [S1].

**Edge Cases and Pitfalls:**
*   **Color Bleeding:** Without the segmentation module $\alpha$, color edits bleed into adjacent objects. The trainable segmentation module prevents this by providing spatial control over the edit [S1].
*   **View-Dependent Consistency:** If all neurons in the last layer were fine-tuned, view-dependent effects like specularities would be distorted or lost. Classifying and freezing view-dependent neurons ensures multi-view consistency [S1].
*   **Complex 360-Degree Scenes:** Unlike methods relying on reprojection which fail in complex 360-degree scenes, IReNe handles them consistently by operating directly in the 3D volume representation [S1].

## Operating characteristics
*   **Time Complexity:** Extremely fast optimization, typically under 5 seconds per edit [S1]. This is 5× to 500× faster than competitors like Palette-NeRF (hours) and ICE-NeRF (seconds, but with quality issues) [S1].
*   **Memory Complexity:** Low additional memory overhead due to the lightweight segmentation MLP and the fact that only a subset of the last layer weights are trained [S1].
*   **Strengths:** Near real-time interactive editing, prevention of color bleeding, multi-view consistency for specularities, and support for partial and multi-object edits [S1].
*   **Limitations:** Requires a pre-trained Instant-NGP model; relies on the accuracy of the automatic diffuse neuron classification [S1].
*   **When to use:** When instant, high-quality color editing of a NeRF is required, especially for scenes with complex boundaries or view-dependent effects.
*   **When not to use:** When the underlying NeRF is not trained with Instant-NGP or when global scene reconstruction (not just color editing) is needed.

## Background
IReNe was introduced by Mazzucchelli et al. in 2024 [S1]. It builds upon the Instant-NGP architecture [S1] and addresses limitations in previous NeRF editing methods such as Palette-NeRF [S1] and ICE-NeRF [S1]. The method was validated on a new dataset with edited object colors, demonstrating significant quantitative and qualitative advancements [S1].

## Sources
[S1] IReNe: Instant Recoloring of Neural Radiance Fields, https://openaccess.thecvf.com/content/CVPR2024/papers/Mazzucchelli_IReNe_Instant_Recoloring_of_Neural_Radiance_Fields_CVPR_2024_paper.pdf, fetched 2026-10-29
