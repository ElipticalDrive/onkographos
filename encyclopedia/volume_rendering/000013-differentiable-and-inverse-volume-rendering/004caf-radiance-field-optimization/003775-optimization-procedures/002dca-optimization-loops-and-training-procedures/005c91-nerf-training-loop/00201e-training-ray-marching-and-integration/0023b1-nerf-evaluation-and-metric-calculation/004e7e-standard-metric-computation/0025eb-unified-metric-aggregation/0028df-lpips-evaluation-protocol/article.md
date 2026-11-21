# LPIPS Evaluation Protocol

Also called: Learned Perceptual Image Patch Similarity

## Overview
LPIPS is a perceptual similarity metric used in volume rendering evaluation to quantify reconstruction fidelity [S1]. It differs from pixel-wise metrics by requiring a specific input intensity scaling step before feature comparison [S1].

## How it works
**Inputs and Outputs:**
- Input: Rendered image $I_{render}$ and ground-truth image $I_{gt}$.
- Output: Scalar LPIPS score $L$.

**Data Structures:**
- Feature maps: Tensors extracted from multiple layers of a pretrained network. Not given in the sources.
- Normalization parameters: Per-channel mean and standard deviation. Not given in the sources.

**Steps:**
1. **Intensity Scaling:** Scale pixel values of both $I_{render}$ and $I_{gt}$ to the range $[-1, 1]$ [S1]. This is critical for correct evaluation, as noted in comparisons with 3DGS variants [S1].
2. **Feature Extraction:** Pass both images through a pretrained convolutional neural network. Not given in the sources.
3. **Normalization:** Normalize the feature maps at each layer using dataset statistics. Not given in the sources.
4. **Distance Calculation:** Compute the squared Euclidean distance between normalized feature maps. Not given in the sources.
5. **Weighting:** Apply weights $w_l$ to each layer's distance. Not given in the sources.
6. **Aggregation:** Sum the weighted distances across all layers to obtain the final score. Not given in the sources.

**Formulas:**
- Intensity scaling: $I_{scaled} = 2 \cdot I_{norm} - 1$, where $I_{norm} \in [0, 1]$ [G].
- Feature distance: Not given in the sources.
- Final score: $L = \sum_{l=1}^{L} w_l d_l$. Not given in the sources.

**Parameters:**
- Intensity scale: $[-1, 1]$ [S1].
- Network architecture: Not given in the sources.
- Weights: Learned via regression against human judgments. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Intensity Range:** Failure to scale inputs to $[-1, 1]$ leads to incorrect LPIPS values, as highlighted in evaluations of 3DGS variants [S1].
- **Network Choice:** Consistency in network choice is required for fair comparison. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lpips_evaluation(rendered_image, ground_truth_image):
    # 1. Intensity Scaling to [-1, 1]
    rendered_scaled = 2.0 * (rendered_image / 255.0) - 1.0
    ground_truth_scaled = 2.0 * (ground_truth_image / 255.0) - 1.0
    # 2-6. Feature extraction, normalization, distance calculation, weighting, and aggregation
    # Not given in the sources.
    pass
```

## Subtopics

### [LPIPS Computation](5498-lpips-computation.md)
LPIPS Computation quantifies perceptual similarity by comparing deep feature activations from a pretrained convolutional neural network rather than raw pixel values [G]. It is chosen when perceptual quality is prioritized over pixel-wise accuracy, and differs from siblings by focusing on high-level semantic and textural alignment [G].

## Operating characteristics
- Time complexity: Not given in the sources.
- Memory complexity: Not given in the sources.
- Strengths: Aligns with human perception when evaluated correctly [S1].
- Limitations: Computationally more expensive than pixel-wise metrics [G].
- When to use: Evaluating reconstruction fidelity in volume rendering where perceptual quality matters [S1].
- When not to use: Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
