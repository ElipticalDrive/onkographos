# Multi-input MLP (mi-MLP)
Also called: mi-MLP, per-layer inputs incorporation

## Overview
The multi-input MLP (mi-MLP) is a network architecture modification for Neural Radiance Fields (NeRF) designed to prevent overfitting in few-shot view synthesis without sacrificing the capacity to synthesize high-frequency details [S1]. Unlike the standard NeRF architecture [Parent](11313-model-architecture-modifications.md), which relies on a single deep path from input embeddings to output predictions, mi-MLP incorporates the positional encoding of the 3D location and viewing direction into every layer of the MLP [S1]. This creates shorter paths between inputs and outputs, allowing the model to learn flexible mappings that alleviate overfitting while maintaining the original model capacity [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 5D coordinate vector $\mathbf{x} \in \mathbb{R}^5$, composed of the 3D spatial position $\mathbf{p} \in \mathbb{R}^3$ and the 2D viewing direction $\mathbf{d} \in \mathbb{R}^2$ (normalized). Note: Standard NeRF uses $\mathbf{d} \in \mathbb{R}^3$, but the source formulation for mi-MLP explicitly references 2D viewing direction in the definition of inputs, though the positional encoding $\gamma_L$ is applied to the combined input $\mathbf{x}$ [S1].
*   **Output:** Volume density $\sigma \in \mathbb{R}$ and view-dependent color $\mathbf{c} \in \mathbb{R}^3$.

**Data Structures:**
The network consists of $N$ layers. Each layer $i$ (for $i > 1$) contains:
*   Weight matrix components $\mathbf{w}_i^{j0} \in \mathbb{R}^{1 \times d_1}$ connecting the input embedding to the $j$-th neuron.
*   Weight matrix components $\mathbf{w}_i^{j1} \in \mathbb{R}^{1 \times d_2}$ connecting the previous layer's output to the $j$-th neuron.
*   Bias vector $\mathbf{b}_i \in \mathbb{R}^{d_2 \times 1}$.
*   Activation function $\epsilon$ (default: ReLU) [S1].

**Parameters:**
*   $L$: Hyperparameter controlling the number of frequencies in the positional encoding.
*   $d_1$: Dimension of the encoded input embedding $\gamma_L(\mathbf{x})$.
*   $d_2$: Hidden dimension of the MLP layers.
*   Weight Initialization: Default PyTorch initialization is used, which satisfies the condition $\|\mathbf{w}_i^{j1}\|_1 \in (0, 1]$ [S1].

**Positional Encoding:**
The input $\mathbf{x}$ is transformed using positional encoding $\gamma_L(\mathbf{x})$ [S1]:
$$
\gamma_L(\mathbf{x}) = (\sin(2^0 \mathbf{x}), \cos(2^0 \mathbf{x}), \dots, \sin(2^{L-1} \mathbf{x}), \cos(2^{L-1} \mathbf{x}))
$$
This results in an embedding vector of size $2 \cdot L \cdot \dim(\mathbf{x})$ [S1].

**Algorithm:**
The output of layer $i$, denoted as $f_i$, is computed by applying the layer function $\phi_i$ to the previous layer's output $f_{i-1}$ and the positional encoding of the input $\gamma_L(\mathbf{x})$ [S1].

1.  Compute the positional encoding of the input:
    $$ \mathbf{e} = \gamma_L(\mathbf{x}) $$
2.  Initialize the first layer output $f_1$:
    $$ f_1 = \phi_1(\mathbf{e}) $$
    Where $\phi_1$ is the first layer function.
3.  For each subsequent layer $i$ from 2 to $N$:
    $$ f_i = \phi_i(f_{i-1}, \mathbf{e}) $$
    The $j$-th element of the output feature $f_i$, denoted $\phi_i^j$, is calculated as [S1]:
    $$ \phi_i^j(\mathbf{e}) = \epsilon\left( \mathbf{w}_i^{j0} \mathbf{e} + \mathbf{w}_i^{j1} f_{i-1} + \mathbf{b}_i \right) $$
    Where:
    *   $\epsilon$ is the activation function (default ReLU).
    *   $\mathbf{w}_i^{j0} \in \mathbb{R}^{1 \times d_1}$ are the weights connecting the input embedding to the neuron.
    *   $\mathbf{w}_i^{j1} \in \mathbb{R}^{1 \times d_2}$ are the weights connecting the previous layer's output to the neuron.
    *   $\mathbf{b}_i \in \mathbb{R}^{1}$ is the bias for the neuron.
4.  The final layer outputs $\sigma$ and $\mathbf{c}$ are derived from $f_N$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

class PositionalEncoding(nn.Module):
    def __init__(self, L):
        super(PositionalEncoding, self).__init__()
        self.L = L

    def forward(self, x):
        # x shape: (..., D)
        # Output shape: (..., 2 * L * D)
        # Note: The source defines gamma_L(x) as sin/cos of 2^k * x
        # We assume x is already scaled or the frequencies are applied directly.
        # Standard NeRF uses frequencies 2^0 to 2^(L-1).
        freq_bands = 2.0 ** torch.arange(self.L, device=x.device).float()
        # Expand dimensions for broadcasting
        # x: (..., D) -> (..., 1, D)
        # freq_bands: (L,) -> (L, 1, 1)
        scaled_x = x.unsqueeze(1) * freq_bands.unsqueeze(0).unsqueeze(-1)
        # Apply sin and cos
        # scaled_x: (..., L, D)
        sin_vals = torch.sin(scaled_x)
        cos_vals = torch.cos(scaled_x)
        # Concatenate along the last dimension
        # Result: (..., L * D * 2)
        out = torch.cat([sin_vals, cos_vals], dim=-1)
        return out

class MiMLP(nn.Module):
    def __init__(self, input_dim, hidden_dim, num_layers, L):
        super(MiMLP, self).__init__()
        self.input_dim = input_dim
        self.hidden_dim = hidden_dim
        self.num_layers = num_layers
        self.L = L
        
        # Input dimension for positional encoding is 5 (3 pos + 2 view dir as per source text)
        # However, standard NeRF uses 3 for view dir. The source text says "2D viewing direction".
        # We follow the source text's explicit mention of 2D for the input definition, 
        # but note that standard NeRF uses 3D. If 3D is required, adjust input_dim.
        # The source formula uses gamma_L(x) where x is 5D.
        
        self.pos_enc = PositionalEncoding(L)
        
        # Calculate encoded input dimension
        # x is 5D. gamma_L produces 2 * L * 5 dimensions.
        self.encoded_input_dim = 2 * L * input_dim
        
        # Layer 1: Takes only encoded input
        self.layer1 = nn.Linear(self.encoded_input_dim, hidden_dim)
        
        # Layers 2 to N: Take previous output AND encoded input
        self.layers = nn.ModuleList([
            nn.Linear(hidden_dim + self.encoded_input_dim, hidden_dim)
            for _ in range(num_layers - 1)
        ])
        
        # Output layer
        self.output_layer = nn.Linear(hidden_dim, 4) # 1 for sigma, 3 for color

    def forward(self, x):
        # x: (..., 5)
        # 1. Encode inputs
        e = self.pos_enc(x) # (..., 2 * L * 5)
        
        # 2. First layer
        h = self.layer1(e)
        h = torch.relu(h)
        
        # 3. Subsequent layers
        for i in range(len(self.layers)):
            # Concatenate previous hidden state with encoded input
            # h: (..., hidden_dim)
            # e: (..., encoded_input_dim)
            combined = torch.cat([h, e], dim=-1)
            h = self.layers[i](combined)
            h = torch.relu(h)
            
        # 4. Output
        out = self.output_layer(h)
        sigma = out[..., 0:1]
        color = out[..., 1:4]
        return sigma, color
```

**Edge Cases and Pitfalls:**
*   **Gradient Amplitude:** During early training, vanilla MLPs result in similar gradient amplitudes for all layers. mi-MLP ensures that deeper layers (close to outputs) are updated with large gradients while shallower layers are updated with extremely small ones [S1]. This behavior is theoretically supported by the ratio of gradient amplitudes between adjacent layers [S1].
*   **Weight Initialization:** The condition $\|\mathbf{w}_i^{j1}\|_1 \in (0, 1]$ is critical for the gradient ratio property. Default PyTorch initialization satisfies this [S1].
*   **Input Dimension:** The source explicitly states "2D viewing direction" in the definition of inputs for mi-MLP, which differs from the standard 3D viewing direction in vanilla NeRF. Implementers should verify if the specific dataset requires 2D or 3D direction encoding [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The mi-MLP boosts the PSNR of the baseline from 14.73 to 24.23 on few-shot view synthesis benchmarks [S1].
*   **Strengths:**
    *   Prevents overfitting in few-shot view synthesis without reducing model capacity [S1].
    *   Allows synthesis with fewer parameters in an end-to-end way due to shorter paths between inputs and outputs [S1].
    *   Keeps inputs and outputs unchanged, making it a plug-and-play solution for current NeRF-based pipelines [S1].
*   **Limitations:**
    *   Rendered novel views may still contain noticeable artifacts if colors and volume density are not modeled separately [S1].
*   **When to use:** When performing few-shot view synthesis and experiencing overfitting with a standard vanilla MLP architecture [S1].

## Background
The multi-input MLP (mi-MLP) was proposed by [S1] in 2024. It addresses the overfitting issue in few-shot view synthesis by modifying the network structure to incorporate inputs into each layer. The method is orthogonal to parameter reduction techniques, as it maintains model capacity while improving generalization [S1].

## Sources
[S1] Is Vanilla MLP in Neural Radiance Field Enough for Few-Shot View Synthesis?, https://doi.org/10.1109/cvpr52733.2024.01918, fetched 2026-10-26
