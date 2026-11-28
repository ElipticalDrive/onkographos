# Rotary Position Embedding (RoPE)

Also called: RoPE2D, Rotary Position Embedding 2D, Rotary Positional Embedding, cuRoPE2D

## Overview
Rotary Position Embedding (RoPE) is a positional encoding method that injects spatial or sequential position information into token embeddings by applying relative rotational transformations. Unlike Fourier Positional Encoding [1267-fourier-positional-encoding.md], which maps coordinates to a higher-dimensional space using fixed-frequency sine/cosine functions, RoPE modifies the internal representation of the tokens themselves via rotation matrices, preserving the inner product between tokens as a function of their relative distance. This article describes the 2D variant (RoPE2D) used in [S1, unofficial], which applies 1D RoPE independently to split feature halves based on 2D spatial coordinates.

## How it works
**Inputs and Outputs:**
- Input tokens: Tensor of shape `[batch_size, nheads, ntokens, dim]`.
- Input positions: Tensor of shape `[batch_size, ntokens, 2]`, where the last dimension contains `(y, x)` coordinates.
- Output: Transformed tokens of the same shape as input tokens.

**Data Structures and Parameters:**
- `dim`: Feature dimension. Must be even.
- `base`: Base frequency for inverse frequency calculation. Default: 100.0 [S1, unofficial].
- `inv_freq`: Precomputed tensor of shape `[dim // 2]`.

**Steps:**

1.  **Precompute Inverse Frequencies:**
    Calculate the inverse frequencies for the embedding dimensions. Let $D$ be the feature dimension `dim`.
    $$ \text{inv\_freq} = \frac{1.0}{\text{base}^{\left(\frac{0, 2, \dots, D-2}{D}\right)}} $$
    Specifically, for $k \in \{0, \dots, D/2 - 1\}$:
    $$ \text{inv\_freq}[k] = \frac{1}{\text{base}^{2k/D}} $$

2.  **Compute Positional Frequencies:**
    For a sequence of positions, calculate the frequencies for each dimension. Let $t$ be the array of position indices (e.g., `arange(seq_len)`).
    $$ \text{freqs} = t \otimes \text{inv\_freq} $$
    Where $\otimes$ denotes the outer product, resulting in a tensor of shape `[seq_len, dim // 2]`.

3.  **Double Dimensionality:**
    Concatenate `freqs` with itself along the last dimension to match the full feature dimension.
    $$ \text{freqs}_{\text{full}} = \text{concat}(\text{freqs}, \text{freqs}, \text{dim}=-1) $$
    Shape: `[seq_len, dim]`.

4.  **Compute Trigonometric Values:**
    $$ \cos\_vals = \cos(\text{freqs}_{\text{full}}) $$
    $$ \sin\_vals = \sin(\text{freqs}_{\text{full}}) $$

5.  **Rotate Half Function:**
    Define a helper function `rotate_half(x)` that splits the input tensor `x` along the last dimension into two equal halves, $x_1$ and $x_2$, and returns `concat(-x2, x1)`.
    $$ \text{rotate\_half}(x) = \text{concat}(-x_{\text{second\_half}}, x_{\text{first\_half}}) $$

6.  **Apply 1D RoPE:**
    For a given token sequence and position indices, apply the rotation:
    $$ \text{encoded} = (\text{tokens} \times \cos\_vals) + (\text{rotate\_half}(\text{tokens}) \times \sin\_vals) $$

7.  **Apply 2D RoPE (RoPE2D):**
    In the forward pass, split the input tokens along the feature dimension (`dim=-1`) into two halves: $y$-tokens and $x$-tokens.
    - Apply 1D RoPE to $y$-tokens using the y-coordinates from `positions[:,:,0]`.
    - Apply 1D RoPE to $x$-tokens using the x-coordinates from `positions[:,:,1]`.
    - Concatenate the transformed halves back together along the feature dimension.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def precompute_freqs_2d(base: float, dim: int, y_positions: torch.Tensor, x_positions: torch.Tensor) -> torch.Tensor:
    """
    Precompute 2D rotary frequencies.
    Args:
        base: Base frequency (default 100.0).
        dim: Feature dimension (must be even).
        y_positions: Tensor of shape [batch_size, ntokens].
        x_positions: Tensor of shape [batch_size, ntokens].
    Returns:
        freqs: Tensor of shape [batch_size, ntokens, dim].
    """
    # Step 1: Inverse frequencies
    # inv_freq shape: [dim // 2]
    inv_freq = 1.0 / (base ** (torch.arange(0, dim, 2).float() / dim))
    
    # Step 2 & 3: Compute frequencies for y and x
    # t_y shape: [batch_size, ntokens]
    # freqs_y shape: [batch_size, ntokens, dim // 2]
    freqs_y = torch.einsum('i,j->ij', y_positions, inv_freq)
    freqs_x = torch.einsum('i,j->ij', x_positions, inv_freq)
    
    # Concatenate to double dimensionality
    # freqs_y_full shape: [batch_size, ntokens, dim]
    freqs_y_full = torch.cat((freqs_y, freqs_y), dim=-1)
    freqs_x_full = torch.cat((freqs_x, freqs_x), dim=-1)
    
    # Step 4: Cos and Sin
    cos_y = freqs_y_full.cos()
    sin_y = freqs_y_full.sin()
    cos_x = freqs_x_full.cos()
    sin_x = freqs_x_full.sin()
    
    return cos_y, sin_y, cos_x, sin_x

