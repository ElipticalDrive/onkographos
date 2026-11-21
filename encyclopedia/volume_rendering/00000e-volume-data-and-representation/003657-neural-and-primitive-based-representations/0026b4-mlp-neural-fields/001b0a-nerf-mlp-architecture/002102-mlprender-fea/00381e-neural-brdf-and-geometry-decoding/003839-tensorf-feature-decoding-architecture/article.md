# TensoRF Feature Decoding Architecture

Also called: TensoRF Feature Decoding Ablation

## Overview
The TensoRF Feature Decoding Architecture is the neural network component responsible for mapping compressed feature vectors to final radiance values (color) and volume density (sigma) in the TensoRF framework [S1, unofficial]. It solves the problem of efficiently decoding high-dimensional appearance and geometry information from low-rank tensor factors without the computational overhead of large MLPs [S1, unofficial]. It is distinguished from its parent, Neural BRDF and Geometry Decoding, by its specific focus on standard radiance field decoding (color and density) rather than explicit BRDF or normal estimation, and from its siblings by its minimal depth (typically 3 layers) which balances reconstruction quality with training speed [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A feature vector of dimension $mlpC$ (often denoted as $featureC$ in ablation studies, but distinct in the first layer). The input represents the concatenated or projected features from the tensor decomposition [S1, unofficial].
*   **Output:** A vector of size 3 for color (RGB) or size 1 for density (sigma), depending on the specific MLP instance [S1, unofficial].

**Data Structures**
The architecture consists of a sequence of linear layers with ReLU activations. The specific layer dimensions are defined by the parameters $mlpC$ (input feature size) and $featureC$ (hidden feature size).

**Numbered Steps**
1.  **Layer 1:** Apply a linear transformation from input dimension $mlpC$ to hidden dimension $featureC$.
    $$ h_1 = \text{ReLU}(W_1 x + b_1) $$
    where $W_1 \in \mathbb{R}^{featureC \times mlpC}$ and $b_1 \in \mathbb{R}^{featureC}$.
2.  **Layer 2:** Apply a linear transformation from $featureC$ to $featureC$.
    $$ h_2 = \text{ReLU}(W_2 h_1 + b_2) $$
    where $W_2 \in \mathbb{R}^{featureC \times featureC}$ and $b_2 \in \mathbb{R}^{featureC}$.
3.  **Layer 3:** Apply a linear transformation from $featureC$ to the output dimension (3 for color, 1 for sigma).
    $$ y = W_3 h_2 + b_3 $$
    where $W_3 \in \mathbb{R}^{outDim \times featureC}$ and $b_3 \in \mathbb{R}^{outDim}$.

**Parameters**
*   $mlpC$: Input feature dimension. Not given in the sources.
*   $featureC$: Hidden layer dimension. Not given in the sources.
*   Activation: ReLU [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Depth Sensitivity:** Increasing the depth beyond 3 layers (e.g., adding 3 more linear layers) did not yield significant improvements in PSNR [S1, unofficial].
*   **Residual Connections:** Adding residual connections from the initial input or alternate layers did not significantly improve performance compared to the default 3-layer architecture [S1, unofficial].
*   **Convolutional Layers:** Replacing linear layers with 1D convolutional layers reduced trainable parameters and training time but was not the default configuration [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class TensoRFDecoder(nn.Module):
    """
    Default TensoRF Feature Decoding Architecture.
    Consists of 3 linear layers with ReLU activations.
    """
    def __init__(self, mlpC, featureC, out_dim=3):
        super(TensoRFDecoder, self).__init__()
        # Layer 1: Linear(in mlpC, featureC), ReLU
        self.fc1 = nn.Linear(mlpC, featureC)
        self.relu1 = nn.ReLU()
        
        # Layer 2: Linear(featureC, featureC), ReLU
        self.fc2 = nn.Linear(featureC, featureC)
        self.relu2 = nn.ReLU()
        
        # Layer 3: Linear(featureC, out_dim)
        self.fc3 = nn.Linear(featureC, out_dim)

    def forward(self, x):
        # x shape: (batch_size, mlpC)
        h1 = self.relu1(self.fc1(x))
        h2 = self.relu2(self.fc2(h1))
        out = self.fc3(h2)
        return out
```

## Subtopics

### [Deformation Network Architecture](9393-deformation-network-architecture.md)
A lightweight 4-layer MLP used within TensoRF for dynamic scenes, handling temporal displacements. It differs from the standard feature decoder by explicitly modeling time and occlusion weights.

## Operating characteristics
**Time and Memory Complexity**
*   The architecture is shallow (3 layers), resulting in low computational complexity and fast training times (e.g., less than 10 minutes for 2,500 iterations on downsampled data) [S1, unofficial].
*   Memory usage is low due to the small number of parameters compared to deeper MLPs [S1, unofficial].

**Measured Performance**
*   On the synthetic Lego scene (100 train, 200 test views), the default architecture achieved a Train PSNR of 31.11 and Test PSNR of 31.297 after 2,500 iterations on 4x downsampled data [S1, unofficial].
*   This performance is comparable to NeRF (PSNR ~31) but achieved much faster [S1, unofficial].

**Strengths and Limitations**
*   **Strengths:** Fast training, sufficient reconstruction quality, parameter efficiency [S1, unofficial].
*   **Limitations:** Further depth or residual connections do not significantly improve PSNR [S1, unofficial].

**When to Use**
*   Use for standard static scene reconstruction in TensoRF where training speed and memory efficiency are priorities [S1, unofficial].

## Background
The TensoRF Feature Decoding Architecture was introduced in the context of the TensoRF paper, which proposed low-rank tensor decomposition for efficient neural radiance fields [S1, unofficial]. An ablation study was conducted to evaluate the effect of input downsampling and architectural variations (layer removal/addition) on reconstruction quality [S1, unofficial]. The study found that the default 3-layer linear architecture was optimal for the given constraints [S1, unofficial].

## Sources
[S1] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
