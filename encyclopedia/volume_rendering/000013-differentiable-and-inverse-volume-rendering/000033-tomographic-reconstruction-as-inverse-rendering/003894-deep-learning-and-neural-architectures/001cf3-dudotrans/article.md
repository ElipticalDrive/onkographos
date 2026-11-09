# DuDoTrans

Also called: DuDoTransModel

## Overview
DuDoTrans is a dual-domain transformer architecture designed for sparse-view CT reconstruction [S1, unofficial]. It addresses the ill-posed nature of tomographic reconstruction from limited angular views by integrating a differentiable forward projection operator with a deep learning prior [S1, unofficial]. Unlike standard Multi-Head Self-Attention (MHSA) which computes global attention with quadratic complexity, DuDoTrans utilizes Window-based Multi-Head Self-Attention (W-MSA) to restrict attention to local windows, reducing computational complexity from $O(N^2)$ to $O(N \cdot W_s^2)$ where $N$ is the number of tokens and $W_s$ is the window size [S1, unofficial]. It builds on [Deep Learning and Neural Architectures](8282-deep-learning-and-neural-architectures.md) by employing Swin Transformer blocks as the core regularizer $R(x)$ in the inverse rendering optimization loop.

## How it works
DuDoTrans operates within the iterative optimization framework defined by the parent article [Deep Learning and Neural Architectures](8282-deep-learning-and-neural-architectures.md). The method minimizes the loss $\mathcal{L}$ between the measured sinogram $p$ and the rendered projection $\hat{p} = \mathcal{P}(x)$, where the regularizer $R(x)$ is implemented via the DuDoTrans neural network. The core innovation lies in the internal attention mechanism of the transformer blocks, which processes the volume features (or sinogram features, depending on the specific dual-domain implementation stage) using windowed attention with relative position biases.

### Inputs and Outputs
*   **Input:** A batch of 2D feature maps (representing either the sinogram domain or the image domain) with shape $(B, H, W, C)$, where $B$ is batch size, $H$ and $W$ are spatial dimensions, and $C$ is the channel dimension (embedding dimension).
*   **Output:** A processed feature map of the same shape $(B, H, W, C)$.

### Data Structures
The method relies on specific tensor layouts for efficient windowed attention:

1.  **Window Partitioned Tensor:**
    *   Shape: $(num\_windows \cdot B, W_s, W_s, C)$
    *   Derived from reshaping and permuting the input tensor to isolate local windows of size $W_s \times W_s$.
2.  **Relative Position Bias Table:**
    *   Shape: $((2 W_h - 1)(2 W_w - 1), num\_heads)$
    *   Parameter tensor storing learnable bias values for relative spatial positions. Initialized with zeros and truncated normal distribution (std=0.02) [S1, unofficial].
3.  **Relative Position Index:**
    *   Shape: $(W_h W_w, W_h W_w)$
    *   Buffer tensor containing integer indices mapping spatial pairs to the `relative_position_bias_table`. Computed deterministically from window dimensions [S1, unofficial].

### Parameters
*   **window_size:** Tuple $(W_h, W_w)$ defining the height and width of the attention window. Default: $(7, 7)$ [S1, unofficial].
*   **shift_size:** Integer defining the shift amount for shifted window partitioning. If 0, non-shifted windows are used. Default: $0$ [S1, unofficial].
*   **mlp_ratio:** Float ratio of the MLP hidden dimension to the embedding dimension. Default: $4.0$ [S1, unofficial].
*   **qk_scale:** Float overriding the default QK scaling factor. If `None`, defaults to $head\_dim^{-0.5}$ [S1, unofficial].
*   **num_heads:** Integer number of attention heads. Implicitly derived from embedding dimension $C$ and head dimension $head\_dim = C / num\_heads$.

### Algorithm Steps

#### 1. Window Partitioning
Reshape the input tensor $x$ of shape $(B, H, W, C)$ into local windows.
1.  Reshape $x$ to $(B, H/W_s, W_s, W/W_s, W_s, C)$.
2.  Permute dimensions to $(B, H/W_s, W/W_s, W_s, W_s, C)$.
3.  Flatten the first three dimensions and the last two spatial dimensions to obtain shape $(num\_windows \cdot B, W_s, W_s, C)$, where $num\_windows = (H/W_s) \cdot (W/W_s)$ [S1, unofficial].

