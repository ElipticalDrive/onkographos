# 1D Convolutional MLP

Also called: Point-wise prediction network

## Overview
The 1D Convolutional MLP is a neural network module that stacks 1D convolutional layers with kernel size 1 to process feature vectors, optionally employing residual connections and batch normalization. It is used in volume rendering to predict volume values (such as density or color) from concatenated view-dependent features and interpolated spatial features [S1, unofficial]. It differs from standard fully connected MLPs by explicitly structuring the transformation as a sequence of 1D convolutions, which allows for consistent channel-wise mixing, and from sibling methods like CoDeF by relying on this specific residual 1D convolutional architecture rather than coordinate-based deformation fields or multi-head decoders [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: A tensor $x$ of shape $(N, C_{in}, L)$, where $N$ is the batch size, $C_{in}$ is the number of input channels (features), and $L$ is the sequence length (typically 1 for point-wise prediction, but the architecture supports $L > 1$). In the context of volume rendering, this input is the concatenation of view direction embeddings and interpolated spatial features [S1, unofficial].
*   **Output**: A tensor of shape $(N, C_{out}, L)$, where $C_{out}$ is the final channel dimension specified in the configuration.

**Data Structures**
The module is defined by a configuration dictionary or object containing:
*   `mlp_list` (or `channels`): A list of integers $[C_0, C_1, \dots, C_K]$ specifying the channel dimensions for each layer. $C_0$ is the input channel count.
*   `residual` (Boolean): If `True`, enables residual connections for layers $i > 0$.
*   `use_bn` (Boolean): If `True`, applies Batch Normalization to intermediate layers.
*   `last_bn` (Boolean): If `True`, applies Batch Normalization to the final layer.
*   `last_act` (Boolean): If `True`, applies LeakyReLU to the final layer.

**Numbered Steps**
1.  **Initialization**: Create a list of modules `mlps`. Iterate through the channel list indices $i$ from $0$ to $K-1$.
2.  **Layer Construction**: For each layer $i$:
    *   Determine input channels: If $i=0$ or `residual` is `False`, input channels are $C_i$. If $i>0$ and `residual` is `True`, input channels are $C_i + C_0$ (concatenating the original input $x$ with the previous output).
    *   Determine output channels: $C_{i+1}$.
    *   Create a `Conv1d` layer with kernel size 1, input channels as determined, and output channels $C_{i+1}$.
    *   If $i < K-1$ (not the final layer):
        *   If `use_bn` is `True`, append `BatchNorm1d(C_{i+1})`.
        *   Append `LeakyReLU(inplace=True)`.
    *   If $i = K-1$ (final layer):
        *   If `last_bn` is `True`, append `BatchNorm1d(C_{i+1})`.
        *   If `last_act` is `True`, append `LeakyReLU(inplace=True)`.
    *   Wrap the sequence of layers for index $i$ in an `nn.Sequential` container and add to `mlps`.
3.  **Forward Pass**:
    *   Let $x_{curr} = x$.
    *   For each layer module $m$ in `mlps` with index $i$:
        *   If $i > 0$ and `residual` is `True`:
            *   Concatenate the original input $x$ with $x_{curr}$ along the channel dimension: $x_{cat} = \text{concat}(x_{curr}, x)$.
            *   Set $x_{curr} = x_{cat}$.
        *   Pass $x_{curr}$ through the module $m$: $x_{curr} = m(x_{curr})$.
    *   Return $x_{curr}$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class Conv1dMLP(nn.Module):
    def __init__(self, channels, residual=True, use_bn=True, last_bn=False, last_act=False):
        """
        :param channels: List of integers [C0, C1, ..., CK]
        :param residual: Boolean, use residual connections for i > 0
        :param use_bn: Boolean, use BatchNorm1d on intermediate layers
        :param last_bn: Boolean, use BatchNorm1d on the final layer
        :param last_act: Boolean, use LeakyReLU on the final layer
        """
        super().__init__()
        self.residual = residual
        self.mlps = nn.ModuleList()
        
        for i in range(len(channels) - 1):
            modules = []
            in_ch = channels[i]
            out_ch = channels[i + 1]
            
            # Residual connection logic for input channels
            if i > 0 and self.residual:
                in_ch = channels[i] + channels[0]
            
            # Conv1d with kernel size 1
            modules.append(nn.Conv1d(in_ch, out_ch, kernel_size=1))
            
            # BatchNorm and Activation logic
            is_last_layer = (i == len(channels) - 2)
            
            if not is_last_layer:
                if use_bn:
                    modules.append(nn.BatchNorm1d(out_ch))
                modules.append(nn.LeakyReLU(inplace=True))
            else:
                # Final layer specific flags
                if last_bn:
                    modules.append(nn.BatchNorm1d(out_ch))
                if last_act:
                    modules.append(nn.LeakyReLU(inplace=True))
            
            self.mlps.append(nn.Sequential(*modules))

    def forward(self, x):
        """
        :param x: Tensor of shape (N, C_in, L)
        :return: Tensor of shape (N, C_out, L)
        """
        x_curr = x
        for i, m in enumerate(self.mlps):
            if i > 0 and self.residual:
                # Concatenate original input x with current output along channel dim
                x_curr = torch.cat([x_curr, x], dim=1)
            x_curr = m(x_curr)
        return x_curr
```

**Parameters with Typical Values**
*   `channels`: Varies by model depth. Common configurations might be `[64, 128, 256, 128]` or similar increasing-then-decreasing structures. Not given in the sources.
*   `residual`: `True` is common to facilitate gradient flow in deeper networks.
*   `use_bn`: `True` is common to stabilize training.
*   `last_bn`, `last_act`: Often `False` for the final layer if followed by a specific activation or loss function, but configurable.

**Edge Cases and Pitfalls**
*   **Channel Mismatch**: If `residual` is `True`, the input tensor $x$ must have the same number of channels as $C_0$ (the first element of `channels`). If the input feature dimension does not match $C_0$, the concatenation will fail. [G]
*   **Sequence Length**: While designed for point-wise prediction ($L=1$), the 1D convolution operates along $L$. If $L > 1$, the kernel size 1 ensures no spatial mixing across the sequence dimension, treating each position independently.
*   **Precision**: Standard float32 is sufficient. No specific precision issues noted.

## Operating characteristics
*   **Time Complexity**: $O(N \cdot L \cdot \sum_{i} C_i \cdot C_{i+1})$, where $N$ is batch size, $L$ is sequence length, and $C_i$ are channel dimensions. Since kernel size is 1, it is effectively a series of linear transformations per position.
*   **Memory Complexity**: $O(\sum_{i} C_i \cdot C_{i+1})$ for weights. Activation memory is $O(N \cdot L \cdot \max(C_i))$.
*   **Strengths**: Simple to implement, efficient for point-wise feature transformation, residual connections help train deeper networks.
*   **Limitations**: Lacks spatial context mixing (kernel size 1), so it cannot capture local correlations in the feature dimension if $L > 1$. Not suitable for tasks requiring spatial smoothing.
*   **When to use**: When predicting per-point volume attributes (density, color) from pre-computed features.
*   **When not to use**: When spatial coherence in the feature space is required, or when input is not point-wise.

## Background
The 1D Convolutional MLP is used in the DIF-Gaussian framework for volume rendering [S1, unofficial]. It serves as a component for predicting volume values from features. The specific implementation with residual connections and 1D convolutions is derived from the codebase of DIF-Gaussian.

## Sources
[S1] DIF-Gaussian, https://github.com/xmed-lab/DIF-Gaussian, fetched 2026-10-27 (unofficial)
