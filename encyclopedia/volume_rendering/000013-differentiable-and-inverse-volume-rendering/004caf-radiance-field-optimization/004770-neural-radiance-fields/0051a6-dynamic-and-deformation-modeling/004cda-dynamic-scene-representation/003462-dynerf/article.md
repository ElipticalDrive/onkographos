# DyNeRF

Also called: DyNeRF

## Overview
DyNeRF is a baseline method for dynamic scene representation that models volumetric video using a single NeRF-like Multi-Layer Perceptron (MLP) [S1]. It solves the problem of novel view synthesis for dynamic scenes by conditioning the radiance field prediction on a temporal variable [S1]. Unlike its sibling [Neural Rendering of Variable Scenes](12734-neural-rendering-of-variable-scenes.md) which may use different structural priors, DyNeRF relies on a monolithic shared MLP structure [S1]. This simplicity allows it to serve as a direct baseline for more complex decomposition or deformation-based methods, though it tends to lose high-frequency details due to the limitations of a single shared network [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of images capturing a dynamic scene from various viewpoints at different timestamps.
*   **Output:** A continuous volumetric radiance field function $F_\Theta$ that maps a 3D spatial coordinate $\mathbf{x}$, a viewing direction $\mathbf{d}$, and a time coordinate $t$ to volume density $\sigma$ and view-dependent color $\mathbf{c}$.

**Data Structures:**
*   **MLP Parameters $\Theta$:** A single shared neural network with weights and biases. The specific architecture (number of layers, hidden dimensions) is not explicitly detailed in the provided sources, but it is described as a "NeRF-like MLP" [S1].
*   **Temporal Embedding:** A vector representation of the time coordinate $t$. The specific dimensionality or encoding function (e.g., Fourier features) is not explicitly detailed in the provided sources, but it is concatenated with spatial coordinates [S1].

**Steps:**
1.  **Temporal Embedding Construction:** For a given timestamp $t$, construct a temporal embedding vector $\mathbf{e}_t$ [S1].
2.  **Coordinate Concatenation:** For a sampled point $\mathbf{x}$ along a ray at time $t$, concatenate the spatial coordinate $\mathbf{x}$ with the temporal embedding $\mathbf{e}_t$ [S1].
3.  **Radiance Field Prediction:** Pass the concatenated vector $[\mathbf{x}, \mathbf{e}_t]$ through the shared MLP $F_\Theta$ to predict the volume density $\sigma$ and view-dependent color $\mathbf{c}$ for that specific frame [S1].
4.  **Volume Rendering:** Perform standard volume rendering integration along the ray using the predicted $\sigma$ and $\mathbf{c}$ values to compute the final pixel color [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynerf_forward(x, d, t, mlp_params):
    """
    Predicts radiance field for a dynamic scene point.
    
    Args:
        x: Spatial coordinate (3D vector).
        d: Viewing direction (3D vector).
        t: Time coordinate (scalar or vector).
        mlp_params: Parameters for the shared NeRF-like MLP.
        
    Returns:
        sigma: Volume density.
        c: View-dependent color.
    """
    # 1. Construct temporal embedding
    # Note: Specific embedding function (e.g., positional encoding) is not detailed in sources [S1]
    e_t = temporal_embedding(t) 
    
    # 2. Concatenate spatial coordinate and temporal embedding
    # Note: The source states "takes spatial coordinate and temporal embedding as input" [S1]
    # It does not explicitly mention concatenating viewing direction d in the input to the 
    # main MLP in the provided text, though standard NeRFs do. 
    # Assuming standard NeRF structure where d is also processed or concatenated.
    # However, strictly following [S1]: "takes spatial coordinate and temporal embedding as input"
    input_vector = concatenate(x, e_t)
    
    # 3. Predict radiance field
    # Note: The source implies a single MLP predicts both sigma and c [S1]
    sigma, c = mlp_forward(input_vector, mlp_params)
    
    return sigma, c
```

**Parameters:**
*   **MLP Architecture:** Not given in the sources.
*   **Temporal Embedding Dimension:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **High-Frequency Detail Loss:** The single shared MLP structure tends to lose details in high-frequency regions of the image [S1].
*   **Code Availability:** The original DyNeRF code was not released, requiring re-implementation for comparison [S1].

## Subtopics

### [Decomposed Spatiotemporal Representation](7525-decomposed-spatiotemporal-representation.md)
Decomposed Spatiotemporal Representation is a method for modeling dynamic volumetric scenes by categorizing spatial regions into static, deforming, and new content, assigning each category a specialized neural field to optimize temporal consistency and computational efficiency. It sets itself apart from the parent DyNeRF method by replacing a single monolithic MLP with a mixture of specialized fields (stationary, deformation, and newness) weighted by a learned decomposition probability, rather than relying on a single temporal embedding.

### FDRF
FDRF is a method that uses time-aware voxel features to represent scenes and simulates rough motion trajectories with a small coordinate deformation network [S2]. It enhances time information in the radiation network and significantly accelerates the optimization of dynamic radiation fields while maintaining high rendering quality [S2].

## Operating characteristics
*   **Performance:** In quantitative comparisons, DyNeRF achieved a PSNR of 30.87, SSIM of 0.943, and LPIPS of 0.118 on the evaluated dataset [S1].
*   **Rendering Time:** DyNeRF had a rendering time of 5195 milliseconds [S1].
*   **Storage:** DyNeRF required 12 MB of storage [S1].
*   **Strengths:** Serves as a simple baseline for dynamic scene representation [S1].
*   **Limitations:** Tends to lose details in high-frequency regions due to its single shared MLP structure [S1]. Rendering time is significantly higher than more optimized variants like the proposed method in [S1] (33 ms).

## Background
DyNeRF was introduced as a baseline method for representing volumetric videos with a NeRF-like MLP [S1]. It was compared against other methods such as Neural Volumes, C-NeRF, and D-NeRF [S1]. The original code for DyNeRF was not released, necessitating re-implementation for comparative studies [S1].

## Sources
[S1] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