def rotate_half(x: torch.Tensor) -> torch.Tensor:
    """
    Rotates half the dimensions of the input tensor.
    Args:
        x: Tensor of shape [..., dim].
    Returns:
        Rotated tensor.
    """
    # Split into two halves
    x1, x2 = x.chunk(2, dim=-1)
    # Return concat(-x2, x1)
    return torch.cat((-x2, x1), dim=-1)

def apply_rope2d(tokens: torch.Tensor, positions: torch.Tensor, base: float = 100.0) -> torch.Tensor:
    """
    Apply 2D Rotary Position Embedding.
    Args:
        tokens: Tensor of shape [batch_size, nheads, ntokens, dim].
        positions: Tensor of shape [batch_size, ntokens, 2] with [y, x] coordinates.
        base: Base frequency.
    Returns:
        Encoded tokens.
    """
    batch_size, nheads, ntokens, dim = tokens.shape
    
    # Ensure dim is even
    assert dim % 2 == 0, "Feature dimension must be even."
    
    # Extract y and x positions
    y_pos = positions[:, :, 0] # [batch_size, ntokens]
    x_pos = positions[:, :, 1] # [batch_size, ntokens]
    
    # Precompute frequencies
    cos_y, sin_y, cos_x, sin_x = precompute_freqs_2d(base, dim, y_pos, x_pos)
    
    # Reshape for broadcasting
    # cos_y shape: [batch_size, ntokens, dim] -> [batch_size, 1, ntokens, dim]
    cos_y = cos_y.unsqueeze(1)
    sin_y = sin_y.unsqueeze(1)
    cos_x = cos_x.unsqueeze(1)
    sin_x = sin_x.unsqueeze(1)
    
    # Split tokens into y and x halves along feature dimension
    # tokens shape: [batch_size, nheads, ntokens, dim]
    tokens_y, tokens_x = tokens.chunk(2, dim=-1)
    
    # Apply 1D RoPE to y using y-positions
    # tokens_y shape: [batch_size, nheads, ntokens, dim // 2]
    # cos_y shape: [batch_size, 1, ntokens, dim] -> needs slicing for dim // 2
    # Actually, the precomputed freqs are for the full dim, but we only apply to half.
    # The standard RoPE applies the rotation matrix to pairs of dimensions.
    # In RoPE2D, we apply 1D RoPE to the y-half using y-positions and x-half using x-positions.
    # This implies we need freqs for dim // 2.
    
    # Re-calculate or slice freqs for half dimension
    # Let's recalculate for clarity in implementation
    inv_freq_half = 1.0 / (base ** (torch.arange(0, dim, 2).float() / dim))
    freqs_y_half = torch.einsum('i,j->ij', y_pos, inv_freq_half) # [batch_size, ntokens, dim // 2]
    freqs_x_half = torch.einsum('i,j->ij', x_pos, inv_freq_half) # [batch_size, ntokens, dim // 2]
    
    cos_y_half = freqs_y_half.cos().unsqueeze(1) # [batch_size, 1, ntokens, dim // 2]
    sin_y_half = freqs_y_half.sin().unsqueeze(1)
    cos_x_half = freqs_x_half.cos().unsqueeze(1)
    sin_x_half = freqs_x_half.sin().unsqueeze(1)
    
    # Apply RoPE formula: (tokens * cos) + (rotate_half(tokens) * sin)
    # Note: rotate_half is defined on the full half-tensor
    encoded_y = (tokens_y * cos_y_half) + (rotate_half(tokens_y) * sin_y_half)
    encoded_x = (tokens_x * cos_x_half) + (rotate_half(tokens_x) * sin_x_half)
    
    # Concatenate back
    encoded_tokens = torch.cat((encoded_y, encoded_x), dim=-1)
    
    return encoded_tokens
```

**Edge Cases and Pitfalls:**
- The feature dimension `dim` must be a multiple of two. If not, the chunk operation will fail [S1, unofficial].
- Positions must be 3D with shape `[Batch, Seq, 2]` [S1, unofficial].
- Precision issues may arise with very large position indices or very small/large base values due to floating-point limitations in trigonometric functions.

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$, where $N$ is the number of tokens and $D$ is the feature dimension. The computation involves element-wise multiplication and addition, which is efficient.
- **Memory Complexity:** $O(N \cdot D)$ for storing the frequency tensors.
- **Strengths:** Provides relative positional information naturally through rotation, which is beneficial for tasks requiring understanding of relative distances (e.g., language modeling, spatial reasoning in 2D). It is differentiable and integrates seamlessly with attention mechanisms.
- **Limitations:** Requires even feature dimensions. Performance may degrade if the position indices are out of the range seen during training. Not inherently designed for 3D or higher-dimensional spatial data without extension.

## Background
Rotary Position Embedding (RoPE) was introduced in the paper "RoFormer: Enhanced Transformer with Rotary Position Embedding" [S1, unofficial]. It was developed to address the limitations of absolute positional encodings by providing a relative positional bias that scales well with sequence length. The 2D variant (RoPE2D) described in [S1, unofficial] adapts this concept for spatial coordinates, applying 1D RoPE independently to the y and x components of the feature space. This method is used in the NoPoSplat project [S1, unofficial] for handling spatial positions in neural field representations.

## Sources
[S1] cvg/NoPoSplat, https://github.com/cvg/NoPoSplat, fetched 2026-10-27 (unofficial)
