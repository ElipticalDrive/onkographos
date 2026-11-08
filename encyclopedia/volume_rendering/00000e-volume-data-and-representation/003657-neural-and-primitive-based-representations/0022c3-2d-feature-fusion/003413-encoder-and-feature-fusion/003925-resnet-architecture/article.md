# ResNet Architecture

Also called: 2D residual block, 3D residual block, ResNet, Residual Block

## Overview
The ResNet architecture serves as the basic building block for residual connections in 3D and 2D convolutions, primarily used to mitigate the vanishing gradient problem in deep networks [S1, unofficial]. It sets itself apart from standard convolutional layers by introducing shortcut connections that add the input directly to the output, allowing the network to learn residual functions [S1, unofficial]. In the context of volume rendering and feature fusion, it acts as a core component for encoding 3D volumetric data or processing 2D feature maps, often appearing in encoders like RenderNet [S1, unofficial].

## How it works
The method defines two primary variants: the 3D Residual Block for volumetric data and the 2D Residual Block for planar feature maps. The core logic involves applying convolutions, activations, and a skip connection.

**Inputs and Outputs:**
- Input: A tensor $X$ with shape $(C_{in}, D, H, W)$ for 3D or $(C_{in}, H, W)$ for 2D.
- Output: A tensor $Y$ with shape $(C_{out}, D', H', W')$ for 3D or $(C_{out}, H', W')$ for 2D.

**Data Structures and Layout:**
1.  **3D Residual Block:**
    -   Layer 1: 3D Convolution, kernel size $3 \times 3 \times 3$, stride 1, padding 1.
    -   Activation: Parametric ReLU (PReLU) [S1, unofficial].
    -   Layer 2: 3D Convolution, kernel size $3 \times 3 \times 3$, stride 1, padding 1.
    -   Dropout: Probability 0.5 applied after every convolution *except* those inside the residual block [S1, unofficial].
    -   Shortcut: Element-wise addition of Input $X$ and Output of Layer 2.
2.  **2D Residual Block:**
    -   Identical structure to the 3D block but uses 2D convolutions ($3 \times 3$ kernels) [S1, unofficial].

**Numbered Steps:**
1.  Pass input $X$ through the first convolutional layer $Conv_1$.
2.  Apply PReLU activation: $A_1 = \text{PReLU}(Conv_1(X))$.
3.  Pass $A_1$ through the second convolutional layer $Conv_2$.
4.  Apply PReLU activation: $A_2 = \text{PReLU}(Conv_2(A_1))$.
5.  Add the original input $X$ to $A_2$ via the shortcut connection: $Y = A_2 + X$.
    -   *Note:* If channel dimensions differ, $X$ must be projected via a $1 \times 1$ convolution to match $A_2$'s channels [G].
6.  Apply a final activation (often ReLU or Sigmoid depending on the network depth; the last layer of the entire network uses Sigmoid [S1, unofficial]).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class ResidualBlock3D(nn.Module):
    def __init__(self, in_channels, out_channels):
        super(ResidualBlock3D, self).__init__()
        # 3D Convolution layers
        self.conv1 = nn.Conv3d(in_channels, out_channels, kernel_size=3, stride=1, padding=1)
        self.conv2 = nn.Conv3d(out_channels, out_channels, kernel_size=3, stride=1, padding=1)
        # Parametric ReLU
        self.prelu = nn.PReLU()
        # Shortcut projection if channels change
        if in_channels != out_channels:
            self.shortcut = nn.Conv3d(in_channels, out_channels, kernel_size=1, stride=1, padding=0)
        else:
            self.shortcut = nn.Identity()

    def forward(self, x):
        residual = self.shortcut(x)
        out = self.conv1(x)
        out = self.prelu(out)
        out = self.conv2(out)
        out = self.prelu(out)
        out += residual
        return out

class ResidualBlock2D(nn.Module):
    def __init__(self, in_channels, out_channels):
        super(ResidualBlock2D, self).__init__()
        # 2D Convolution layers
        self.conv1 = nn.Conv2d(in_channels, out_channels, kernel_size=3, stride=1, padding=1)
        self.conv2 = nn.Conv2d(out_channels, out_channels, kernel_size=3, stride=1, padding=1)
        # Parametric ReLU
        self.prelu = nn.PReLU()
        # Shortcut projection if channels change
        if in_channels != out_channels:
            self.shortcut = nn.Conv2d(in_channels, out_channels, kernel_size=1, stride=1, padding=0)
        else:
            self.shortcut = nn.Identity()

    def forward(self, x):
        residual = self.shortcut(x)
        out = self.conv1(x)
        out = self.prelu(out)
        out = self.conv2(out)
        out = self.prelu(out)
        out += residual
        return out
```

**Parameters:**
-   Kernel Size: $3 \times 3 \times 3$ (3D) or $3 \times 3$ (2D) [S1, unofficial].
-   Dropout Probability: 0.5 (applied after convolutions outside residual blocks) [S1, unofficial].
-   Activation: PReLU inside blocks, Sigmoid in the final output layer of the network [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Dimension Mismatch:** If the number of input and output channels differs, the shortcut connection must include a $1 \times 1$ convolution to match dimensions before addition [G].
-   **Boundary Conditions:** Padding must be set to 1 for $3 \times 3 \times 3$ kernels to maintain spatial dimensions if stride is 1 [G].

## Subtopics

### ResNet_Grad Architecture
A variant used in inverse problem reconstruction for CT scans. It consists of a sequential model with 4 layers. Each layer comprises a GradBlock (taking operator A) followed by a ResBlock with `in_channels=2` and `out_channels=1` [S2, unofficial]. The forward pass involves passing the input $x$ through `self.model` to get a reconstruction $f_{recon}$, and then returning $f_{recon}$ and `self.head(f_recon)` [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Linear with respect to the number of layers and spatial dimensions, similar to standard convolutions, but with a slight overhead for the addition operation.
-   **Memory Complexity:** Requires storing intermediate activations for backpropagation. The shortcut connection does not significantly increase memory usage.
-   **Strengths:** Enables training of very deep networks by facilitating gradient flow. PReLU allows for adaptive learning of the activation function's negative slope [S1, unofficial].
-   **Limitations:** The use of PReLU and dropout increases parameter count and training complexity compared to standard ReLU networks. The specific architecture described (RenderNet) is tailored for differentiable rendering and may not be optimal for general-purpose image classification without modification [S1, unofficial].
-   **When to Use:** When building deep encoders for 3D volumetric data or 2D feature maps where gradient vanishing is a concern. Specifically useful in differentiable rendering pipelines like RenderNet [S1, unofficial] or iterative reconstruction networks [S2, unofficial].

## Background
The ResNet architecture was introduced as a fundamental component in deep learning for computer vision and volumetric analysis. In the context of volume rendering, it was prominently featured in **RenderNet**, a deep convolutional network for differentiable rendering from 3D shapes [S1, unofficial]. RenderNet utilized 3D residual blocks for the volumetric encoder and 2D residual blocks for the feature fusion and projection stages. The use of PReLU and dropout was specified to stabilize training in this specific application [S1, unofficial]. Another application is found in inverse problem solvers for CT scans, where a variant called ResNet_Grad is used to integrate gradient information into the reconstruction process [S2, unofficial].

## Sources
[S1] RenderNet: A deep convolutional network for differentiable rendering from 3D shapes, https://doi.org/10.48550/arxiv.1806.06575, fetched 2026-10-26 (unofficial)
[S2] imanfs/inverse-problems-CT-scans - GitHub, https://github.com/imanfs/inverse-problems-CT-scans, fetched 2026-10-27 (unofficial)
