# CLIP Distillation for Semantic NeRFs
Also called: Kobayashi et al. (CLIP Distillation)

## Overview
A method that distills knowledge from an off-the-shelf 2D feature extractor (CLIP-based) into 3D feature fields, optimized in conjunction with in-scene radiance fields to produce a NeRF model with semantic understanding [S1]. It sets itself apart from standard photometric NeRFs by enabling zero-shot segmentation from an open set of text labels or queries, and from direct supervision methods by using novel-view features as a teacher signal rather than ground-truth labels [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input images with known camera poses; an off-the-shelf CLIP-based 2D feature extractor (teacher); a text query or label set for segmentation.
*   **Outputs:** A 3D radiance field model (MLPs) that outputs color and semantic feature vectors for any 3D point and viewing direction.

**Data Structures:**
*   **3D Feature Field:** Represented by Multi-Layer Perceptrons (MLPs) [S1]. The network takes 3D coordinates and viewing directions as input and outputs a feature vector. The specific dimensionality of the feature vector is not given in the sources.
*   **Teacher Features:** Extracted from novel views using the CLIP encoder.

**Steps:**
1.  **Initialize Radiance Field:** Initialize the standard NeRF MLPs for color and density, along with the semantic feature MLPs [S1].
2.  **Render Features:** For a given camera pose, sample points along rays. Use the semantic MLP to predict feature vectors for these points. Aggregate these features along the ray (e.g., via volume rendering) to produce a rendered 2D feature map for the view [S1].
3.  **Extract Teacher Features:** Use the off-the-shelf CLIP-based 2D feature extractor to extract features from the input images (or novel views if available) [S1].
4.  **Compute Distillation Loss:** Calculate the difference between the rendered 3D features and the extracted 2D teacher features. The specific loss function (e.g., MSE) is not explicitly defined in the provided notes, but standard distillation implies minimizing this distance [S1].
5.  **Optimize:** Update the NeRF parameters (including semantic MLPs) to minimize the distillation loss in conjunction with the standard radiance field losses (e.g., photometric loss) [S1].
6.  **Inference:** Use the trained model to render semantic feature maps for any novel view. Perform zero-shot segmentation by comparing rendered features to text embeddings from the CLIP text encoder [S1].

**Parameters:**
*   Learning rate: Not given in the sources.
*   Feature dimension: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Novel View Availability:** The method relies on "novel-view features" as a teacher signal. If novel views are not available during training, the source does not specify the fallback mechanism [S1].
*   **Precision:** Not given in the sources.

## Subtopics

### [2D Semantic Distillation Loss](15121-2d-semantic-distillation-loss.md)
The 2D Semantic Distillation Loss is a regularization term used in semantic-aware NeRF training to enhance the discrimination and semantic consistency of rendered features. It addresses the problem of weak semantic understanding in standard radiance fields by distilling knowledge from an off-the-shelf 2D feature extractor (specifically CLIP-based features from novel views) into the 3D feature field. It sets itself apart from standard color losses by operating in the semantic feature space rather than RGB space, and from direct supervision by using novel-view features as a teacher signal rather than ground-truth labels. It builds on [CLIP Distillation for Semantic NeRFs](11968-clip-distillation-for-semantic-nerfs.md).

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables zero-shot segmentation from an open set of text labels or queries [S1]. Allows for scene editing via manipulating object-specific MLPs [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When semantic understanding and zero-shot segmentation are required without ground-truth semantic labels.
*   **When not:** When only photometric accuracy is needed.

## Background
Originated by Kobayashi et al. in May 2022 [S1]. The work demonstrated that distilling CLIP-based features into 3D fields allows for semantic segmentation synthesis, instance segmentation synthesis, and scene editing [S1]. It is part of the broader effort to add semantic capabilities to NeRFs, following works like SS-NeRF (June 2022) which used different encoding strategies [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