#### 2. Relative Position Bias Calculation
Compute the indices for the relative position bias table.
1.  Generate coordinates: $coords_h = \text{arange}(W_h)$, $coords_w = \text{arange}(W_w)$.
2.  Create meshgrid from $coords_h$ and $coords_w$, flatten to shape $(2, W_h W_w)$. Let this be $coords\_flatten$.
3.  Compute relative coordinates: $relative\_coords = coords\_flatten[:, :, None] - coords\_flatten[:, None, :]$. Shape: $(2, W_h W_w, W_h W_w)$.
4.  Permute to $(W_h W_w, W_h W_w, 2)$.
5.  Shift indices to ensure non-negative values:
    *   $relative\_coords[:, :, 0] += W_h - 1$
    *   $relative\_coords[:, :, 1] += W_w - 1$
6.  Map 2D relative coordinates to 1D index space:
    *   $relative\_coords[:, :, 0] *= 2 W_w - 1$
7.  Sum the coordinates to get the final linear index:
    *   $relative\_position\_index = relative\_coords.sum(-1)$
    *   Result shape: $(W_h W_w, W_h W_w)$ [S1, unofficial].

#### 3. Window Attention Forward
Process each window using Multi-Head Self-Attention with relative position bias.
1.  **Linear Projection:** Compute $qkv = \text{Linear}(x_{window})$. Reshape to $(B_, N, 3, num\_heads, C/num\_heads)$, where $B_ = num\_windows \cdot B$ and $N = W_s^2$. Permute to $(3, B_, num\_heads, N, C/num\_heads)$ to separate Q, K, V.
2.  **Scale Q:** $q = q \cdot scale$, where $scale = qk\_scale$ if provided, else $head\_dim^{-0.5}$.
3.  **Attention Scores:** $attn = q @ k^T$. Shape: $(B_, num\_heads, N, N)$.
4.  **Add Bias:** Retrieve bias from table: $bias = \text{table}[relative\_position\_index]$. Reshape bias to $(W_h W_w, W_h W_w, num\_heads)$, permute to $(num\_heads, W_h W_w, W_h W_w)$. Add $bias$ to $attn$.
5.  **Softmax:** Apply softmax along the last dimension (key dimension).
6.  **Dropout:** Apply attention dropout.
7.  **Output Projection:** Compute $out = attn @ v$. Transpose and reshape to $(B_, N, C)$. Apply linear projection and projection dropout [S1, unofficial].

#### 4. Window Reverse (Merge)
Reconstruct the full image from the processed windows.
1.  Reshape windows from $(num\_windows \cdot B, W_s, W_s, C)$ to $(B, H/W_s, W/W_s, W_s, W_s, C)$.
2.  Permute to $(B, H/W_s, W_s, W/W_s, W_s, C)$.
3.  Flatten to $(B, H, W, C)$ [S1, unofficial].

