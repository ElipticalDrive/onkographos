# DuDoTrans

Also called: DuDoTransModel, Dual-Domain Transformer

## Overview
DuDoTrans is a dual-domain transformer architecture designed for sparse-view CT reconstruction, addressing the ill-posed nature of tomographic inversion when projection data is limited [S1, unofficial]. It distinguishes itself from standard self-attention methods by employing Window-based Multi-Head Self-Attention (W-MSA) within a Swin Transformer block structure, which restricts global context modeling to local windows to reduce computational complexity while maintaining the ability to capture long-range dependencies through shifted windows [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method processes 2D image tensors representing CT slices or sinograms. The primary input is a tensor $x$ of shape $(B, H, W, C)$, where $B$ is batch size, $H$ and $W$ are spatial dimensions, and $C$ is the channel dimension. The output is a reconstructed tensor of the same shape $(B, H, W, C)$ after passing through the transformer blocks.

**Data Structures:**
1.  **`relative_position_bias_table`**: A learnable parameter tensor of shape $((2W_h - 1)(2W_w - 1), \text{num\_heads})$. It stores the relative position biases for attention calculations. Initialized with zeros and truncated normal distribution (std=0.02) [S1, unofficial].
2.  **`relative_position_index`**: A buffer tensor of shape $(W_h W_w, W_h W_w)$ containing integer indices that map spatial positions within a window to the `relative_position_bias_table` [S1, unofficial].

**Parameters:**
*   `window_size`: Tuple $(W_h, W_w)$ defining the height and width of the attention window. Default: $(7, 7)$ [S1, unofficial].
*   `shift_size`: Integer defining the shift size for Swin-MSA. If 0, non-shifted windows are used. Default: 0 [S1, unofficial].
*   `mlp_ratio`: Ratio of MLP hidden dimension to embedding dimension. Default: 4.0 [S1, unofficial].
*   `qk_scale`: Override for QK scaling factor. If `None`, defaults to $head\_dim^{-0.5}$ [S1, unofficial].

**Algorithm Steps:**

1.  **Window Partitioning**:
    The input tensor $x$ of shape $(B, H, W, C)$ is partitioned into non-overlapping windows.
    1.  Reshape $x$ to $(B, H/W_s, W_s, W/W_s, W_s, C)$.
    2.  Permute dimensions to $(B, H/W_s, W/W_s, W_s, W_s, C)$.
    3.  Flatten to $(num\_windows \cdot B, W_s, W_s, C)$, where $num\_windows = (H/W_s) \times (W/W_s)$ and $W_s$ is the window size (assuming square windows $W_h=W_w=W_s$ for simplicity in flattening, or handling rectangular windows similarly) [S1, unofficial].

2.  **Relative Position Bias Calculation**:
    Precompute the indices for relative position bias within a window of size $W_h \times W_w$.
    1.  Generate coordinates: $coords_h = \text{arange}(W_h)$, $coords_w = \text{arange}(W_w)$.
    2.  Create meshgrid and flatten to shape $(2, W_h W_w)$. Let this be $coords\_flatten$.
    3.  Compute relative coordinates: $relative\_coords = coords\_flatten[:, :, None] - coords\_flatten[:, None, :]$. Shape: $(2, W_h W_w, W_h W_w)$.
    4.  Permute to $(W_h W_w, W_h W_w, 2)$.
    5.  Shift indices to make them non-negative:
        $relative\_coords[:, :, 0] += W_h - 1$
        $relative\_coords[:, :, 1] += W_w - 1$
    6.  Map 2D relative coordinates to 1D index space:
        $relative\_coords[:, :, 0] *= (2 W_w - 1)$
    7.  Sum to get final index: $relative\_position\_index = relative\_coords.sum(-1)$. Shape: $(W_h W_w, W_h W_w)$ [S1, unofficial].

3.  **Window Attention Forward (W-MSA)**:
    Process each window independently.
    1.  **Linear Projection**: Compute $qkv = \text{Linear}(x_{window})$. Reshape to $(B_, N, 3, \text{num\_heads}, C/\text{num\_heads})$ and permute to $(3, B_, \text{num\_heads}, N, C/\text{num\_heads})$, where $N = W_h W_w$ is the number of tokens in the window.
    2.  **Scale Q**: Extract $Q, K, V$ from $qkv$. Scale $Q$: $Q = Q \cdot scale$, where $scale = qk\_scale$ or $head\_dim^{-0.5}$ [S1, unofficial].
    3.  **Attention Scores**: Compute $attn = Q @ K^T$. Shape: $(B_, \text{num\_heads}, N, N)$.
    4.  **Add Bias**: Retrieve bias using the precomputed index: $bias = \text{relative\_position\_bias\_table}[relative\_position\_index]$. Reshape bias to $(N, N, \text{num\_heads})$ and permute to $(\text{num\_heads}, N, N)$. Add to $attn$: $attn = attn + bias$.
    5.  **Softmax**: Apply softmax along the last dimension: $attn = \text{softmax}(attn)$.
    6.  **Dropout**: Apply attention dropout [S1, unofficial].
    7.  **Output**: Compute $out = attn @ V$. Transpose and reshape to merge heads: $x_{out} = (attn @ V)^T$. Reshape to $(B_, N, C)$, apply linear projection, and apply projection dropout [S1, unofficial].

4.  **Window Reverse**:
    Reconstruct the full image tensor from the processed windows.
    1.  Reshape windows from $(num\_windows \cdot B, W_s, W_s, C)$ to $(B, H/W_s, W/W_s, W_s, W_s, C)$.
    2.  Permute to $(B, H/W_s, W_s, W/W_s, W_s, C)$.
    3.  Flatten to $(B, H, W, C)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Input Resolution < Window Size**: If the input height or width is smaller than or equal to the window size, the shift size is forced to 0, and the window size is reduced to the minimum input dimension to avoid invalid partitioning [S1, unofficial].
*   **Precision**: Floating-point precision in ray marching (if used in conjunction with differentiable rendering backends) can affect gradient stability, though the transformer core relies on standard float32/float16 operations [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DuDoTransWindowAttention(nn.Module):
    def __init__(self, dim, window_size, num_heads, qkv_bias=True, qk_scale=None, attn_drop=0., proj_drop=0.):
        super().__init__()
        self.dim = dim
        self.window_size = window_size  # Wh, Ww
        self.num_heads = num_heads
        head_dim = dim // num_heads
        self.scale = qk_scale or head_dim ** -0.5
        
        # qkv linear layer
        self.qkv = nn.Linear(dim, dim * 3, bias=qkv_bias)
        
        # Relative position bias table
        # (2*Wh-1) * (2*Ww-1) * num_heads
        self.relative_position_bias_table = nn.Parameter(
            torch.zeros((2 * window_size[0] - 1) * (2 * window_size[1] - 1), num_heads)
        )
        
        # Get pair-wise relative position index for each token inside the window
        coords_h = torch.arange(self.window_size[0])
        coords_w = torch.arange(self.window_size[1])
        coords = torch.stack(torch.meshgrid([coords_h, coords_w]))  # 2, Wh, Ww
        coords_flatten = torch.flatten(coords, 1)  # 2, Wh*Ww
        relative_coords = coords_flatten[:, :, None] - coords_flatten[:, None, :]  # 2, Wh*Ww, Wh*Ww
        relative_coords = relative_coords.permute(1, 2, 0).contiguous()  # Wh*Ww, Wh*Ww, 2
        relative_coords[:, :, 0] += self.window_size[0] - 1  # shift to start from 0
        relative_coords[:, :, 1] += self.window_size[1] - 1
        relative_coords[:, :, 0] *= 2 * self.window_size[1] - 1
        relative_position_index = relative_coords.sum(-1)  # Wh*Ww, Wh*Ww
        self.register_buffer("relative_position_index", relative_position_index)
        
        self.attn_drop = nn.Dropout(attn_drop)
        self.proj = nn.Linear(dim, dim)
        self.proj_drop = nn.Dropout(proj_drop)
        
        nn.init.trunc_normal_(self.relative_position_bias_table, std=.02)

    def forward(self, x, mask=None):
        """
        x: shape (B_, N, C) where N = Wh * Ww
        mask: shape (B_, num_windows, Wh*Ww, Wh*Ww) or None
        """
        B_, N, C = x.shape
        
        qkv = self.qkv(x).reshape(B_, N, 3, self.num_heads, C // self.num_heads).permute(2, 0, 3, 1, 4)
        q, k, v = qkv[0], qkv[1], qkv[2]  # B_, num_heads, N, head_dim
        
        q = q * self.scale
        attn = (q @ k.transpose(-2, -1))
        
        # Add relative position bias
        relative_position_bias = self.relative_position_bias_table[self.relative_position_index.view(-1)].view(
            self.window_size[0] * self.window_size[1], self.window_size[0] * self.window_size[1], -1)  # Wh*Ww, Wh*Ww, nH
        relative_position_bias = relative_position_bias.permute(2, 0, 1).contiguous()  # nH, Wh*Ww, Wh*Ww
        attn = attn + relative_position_bias.unsqueeze(0)
        
        if mask is not None:
            nW = mask.shape[0]
            attn = attn.view(B_ // nW, nW, self.num_heads, N, N) + mask.unsqueeze(1).unsqueeze(0)
            attn = attn.view(-1, self.num_heads, N, N)
            
        attn = self.attn_drop(attn.softmax(dim=-1))
        
        x = (attn @ v).transpose(1, 2).reshape(B_, N, C)
        x = self.proj(x)
        x = self.proj_drop(x)
        return x

    def extra_repr(self) -> str:
        return f'dim={self.dim}, window_size={self.window_size}, num_heads={self.num_heads}'
```

## Subtopics

### Swin Transformer Blocks
The core building block of DuDoTrans, combining Window Attention with Shifted Window Attention to enable cross-window connection. Choose this when computational efficiency is critical for high-resolution volumes compared to full self-attention. [Link](10441-transformers-and-vision-transformers-vit.md)

### Transformer-based Tomographic Reconstruction
The broader class of methods using transformers for CT reconstruction. DuDoTrans is a specific instance focusing on dual-domain (spatial and frequency or multi-view) features. [Link](11047-transformer-based-tomographic-reconstruction.md)

## Operating characteristics
*   **Time Complexity**: The window-based attention reduces complexity from $O(N^2)$ to $O(N \cdot W_s^2)$, where $N$ is the total number of tokens and $W_s$ is the window size [S1, unofficial].
*   **Memory Complexity**: Proportional to the number of tokens and window size, significantly lower than global attention for large images.
*   **Performance**: Effective for sparse-view CT reconstruction, mitigating artifacts common in traditional methods like FBP when projections are limited [S1, unofficial].
*   **Strengths**: Captures global context via shifted windows while maintaining local computational efficiency; robust to sparse data.
*   **Limitations**: Performance may degrade if the window size is not well-tuned for the specific frequency content of the CT data; requires careful handling of boundary conditions [S1, unofficial].

## Background
DuDoTrans was introduced by Ce Wang, Kun Shang, Haimiao Zhang, Qian Li, and S. Kevin Zhou in the paper "DuDoTrans: Dual-Domain Transformer for Sparse-View CT Reconstruction," published in the International Workshop on Machine Learning for Medical Image Reconstruction in 2022 [S1, unofficial]. It builds upon the Swin Transformer architecture, adapting it for the specific challenges of tomographic reconstruction where data sparsity leads to ill-posed inverse problems.

## Sources
[S1] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
