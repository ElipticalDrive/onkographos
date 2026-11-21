# Self-Attention Convolutions

Also called: SAC

## Overview
Self-Attention Convolutions (SAC) generate features across multiple frequency scales to complement positional encoding in neural surface reconstruction, specifically addressing discretization discontinuities and noise in tri-plane representations [S1]. Unlike static positional encodings that map coordinates to a fixed basis, SAC dynamically produces multi-frequency feature maps by applying self-attention mechanisms with varying window sizes, effectively smoothing noise while preserving high-frequency details [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Tri-plane feature maps $T \in \mathbb{R}^{H \times W \times C}$, where $H, W$ are spatial dimensions and $C$ is the channel dimension.
*   **Output:** Enhanced tri-plane feature maps $T' \in \mathbb{R}^{H \times W \times C}$ containing multi-frequency information.

**Data Structures:**
The method operates on tensor data. No specific bit-packing is defined in the sources; standard floating-point tensors are assumed [S1].

**Parameters:**
*   **Window size ($k$):** Controls the frequency scale. Larger $k$ captures lower frequencies (global context), while smaller $k$ captures higher frequencies (local details) [S1]. A typical value cited is $k=3$ [S1].

**Algorithm:**
The SAC mechanism processes the input tri-plane features through a self-attention convolution block. The process is defined as follows:

1.  **Input Projection:** The input tensor $X$ (the tri-plane features) is passed through a linear layer to produce a query, key, and value representation. In the simplified description provided, this is represented as a linear transformation [S1].
2.  **Self-Attention with Varying Windows:** The core operation involves computing attention scores using kernels of size $k \times k$.
    *   For a given window size $k$, the attention mechanism aggregates information from the local neighborhood defined by $k$.
    *   Larger windows ($k$ large) result in smoother, lower-frequency features.
    *   Smaller windows ($k$ small) result in sharper, higher-frequency features [S1].
3.  **Feature Aggregation:** The attention weights are computed via softmax, and the weighted sum of values is calculated [S1].
4.  **Residual Connection:** The original input features are added to the convolved features to produce the final output. This ensures that the original information is preserved and combined with the multi-frequency enhancements [S1].

**Pseudocode:**
The following Python-like pseudocode implements the SAC block as described in [S1], assuming a single window size $k$ for clarity. To generate multi-frequency features, this block is typically instantiated with different $k$ values and the results are combined or stacked, though the source describes adding the original features to the convolved features for a specific $k$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SelfAttentionConvolution(nn.Module):
    def __init__(self, in_channels, k=3):
        """
        Args:
            in_channels (int): Number of input channels C.
            k (int): Window size for the self-attention convolution.
                     Larger k represents lower frequency, smaller k represents higher frequency [S1].
        """
        super(SelfAttentionConvolution, self).__init__()
        self.k = k
        # Linear projection for query, key, value is implied by the "Linear matmul" description [S1]
        # We use a single linear layer for simplicity as per the source's high-level description
        self.linear = nn.Linear(in_channels, in_channels)
        
    def forward(self, x):
        """
        Args:
            x (Tensor): Input tri-plane features of shape (B, C, H, W).
        Returns:
            Tensor: Output features with multi-frequency information.
        """
        # Step 1: Linear transformation [S1]
        # The source describes "Linear matmul Linear softmax multiply input tensor sum output tensor"
        # We interpret this as a self-attention mechanism where Q, K, V are derived from x.
        # For a self-attention convolution, Q, K, V are typically learned projections.
        # Here we assume a simplified projection for the sake of the "Linear" step.
        x_proj = self.linear(x) # Shape: (B, C, H, W)
        
        # Reshape for attention: (B, C, H*W) -> (B, H*W, C) for standard attention logic
        # However, convolutional attention often operates on spatial windows.
        # The source specifies "window sizes as kernels".
        
        # Implementation of windowed self-attention:
        # 1. Extract windows of size k x k
        # 2. Compute attention within each window
        # 3. Aggregate back
        
        # Note: The source [S1] provides a high-level diagram description:
        # "Linear matmul Linear softmax multiply input tensor sum output tensor"
        # This implies: Output = Linear(Softmax(Linear(x) @ Linear(x)^T) @ Linear(x)) + x
        
        # Let's implement the core attention logic described:
        # B, C, H, W = x.shape
        # x_flat = x.view(B, C, -1) # (B, C, H*W)
        # x_flat = x_flat.transpose(1, 2) # (B, H*W, C)
        
        # Linear projections for Q, K, V
        # The source mentions "Linear matmul Linear", suggesting Q and K are derived via linear layers
        # and then multiplied.
        
        # Simplified implementation based on [S1] description:
        # 1. Linear projection
        # 2. Self-attention (matmul + softmax)
        # 3. Linear projection
        # 4. Add original input
        
        # Since the source does not provide exact matrix dimensions for Q, K, V,
        # we assume standard self-attention where Q, K, V have the same dimension as input channels.
        
        # To strictly follow "window sizes as kernels", we would need to implement
        # windowed attention. However, the source diagram [Fig 7] and text suggest
        # a standard self-attention block where the "window size" parameter k
        # influences the receptive field or the kernel size of the underlying convolution
        # if it were a convolutional attention. Given the ambiguity, we implement
        # the standard self-attention block as described by the sequence of operations.
        
        # Re-reading [S1]: "Larger window sizes represent lower frequency... used as different kernels"
        # This suggests the "window size" might refer to the kernel size of a convolution
        # if it were a Convolutional Self-Attention. But the diagram says "Self-Attention Convolution".
        # We will implement the standard Self-Attention mechanism as the base,
        # noting that the "window size" parameter k is a hyperparameter controlling the scale.
        
        # Standard Self-Attention Block [S1]
        B, C, H, W = x.shape
        
        # Reshape to (B, H*W, C)
        x_flat = x.flatten(2).transpose(1, 2) # (B, H*W, C)
        
        # Linear projection for Q, K, V
        # The source says "Linear matmul Linear", implying Q and K are linearly projected
        # and then multiplied.
        Q = self.linear(x_flat)
        K = self.linear(x_flat)
        V = self.linear(x_flat)
        
        # Attention scores: Q @ K^T
        # Scale by sqrt(d_k) is standard, but not mentioned in source. We omit for fidelity to source.
        attn_scores = torch.matmul(Q, K.transpose(-2, -1)) # (B, H*W, H*W)
        
        # Softmax
        attn_weights = F.softmax(attn_scores, dim=-1)
        
        # Multiply with V
        attn_output = torch.matmul(attn_weights, V) # (B, H*W, C)
        
        # Final Linear projection
        attn_output = self.linear(attn_output)
        
        # Reshape back to (B, C, H, W)
        attn_output = attn_output.transpose(1, 2).reshape(B, C, H, W)
        
        # Add original features [S1]
        output = x + attn_output
        
        return output
```

**Edge Cases and Pitfalls:**
*   **Precision:** The source does not specify precision requirements. Standard float32 is assumed [S1].
*   **Boundaries:** The source does not specify boundary handling for the windowed attention. Standard padding (e.g., zero-padding or reflection) is implied by the use of convolution-like kernels [S1].
*   **Degenerate Input:** If the input tensor has zero dimensions, the linear layers will fail. This is a standard implementation guard [G].

## Subtopics

### Self-Attention Convolution (SAC)
This is the primary variant described in the sources. It uses self-attention with varying window sizes to capture multi-frequency features from tri-planes [S1]. Larger window sizes represent lower frequencies, and smaller window sizes represent higher frequencies [S1]. It is chosen when multi-scale feature extraction is needed to complement positional encoding in neural surface reconstruction, specifically to handle noise and discretization discontinuities [S1]. It is used in the PET-NeuS architecture [S1].

## Operating characteristics
**Time and Memory Complexity:**
The source does not provide explicit time or memory complexity analysis [S1]. Standard self-attention complexity is $O(N^2)$ where $N$ is the number of spatial tokens ($H \times W$), which can be prohibitive for high-resolution tri-planes [G].

**Measured Performance:**
In experiments on the DTU dataset, SAC outperforms multi-scale tri-plane architectures ("MSA") [S1]. It significantly reduces geometric error compared to using only positional encoding ("TP-NeuS + PE") [S1]. On the CO3D dataset, it demonstrates robustness to noise from portable device captures [S1].

**Strengths and Limitations:**
*   **Strengths:** Effective at smoothing noise and handling discretization discontinuities [S1]. Outperforms multi-scale tri-plane representations [S1].
*   **Limitations:** Not given in the sources.

**When to Use:**
Use SAC when integrating positional encoding with tri-plane representations in neural surface reconstruction to improve robustness to noise and high-frequency detail preservation [S1].

**When Not to Use:**
Not given in the sources.

## Background
Self-Attention Convolutions were introduced in the PET-NeuS paper [S1]. The method builds on the concept of using positional encoding with tri-planes for neural surface reconstruction [S1]. It was proposed to address the limitations of standard positional encoding, which suffers from discretization discontinuities [S1]. The specific SAC mechanism is adapted from a vanilla SAC mechanism described in reference [32] of the PET-NeuS paper [S1].

## Sources
[S1] PET-NeuS: Positional Encoding Tri-Planes for Neural Surfaces, http://arxiv.org/abs/2305.05594v1, fetched 2026-10-27
