# Semantic and Language Embeddings

Also called: Gaussian Grouping, SAM-guided Gaussian Grouping

## Overview
This family extends 3D Gaussian Splatting by attaching high-dimensional semantic or language feature vectors to each Gaussian, enabling open-vocabulary segmentation, phrase grounding, and visual question answering via real-time alpha-blended feature maps. It differs from sibling methods by prioritizing semantic understanding over geometric fidelity or appearance; while [3D Gaussian Splatting MVP](3410-3d-gaussian-splatting-mvp.md) focuses on minimal viable rendering and [Analytic-Splatting](8084-analytic-splatting.md) on mathematical precision, this approach leverages the rasterizer as a feature distillation pipeline for vision-language models.

## Shared principles
The core mechanism involves storing a feature vector $\mathbf{f}_i$ (of dimension $D$, typically 512 or 768) alongside the standard Gaussian parameters (position $\mathbf{x}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, spherical harmonics coefficients) for each Gaussian $i$. During rendering, these features are accumulated per pixel using alpha-blending, producing a 2D feature map $\mathbf{F}(p)$ that can be compared against text embeddings or decoded by a segmentation head.

The feature blending formula is identical to the color blending in standard 3D Gaussian Splatting [S2, unofficial]:

$$
\mathbf{F}(p) = \sum_{i \in \mathcal{O}} \mathbf{f}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
$$

where:
- $\mathcal{O}$ is the set of Gaussians sorted by depth (front-to-back) that overlap pixel $p$.
- $\mathbf{f}_i$ is the feature vector of Gaussian $i$.
- $\alpha_i$ is the opacity of Gaussian $i$.
- The term $\prod_{j=1}^{i-1} (1 - \alpha_j)$ represents the accumulated transparency from all Gaussians $j$ rendered before $i$ in the sorted order.

The resulting $\mathbf{F}(p)$ is a vector in $\mathbb{R}^D$. To enable semantic tasks, $\mathbf{F}(p)$ is compared against text embeddings $\mathbf{t}_k$ (e.g., from CLIP) using cosine similarity or dot product to determine class labels or phrase grounding scores [S2, unofficial].

## Subtopics

### 3D Gaussian Splatting with Feature Vectors
This is the foundational variant that adds a high-dimensional feature vector $\mathbf{f}_i$ to each Gaussian, rendered through alpha-blending to produce a feature map for segmentation or CLIP comparison [S2, unofficial]. It enables open-vocabulary semantic segmentation by comparing the rendered feature map against text embeddings from CLIP in real time [S2, unofficial]. The method pioneers the use of 3DGS as a platform for distilling 2D vision-model features into a 3D representation [S2, unofficial].

### Language Embedded 3D Gaussians
This variant expands feature blending to full scene understanding, encoding not only pixel-level features but also object-level embeddings for phrase grounding [S2, unofficial]. It allows for more complex semantic queries beyond simple pixel-wise classification, enabling the system to understand relationships between objects and text phrases [S2, unofficial].

### LangSplat
LangSplat pioneers open-vocabulary semantic segmentation in real time by rendering feature maps that are compared against text embeddings from CLIP [S2, unofficial]. It demonstrates that 3DGS can support interactive semantic queries without requiring explicit 3D semantic labels during training, relying instead on the alignment of 2D vision-language features [S2, unofficial].

### 3D Vision-Language GS
This method projects vision-language models into the 3DGS framework to enable 3D visual question answering and zero-shot semantic reasoning via natural language queries on rendered feature maps [S2, unofficial]. It trains a joint embedding space where the rendered feature map can be queried with natural language, enabling tasks like 3D visual question answering and zero-shot semantic reasoning [S2, unofficial].

### Text Embedding Generation with Directional Views
This variant encodes text prompts into latent embeddings using a frozen CLIP text encoder, and generates directional view embeddings by appending view descriptors to prompts [S3, unofficial]. It allows for view-dependent semantic queries, where the semantic meaning of an object may vary based on the viewing angle [S3, unofficial].

The implementation details for text embedding generation are as follows [S3, unofficial]:

1.  **Tokenization**: The prompt is tokenized with padding to the maximum length:
    ```python
    inputs = self.tokenizer(prompt, padding="max_length", max_length=self.tokenizer.model_max_length, return_tensors="pt")
    ```
2.  **Encoding**: The token IDs are passed through the frozen text encoder:
    ```python
    embeddings = self.text_encoder(inputs.input_ids.to(self.device))[0]
    ```
3.  **Directional Views**: For directional views, the string `', {d} view'` is appended to each prompt for each direction $d \in \{'front', 'side', 'back'\}$, encoded, and stored in `self.embeddings` [S3, unofficial].

The layout of the stored embeddings is a dictionary mapping keys `'pos'`, `'neg'`, `'front'`, `'side'`, `'back'` to tensors of shape `[1, 77, 768]` (batch, sequence length, embedding dimension) [S3, unofficial].

## Background
The concept of attaching semantic features to 3D Gaussians emerged as a natural extension of 3D Gaussian Splatting's ability to render high-dimensional data. Feature-3DGS and LangSplat were among the first to demonstrate that 2D vision-language models (like CLIP) could be distilled into 3D Gaussians for open-vocabulary segmentation [S2, unofficial]. Language Embedded 3D Gaussians and 3D Vision-Language GS further expanded this to support phrase grounding and visual question answering [S2, unofficial]. The "Gaussian Grouping" approach, supervised by SAM (Segment Anything Model) or video tracking, provides an alternative method for assigning per-Gaussian instance IDs, focusing on instance segmentation rather than open-vocabulary semantics [S1, unofficial]. Text Embedding Generation with Directional Views introduces view-dependent semantic embeddings, allowing for more nuanced semantic queries based on camera pose [S3, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] dreamgaussian/dreamgaussian, https://github.com/dreamgaussian/dreamgaussian, fetched 2026-10-27 (unofficial)
