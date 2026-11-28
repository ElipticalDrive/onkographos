# Language Embedded 3D Gaussians

Also called: LE3DG

## Overview
Language Embedded 3D Gaussians (LE3DG) extends the feature blending of 3D Gaussian Splatting to full scene understanding by encoding object-level embeddings for phrase grounding in addition to pixel-level features [S1, unofficial]. It enables open-vocabulary semantic segmentation and zero-shot semantic reasoning by allowing rendered feature maps to be compared directly against text embeddings from vision-language models like CLIP [S1, unofficial]. It differs from sibling methods like [LangSplat](8104-langsplat.md) by focusing on the expansion of the standard feature blending pipeline to support object-level grounding rather than just pixel-level alignment, and from [3D Vision-Language GS](8105-3d-vision-language-gs.md) which projects vision-language models into a joint embedding space for visual question answering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\mathcal{G}$, each with position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and a feature vector $\mathbf{f}_i$. The feature vector $\mathbf{f}_i$ is initialized or optimized to represent semantic information (e.g., CLIP image embeddings) rather than just color.
*   **Output:** A rendered 2D feature map $\mathbf{V}(p)$ for each pixel $p$, which can be decoded by a segmentation head or compared against text embeddings.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian $i$ stores a feature vector $\mathbf{f}_i$. The dimensionality of $\mathbf{f}_i$ matches the embedding space of the vision-language model (e.g., 512 or 768 dimensions for CLIP).
*   **Rasterization Buffer:** Stores the accumulated feature vector $\mathbf{V}(p)$ and accumulated opacity $\alpha_{acc}(p)$ for each pixel.

**Algorithm:**
1.  **Rasterization:** For each pixel $p$, identify the set of overlapping Gaussians $\mathcal{O}_p$ sorted front-to-back.
2.  **Feature Blending:** Compute the rendered feature vector $\mathbf{V}(p)$ using the shared differentiable alpha-blending accumulation defined in the parent article [Parent](8130-semantic-and-language-embeddings.md):
    $$
    \mathbf{V}(p) = \sum_{i \in \mathcal{O}_p} \mathbf{f}_i \cdot \alpha_i \cdot \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\mathbf{f}_i$ is the semantic feature vector of the $i$-th Gaussian [S1, unofficial].
3.  **Semantic Query:** To perform open-vocabulary segmentation, encode a target text phrase $T$ into a text embedding $\mathbf{t}_T$ using a pre-trained vision-language model (e.g., CLIP text encoder).
4.  **Matching:** Compute the similarity between the rendered feature $\mathbf{V}(p)$ and the text embedding $\mathbf{t}_T$ (e.g., using cosine similarity) to determine the semantic label for pixel $p$.

**Parameters:**
*   **Feature Dimension:** Determined by the vision-language model (e.g., 512 for CLIP ViT-L/14). Not given in the sources.
*   **Similarity Metric:** Cosine similarity is standard for CLIP embeddings. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Feature Mixing:** Overlapping Gaussians with different semantic meanings may produce mixed feature vectors that do not match any single text embedding well. This is mitigated by the differentiable blending which preserves the contribution of each Gaussian proportional to its opacity [S1, unofficial].
*   **Precision:** High-dimensional feature vectors require sufficient precision during accumulation to avoid quantization errors that degrade semantic similarity. [G] Use float32 or float16 with careful handling of underflow for low-opacity Gaussians.

## Subtopics

### [3D Semantic Segmentation Post-processing (3D-OVS)](19840-3d-semantic-segmentation-post-processing-3d-ovs.md)
3D-OVS is a post-processing pipeline for semantic segmentation in 3D Gaussian Splatting systems that resolves ambiguity by selecting the optimal semantic level from multiple candidate relevancy maps. It solves the problem of inconsistent or noisy semantic predictions by thresholding, binarizing, and evaluating average relevancy within masks to choose the best binary mask. It differs from direct thresholding by explicitly comparing multiple semantic levels (derived from different granularities) to mitigate outlier impact.

### Vision-Language Embedding Fusion
A strategy to extract vision-language embeddings for 3DGS by aligning each Gaussian directly in the image-embedding space using a dynamic weighting mechanism that combines global, local, and masked features [S2].
*   **Step 1:** Extract embeddings from the frames used to optimize the 3DGS [S2].
*   **Step 2:** Combine three distinct features using a dynamic weighting mechanism: global context from the entire frame, local features with background, and masked features without background [S2].
*   **Step 3:** Automatically balance contributions based on each segment's contextual relationship, emphasizing background-inclusive features for integrated objects and object-specific features for isolated objects [S2].

## Operating characteristics
*   **Time Complexity:** Rasterization is $O(N)$ per pixel where $N$ is the number of overlapping Gaussians, same as standard 3DGS. The additional cost is the dimensionality of the feature vector $\mathbf{f}_i$ during accumulation and the similarity computation against text embeddings.
*   **Memory Complexity:** Memory usage increases by the size of the feature vector $\mathbf{f}_i$ per Gaussian. For CLIP embeddings (e.g., 512 dims), this is significantly larger than color (3 dims). Not given in the sources.
*   **Strengths:** Enables real-time open-vocabulary semantic segmentation and phrase grounding without retraining the segmentation head for new classes [S1, unofficial].
*   **Limitations:** Requires a pre-trained vision-language model for text encoding. Semantic quality depends on the alignment between the 3DGS feature space and the VLM embedding space. Not given in the sources.

## Background
Language Embedded 3D Gaussians (LE3DG) was introduced as an expansion of feature blending to full scene understanding, encoding object-level embeddings for phrase grounding [S1, unofficial]. It builds on the foundation of 3D Gaussian Splatting with Feature Vectors [8060-3d-gaussian-splatting-with-feature-vectors.md] and integrates with the broader family of semantic and language embeddings [8130-semantic-and-language-embeddings.md]. The method is cited in the context of enabling open-vocabulary semantic segmentation in real time by comparing rendered feature maps against CLIP text embeddings [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] SceneSplat++: A Large Dataset and Comprehensive Benchmark for Language Gaussian Splatting, https://zenodo.org/records/19113098, fetched 2026-10-26