### Pseudocode
The following Python-like pseudocode implements the core Window Attention mechanism of DuDoTrans. It assumes standard tensor operations (reshape, permute, matmul, add, softmax).

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def window_partition(x, window_size):
    """
    Reshape (B, H, W, C) into windows.
    """
    B, H, W, C = x.shape
    # Reshape to (B, H//Ws, Ws, W//Ws, Ws, C)
    x = x.view(B, H // window_size[0], window_size[0], 
              W // window_size[1], window_size[1], C)
    # Permute to (B, H//Ws, W//Ws, Ws, Ws, C)
    x = x.permute(0, 1, 3, 2, 4, 5).contiguous()
    # Flatten to (num_windows * B, Ws, Ws, C)
    num_windows = (H // window_size[0]) * (W // window_size[1])
    x = x.view(num_windows * B, window_size[0], window_size[1], C)
    return x, num_windows

def window_reverse(x_windows, num_windows, window_size, H, W):
    """
    Reconstruct image from windows.
    """
    B = int(x_windows.shape[0] / num_windows)
    # Reshape to (B, H//Ws, W//Ws, Ws, Ws, C)
    x = x_windows.view(B, H // window_size[0], W // window_size[1], 
                       window_size[0], window_size[1], -1)
    # Permute to (B, H//Ws, Ws, W//Ws, Ws, C)
    x = x.permute(0, 1, 3, 2, 4, 5).contiguous()
    # Flatten to (B, H, W, C)
    x = x.view(B, H, W, -1)
    return x

def relative_position_bias_table(relative_position_index, relative_position_bias_table_tensor):
    """
    Retrieve bias values using precomputed indices.
    """
    # relative_position_index shape: (Wh*Ww, Wh*Ww)
    # relative_position_bias_table_tensor shape: ((2Wh-1)*(2Ww-1), num_heads)
    # Output shape: (num_heads, Wh*Ww, Wh*Ww)
    bias = relative_position_bias_table_tensor[relative_position_index]
    # bias shape: (Wh*Ww, Wh*Ww, num_heads)
    bias = bias.permute(2, 0, 1).contiguous()
    return bias

def window_attn(x, num_heads, window_size, relative_position_bias_table_tensor, 
                qkv_linear, proj_linear, qk_scale=None, dropout_p=0.0):
    """
    Core Window Attention block.
    """
    B_, N, C = x.shape
    head_dim = C // num_heads
    scale = qk_scale if qk_scale is not None else (head_dim ** -0.5)
    
    # 1. Linear Projection: (B_, N, C) -> (B_, N, 3 * C)
    qkv = qkv_linear(x)
    # Reshape to (B_, N, 3, num_heads, head_dim)
    qkv = qkv.view(B_, N, 3, num_heads, head_dim)
    # Permute to (3, B_, num_heads, N, head_dim)
    qkv = qkv.permute(2, 0, 3, 1, 4).contiguous()
    q, k, v = qkv[0], qkv[1], qkv[2]
    
    # 2. Scale Q
    q = q * scale
    
    # 3. Attention Scores: (B_, num_heads, N, N)
    attn = q @ k.transpose(-2, -1)
    
    # 4. Add Relative Position Bias
    # Compute relative position index if not cached, or use cached buffer
    # Assuming relative_position_index is a buffer computed once per window size
    # For implementation completeness, we assume it's available or computed here:
    Wh, Ww = window_size
    coords_h = torch.arange(Wh)
    coords_w = torch.arange(Ww)
    coords = torch.stack(torch.meshgrid([coords_h, coords_w], indexing='ij')) # (2, Wh, Ww)
    coords_flatten = coords.reshape(2, -1) # (2, Wh*Ww)
    relative_coords = coords_flatten[:, :, None] - coords_flatten[:, None, :] # (2, Wh*Ww, Wh*Ww)
    relative_coords = relative_coords.permute(1, 2, 0).contiguous() # (Wh*Ww, Wh*Ww, 2)
    
    # Shift indices
    relative_coords[:, :, 0] += Wh - 1
    relative_coords[:, :, 1] += Ww - 1
    # Map to 1D
    relative_coords[:, :, 0] *= 2 * Ww - 1
    relative_position_index = relative_coords.sum(-1) # (Wh*Ww, Wh*Ww)
    
    # Retrieve bias
    # relative_position_bias_table_tensor shape: ((2Wh-1)*(2Ww-1), num_heads)
    bias = relative_position_bias_table_tensor[relative_position_index] # (Wh*Ww, Wh*Ww, num_heads)
    bias = bias.permute(2, 0, 1).contiguous() # (num_heads, Wh*Ww, Wh*Ww)
    
    # Add bias to attention scores
    # attn shape: (B_, num_heads, N, N) where N = Wh*Ww
    attn = attn + bias
    
    # 5. Softmax
    attn = F.softmax(attn, dim=-1)
    
    # 6. Dropout
    attn = F.dropout(attn, p=dropout_p, training=False) # Training mode handled externally
    
    # 7. Output
    # (B_, num_heads, N, head_dim)
    x = (attn @ v).transpose(1, 2).reshape(B_, N, C)
    
    # 8. Projection
    x = proj_linear(x)
    
    return x

class DuDoTransBlock(torch.nn.Module):
    def __init__(self, dim, num_heads, window_size=(7, 7), shift_size=0, 
                 mlp_ratio=4.0, qk_scale=None):
        super().__init__()
        self.dim = dim
        self.num_heads = num_heads
        self.window_size = window_size
        self.shift_size = shift_size
        self.mlp_ratio = mlp_ratio
        
        # Linear layers for QKV and Output
        self.qkv = torch.nn.Linear(dim, dim * 3)
        self.proj = torch.nn.Linear(dim, dim)
        
        # Relative Position Bias Table
        Wh, Ww = window_size
        num_relative_positions = (2 * Wh - 1) * (2 * Ww - 1)
        self.relative_position_bias_table = torch.nn.Parameter(
            torch.zeros(num_relative_positions, num_heads)
        )
        # Truncated normal initialization (std=0.02)
        torch.nn.init.trunc_normal_(self.relative_position_bias_table, std=0.02)
        
        # Precompute relative position index buffer
        coords_h = torch.arange(Wh)
        coords_w = torch.arange(Ww)
        coords = torch.stack(torch.meshgrid([coords_h, coords_w], indexing='ij'))
        coords_flatten = coords.reshape(2, -1)
        relative_coords = coords_flatten[:, :, None] - coords_flatten[:, None, :]
        relative_coords = relative_coords.permute(1, 2, 0).contiguous()
        relative_coords[:, :, 0] += Wh - 1
        relative_coords[:, :, 1] += Ww - 1
        relative_coords[:, :, 0] *= 2 * Ww - 1
        self.register_buffer("relative_position_index", relative_coords.sum(-1))
        
        self.qk_scale = qk_scale
        self.dropout = torch.nn.Dropout(0.0) # Placeholder for actual dropout rate

    def forward(self, x):
        B, H, W, C = x.shape
        
        # Handle edge case: input smaller than window size
        if H <= self.window_size[0] or W <= self.window_size[1]:
            self.shift_size = 0
            # Window size reduction logic would be needed here if strictly enforced,
            # but typically padding or resizing is applied upstream.
            # For this implementation, we assume valid input dimensions.
            pass

        # Window Partitioning
        x_windows, num_windows = window_partition(x, self.window_size)
        
        # Window Attention
        x_windows_attn = window_attn(
            x_windows, 
            self.num_heads, 
            self.window_size, 
            self.relative_position_bias_table,
            self.qkv, 
            self.proj, 
            self.qk_scale,
            self.dropout.p
        )
        
        # Window Reverse
        x = window_reverse(x_windows_attn, num_windows, self.window_size, H, W)
        
        return x
```

### Edge Cases and Pitfalls
*   **Input Resolution vs. Window Size:** If the input resolution $(H, W)$ is smaller than or equal to the window size $(W_h, W_w)$, the shift size is forced to $0$ and the window size is reduced to the minimum input dimension to avoid invalid partitioning [S1, unofficial].
*   **Precision:** The ray marching implementation in the parent context uses `float` (32-bit) precision. Care must be taken with boundary conditions in the forward projection operator $\mathcal{P}$ to avoid artifacts in the gradient $\frac{\partial \mathcal{L}}{\partial x}$ [S1, unofficial].
*   **Memory:** The relative position bias table size grows quadratically with window size. For large windows, this can consume significant memory.

## Operating characteristics
*   **Time Complexity:** The windowed attention reduces complexity from $O(N^2)$ to $O(N \cdot W_s^2)$, where $N$ is the total number of tokens (pixels/voxels) and $W_s$ is the window size [S1, unofficial]. This makes it significantly faster than global self-attention for high-resolution volumes.
*   **Memory Complexity:** Linear in the number of tokens $N$, plus the constant overhead of the relative position bias table $O(W_h W_w \cdot num\_heads)$ [S1, unofficial].
*   **Performance:** Designed for sparse-view CT reconstruction, where traditional methods suffer from streak artifacts. DuDoTrans provides high-quality reconstructions by leveraging the dual-domain (sinogram and image) processing capabilities of the transformer [S1, unofficial].
*   **Strengths:** Efficient local attention, ability to capture long-range dependencies via shifted windows, differentiable integration with tomographic reconstruction.
*   **Limitations:** Performance depends on the quality of the differentiable forward/backward projection operators. Computational cost is higher than non-learning-based iterative methods (e.g., SIRT, SART) per iteration, though convergence may be faster.

## Background
DuDoTrans was introduced by Ce Wang, Kun Shang, Haimiao Zhang, Qian Li, and S. Kevin Zhou in the paper "DuDoTrans: Dual-Domain Transformer for Sparse-View CT Reconstruction" [S1, unofficial]. It was published in the International Workshop on Machine Learning for Medical Image Reconstruction in 2022. The method adapts the Swin Transformer architecture, originally designed for natural image processing, to the domain of tomographic reconstruction by incorporating differentiable projection operators [S1, unofficial].

## Sources
[S1] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
