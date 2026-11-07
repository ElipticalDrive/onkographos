# 3D Gaussian Splatting LOD methods

Also called: RTGS

## Overview
3D Gaussian Splatting Level-of-Detail (LOD) methods address the scalability and memory constraints of standard 3D Gaussian Splatting (3DGS) by reducing the number of primitives or compressing their attributes to enable real-time rendering of large-scale scenes or efficient storage [S1, S2, unofficial][S3]. Unlike monolithic 3DGS which loads all Gaussians into GPU memory, these methods employ hierarchical structures, spatial partitioning, or attribute quantization to dynamically select detail levels based on camera distance or view importance [S1, S2, unofficial][S3]. They set themselves apart from sibling methods like [Hierarchical Splatting](1168-hierarchical-splatting.md) by specifically targeting the parametric nature of 3D Gaussians, often combining spatial pruning with semantic or radiance-based importance metrics [S1, S2, unofficial].

## How it works
The core mechanism involves reducing redundancy in the Gaussian set through spatial overlap analysis, temporal consistency checks, or hierarchical clustering [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\mathcal{G} = \{g_i\}$, where each $g_i$ is defined by position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, rotation $q_i$, scaling $s_i$, and spherical harmonic (SH) coefficients $c_{i, l, m}$ [S1, unofficial][S3].
*   **Output:** A reduced or compressed set of Gaussians $\mathcal{G}'$ suitable for rendering at a specific LOD or storage [S1, unofficial][S3].

**Data Structures:**
*   **Octree/Cluster Hierarchy:** Nodes store merged Gaussian parameters summarizing children [S2, unofficial].
*   **2D Grids:** Attributes arranged into sorted 2D grids for spatial locality [S3].
*   **Codebooks:** Vector quantization maps attributes to discrete indices [S3].

**Steps:**
1.  **Reduction/Pruning:** Remove Gaussians based on spatial overlap and temporal consistency [S1, unofficial]. Importance scores determine which Gaussians to retain [S3].
2.  **Quantization:** Quantize attributes such as position, scaling, rotation, and opacity to lower bit depths (e.g., 8–16 bits) [S2, unofficial][S3].
3.  **Compression:** Apply entropy coding (Huffman, zstd) or image compression (JPEG XL, OpenEXR) to the quantized data [S2, unofficial][S3].
4.  **Rendering:** Project the reduced set using standard 3DGS rasterization [S1, S2, unofficial].

**Formulas:**
*   **Importance Score:** Used to prune Gaussians. Not explicitly defined in sources, but based on contribution to image error [S3].
*   **Quantization:** $q' = \text{round}(q / \Delta) \cdot \Delta$, where $\Delta$ is the quantization step [S3].

**Parameters:**
*   **Quantization Bits:** 8–16 bits for geometric attributes, 8 bits for opacity [S2, unofficial].
*   **SH Band:** Distilled to band 1 for compression [S2, unofficial].

**Edge Cases:**
*   **Opacity Quantization:** Can lead to fewer floaters or visual artifacts [S3].
*   **Position Quantization:** Sharing positions can result in overlapping Gaussians, so some methods avoid quantizing position [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reduce_gaussians(gaussians, method="RTGS"):
    if method == "RTGS":
        # Remove based on spatial overlap and temporal consistency
        reduced = []
        for g in gaussians:
            if not is_redundant(g, reduced):
                reduced.append(g)
        return reduced
    elif method == "EAGLES":
        # Quantize all attributes
        quantized = []
        for g in gaussians:
            g.q = quantize(g.q, bits=16)
            g.s = quantize(g.s, bits=16)
            g.alpha = quantize(g.alpha, bits=8)
            quantized.append(g)
        return quantized
    elif method == "SOGS":
        # Arrange into 2D grids
        grids = arrange_into_grids(gaussians)
        apply_smoothness_regularization(grids)
        return grids
    else:
        return gaussians
```

## Subtopics

### Hierarchical 3DGS
A cluster-based Level of Detail (LoD) method developed by Inria, capable of rendering large scenes like Mont Saint-Michel interactively [S2, unofficial]. It uses an octree-like tree of clusters and renders only the relevant LoD, achieving a 99% reduction at far distances [S2, unofficial].

### LODGE
A level-of-detail framework for large-scale 3D Gaussian Splatting that enables efficient rendering by dynamically selecting appropriate detail levels [S1, unofficial].

### EAGLES
A method that quantizes all Gaussian attributes including color, position, opacity, rotation, and scaling [S3]. Quantization of opacity leads to fewer floaters or visual artifacts in the novel view synthesis task [S3]. On the MipNeRF360 dataset, EAGLES achieves SSIM 0.81, PSNR 27.15, LPIPS 0.24, and a size of 68 MB [S3].

### SOGS (Sparse Orthographic Grid Splatting)
A method that arranges Gaussian attributes into multiple 2D grids, sorts them, and applies a smoothness regularization to penalize pixels with values very different from their local neighborhood on the 2D grid [S3]. For storage, SOGS compresses the RGB grid with JPEG XL and stores all other attributes as 32-bit OpenEXR images with zip compression [S3]. On the MipNeRF360 dataset, SOGS achieves SSIM 0.801, PSNR 26.98, LPIPS 0.238, and a size of 28.80 MB [S3].

### 3D Gaussian Splatting Compression
A compression method that does not quantize opacity and position because sharing them results in overlapping Gaussians [S3]. On the MipNeRF360 dataset, Compact3D achieves SSIM 0.808, PSNR 27.16, LPIPS 0.228, and a size of / MB (unspecified) [S3].

### TRIPS
Attaches a neural feature onto Gaussians [S3]. Renders pyramid-like image feature planes according to the projected Gaussians' size, similar to ADOP, to resolve blur issues [S3].

### Scale-Adaptive Gaussian Splatting
Utilizes an adaptive 2D low-pass filter at test time, adjusted according to the rendering resolution and camera distance [S3].

## Operating characteristics
*   **Time Complexity:** Varies by method; hierarchical methods require tree traversal or LOD selection, while compression methods require decoding [S1, unofficial][S3].
*   **Memory Complexity:** Significantly reduced compared to standard 3DGS, with size reductions up to 15-20x [S1, S2, unofficial][S3].
*   **Performance:** Enables real-time rendering of large-scale scenes (e.g., cityscapes) on consumer-grade GPUs [S1, S2, unofficial].
*   **Strengths:** Scalability, reduced storage, interactive frame rates for large scenes [S1, S2, unofficial][S3].
*   **Limitations:** Potential quality loss due to pruning or quantization, artifacts from opacity quantization [S3].

## Background
3D Gaussian Splatting was introduced as a real-time explicit scene representation [S1, unofficial]. LOD methods were developed to address the memory and scalability limits of monolithic 3DGS, particularly for large-scale urban scenes and autonomous driving simulations [S1, S2, unofficial]. Key developments include Hierarchical 3DGS (Inria, SIGGRAPH'24), LODGE (Google), and various compression techniques like EAGLES and SOGS [S2, unofficial][S3].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
