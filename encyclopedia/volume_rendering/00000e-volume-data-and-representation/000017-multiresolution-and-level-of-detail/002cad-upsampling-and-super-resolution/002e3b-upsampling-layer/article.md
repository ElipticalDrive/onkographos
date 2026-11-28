# Upsampling Layer

Also called: Decoder Upsampling, Spatial Upsampling Layer

## Overview
The Upsampling Layer expands the spatial dimensions of an input volume to reconstruct the original resolution within decoder networks [S1, unofficial]. It operates without trainable parameters, serving as the structural inverse to encoder pooling layers [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume $U$ with spatial dimensions $x \times y$ and channel depth $p$ [S1, unofficial].
*   **Output:** A volume $V$ with spatial dimensions $(s_x x) \times (s_y y)$ and channel depth $p$ [S1, unofficial].

**Data Structures:**
The input and output are represented as dense multi-dimensional arrays or tensors [G].

**Numbered Steps:**
1. Allocate an output buffer $V$ with dimensions $(s_x x, s_y y, p)$ [G].
2. Iterate over each channel index $k$ from $0$ to $p-1$ [G].
3. For each channel, iterate over input coordinates $(i, j)$ where $0 \le i < x$ and $0 \le j < y$ [G].
4. Map the value $U[i, j, k]$ to position $(s_x \cdot i, s_y \cdot j, k)$ in $V$ [S1, unofficial].
5. Fill all unmapped positions in $V$ using implementation-specific conventions (e.g., zero-filling or replication) [G].

**Formulas:**
*   Output dimensions: $x' = s_x \cdot x$, $y' = s_y \cdot y$ [S1, unofficial].
*   Mapping: $V[s_x \cdot i, s_y \cdot j, k] = U[i, j, k]$ for $i \in [0, x), j \in [0, y), k \in [0, p)$ [S1, unofficial].

**Parameters:**
*   $s_x, s_y$: Scaling factors for the x and y dimensions (typically $s_x=2, s_y=2$) [S1, unofficial].
*   Filter Size: $2 \times 2$ [S1, unofficial].
*   Stride: $2$ [S1, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def upsample_layer(U, sx, sy):
    """
    Upsamples volume U by factors sx and sy.
    U: list of lists of lists representing shape (x, y, p)
    Returns: V of shape (sx*x, sy*y, p)
    """
    x = len(U)
    y = len(U[0])
    p = len(U[0][0])
    
    # Initialize output buffer with zeros [G]
    V = [[[0.0] * p for _ in range(sy * y)] for _ in range(sx * x)]
    
    for i in range(x):
        for j in range(y):
            for k in range(p):
                V[i * sx][j * sy][k] = U[i][j][k]
                
    return V
```

## Subtopics

### Speed-up Module
A lightweight convolutional layer using a $1 \times 1$ kernel to modify feature map channels, aligning low-dimensional rendered semantic features with high-dimensional ground truth 2D feature maps [S2]. It upsamples rendered features to the ground-truth dimension after an initial lower-dimensional rendering step [S2]. Unlike complex CNNs, it relies on the ground truth feature map from a teacher network as a target, simplifying feature alignment [S2]. In practice, it is applied to rendered feature maps with a dimension of 128 to align with higher-dimensional targets (e.g., 256 or 512) [S2]. It is typically optimized with a separate Adam optimizer (learning rate $1e-4$) [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [G].
*   **Memory Complexity:** Not given in the sources [G].
*   **Strengths:** Not given in the sources [G].
*   **Limitations:** Not given in the sources [G].
*   **When to use:** Not given in the sources [G].

## Background
The Upsampling Layer was introduced in the context of compressed volume rendering using deep learning, specifically within autoencoder architectures that downsample input data in the encoder and reconstruct it in the decoder [S1, unofficial]. It is applied in decoder layers to restore the original spatial resolution from compressed latent representations [S1, unofficial].

## Sources
[S1] Compressed volume rendering using deep learning, https://www.cs.umd.edu/sites/default/files/scholarly_papers/Jain.pdf, fetched 2026-10-26 (unofficial)
[S2] Feature 3DGS: Supercharging 3D Gaussian Splatting to Enable Distilled Feature Fields, https://doi.org/10.1109/cvpr52733.2024.02048, fetched 2026-10-26
