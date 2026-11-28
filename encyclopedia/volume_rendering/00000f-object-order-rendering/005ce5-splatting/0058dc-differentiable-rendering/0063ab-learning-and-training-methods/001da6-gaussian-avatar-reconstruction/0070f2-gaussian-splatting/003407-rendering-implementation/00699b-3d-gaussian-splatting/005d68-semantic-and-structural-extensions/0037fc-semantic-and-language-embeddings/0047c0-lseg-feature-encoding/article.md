# LSeg Feature Encoding

Also called: LSeg

## Overview
LSeg Feature Encoding utilizes the LSeg (Language-Segment Anything) model to generate high-dimensional semantic feature maps from 2D input images, which are then distilled into 3D representations. It solves the problem of assigning rich, language-grounded semantic context to 3D scenes without requiring explicit 3D annotations. It differs from global embedding approaches by providing pixel-level or patch-level semantic supervision, allowing for localized semantic editing and segmentation within the 3D scene [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D image $I$ and a set of text queries (class names or phrases).
*   **Output:** A semantic feature map $\mathbf{f}_{img}(I, r)$ aligned with the image resolution.

**Data Structures:**
The LSeg model consists of two encoders:
1.  **Image Encoder:** Vision Transformer Large with 16-pixel patches (ViT-L/16) [S1].
2.  **Text Encoder:** CLIP’s Vision Transformer Base with 32-pixel patches (ViT-B/32) [S1].

**Steps:**
1.  **Text Encoding:** Encode the input text queries using the CLIP ViT-B/32 text encoder to obtain text embeddings [S1].
2.  **Image Encoding:** Process the input image $I$ through the ViT-L/16 image encoder. The encoder outputs feature maps at reduced resolution.
3.  **Multi-Scale Inference:** To stabilize features and alleviate artifacts from discrete patch processing, perform inference at multiple scales. Resize the image to each scale $s \in S$, encode, and interpolate the resulting feature maps back to the original image resolution.
    *   For the Replica dataset, scales $S = [0.75, 0.83, 0.92, 1.0, 1.08, 1.17, 1.25]$ [S1].
    *   For other datasets, scales $S = [0.75, 1.0, 1.25, 1.5, 1.75, 2.0, 2.25]$ [S1].
4.  **Aggregation:** Average the interpolated feature maps from all scales to produce the final feature map $\mathbf{f}_{img}(I, r)$ [S1].
5.  **Distillation Target:** The resulting feature map is used as the target feature map for distilling semantic features into the 3D representation (e.g., NeRF or Gaussian Splatting) [S1].

**Pitfalls:**
*   LSeg exhibits bias towards traffic scenes due to its training data (ADE20K, BDD, Cityscapes, COCO-Panoptic, IDD, Mapillary Vistas, SUN RGBD) [S1].
*   It performs poorly in out-of-distribution regions and queries. Small objects or parts are often not discriminative [S1].
*   Large inference scales lose context information, potentially producing noisy features with incorrect semantic understanding [S1].

## Subtopics

### [LSeg Teacher Network for Feature Distillation](12082-lseg-teacher-network-for-feature-distillation.md)
Uses LSeg as a teacher to distill semantic features into 3D representations, leveraging pretrained CLIP models. It provides pixel-level semantic supervision for localized editing, unlike global embedding approaches.

## Operating characteristics
*   **Strengths:** Provides rich, zero-shot semantic understanding at the pixel level. Multi-scale inference stabilizes features against ViT patch artifacts.
*   **Limitations:** Computationally expensive due to multi-scale inference and large ViT-L backbone. Performance degrades on small objects or out-of-distribution scenes due to dataset bias [S1].
*   **When to use:** When localized semantic editing or segmentation is required in 3D scenes and 2D supervision is available.
*   **When not to use:** For scenes with significant out-of-distribution content or where small object discrimination is critical.

## Background
LSeg (Language-Segment Anything) is a vision-language model that combines language understanding with image segmentation. It was developed by the Facebook AI Research team and the Institute of Science Tokyo. The model leverages pretrained CLIP models for text and image encoding [S1]. It is trained on seven datasets: ADE20K, BDD, Cityscapes, COCO-Panoptic, IDD, Mapillary Vistas, and SUN RGBD [S1]. The specific implementation used for feature encoding in 3D reconstruction is the official demo model [S1].

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
