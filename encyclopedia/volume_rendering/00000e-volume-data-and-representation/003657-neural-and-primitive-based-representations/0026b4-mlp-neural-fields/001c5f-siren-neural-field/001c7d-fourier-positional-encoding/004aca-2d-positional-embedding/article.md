# 2D Positional Embedding

Also called: 2D Sine-Cosine Positional Embedding Generation, Fourier Features, Sinusoidal Positional Embedding

## Overview
2D Positional Embedding maps 2D spatial coordinates to a higher-dimensional feature space using interleaved sine and cosine functions at exponentially increasing frequencies. It solves the problem of MLPs struggling to represent high-frequency details by providing a rich, deterministic basis for the network to learn from [S1, S2, unofficial]. Unlike its sibling [Fourier Feature Networks](2765-fourier-feature-networks.md), which learns the frequency matrix via optimization, this method uses fixed, predefined frequencies [S1, S2, unofficial]. It builds on [Fourier Positional Encoding](1267-fourier-positional-encoding.md) by specifically handling 2D grid or batched coordinate inputs common in vision tasks.

## How it works
**Inputs:**
1.  Coordinates tensor `xy` of shape $(B, N, 2)$, where $B$ is batch size, $N$ is number of points, and the last dimension contains $(x, y)$ [S2, unofficial].
2.  Alternatively, a grid size `grid_size` (integer) and `embed_dim` (integer, must be even) for generating a full 2D grid embedding [S1, unofficial].
3.  Optional parameters: `n_cls_token` (number of class tokens to prepend zeros for) [S1, unofficial], `cat_coords` (boolean flag to concatenate original coordinates) [S2, unofficial].

**Outputs:**
A tensor of shape $(B, N, C)$ or $(N_{grid}, C)$ containing the positional embeddings, where $C$ is the embedding dimension [S1, S2, unofficial].

**Data Structures:**
*   `div_term`: A 1D tensor of shape $(1, 1, C/2)$ containing the frequency scaling factors [S2, unofficial].
*   `omega`: A 1D array of shape $(embed\_dim // 2)$ containing the base frequencies [S1, unofficial].

**Steps:**

1.  **Frequency Generation:**
    *   *Method A (Batched Coordinates, PyTorch-style):* Compute `div_term` as:
        $$ \text{div\_term}_i = \frac{1000.0}{C} \cdot i \quad \text{for } i \in \{0, \dots, C/2 - 1\} $$
        Reshape to $(1, 1, C/2)$ [S2, unofficial].
    *   *Method B (Grid-based, NumPy-style):* Split `embed_dim` in half. Compute base frequencies:
        $$ \omega_j = \frac{1}{10000^{j / (embed\_dim / 2)}} \quad \text{for } j \in \{0, \dots, embed\_dim/2 - 1\} $$
        [S1, unofficial].

2.  **Coordinate Processing:**
    *   *If using `xy` (Batched):* Extract $x$ and $y$ from `xy` [S2, unofficial].
    *   *If using `grid_size`:* Create 1D grids for height and width using `np.arange(grid_size)`. Use `meshgrid` to create 2D coordinate grids $X$ and $Y$ [S1, unofficial]. Reshape to $[2, 1, \text{grid\_size}, \text{grid\_size}]$ [S1, unofficial].

3.  **Sine-Cosine Computation:**
    *   *Method A:* Initialize output tensors `pe_x` and `pe_y` of shape $(B, N, C)$ with zeros [S2, unofficial].
        $$ \text{pe\_x}[:, :, 0::2] = \sin(x \cdot \text{div\_term}) $$
        $$ \text{pe\_x}[:, :, 1::2] = \cos(x \cdot \text{div\_term}) $$
        $$ \text{pe\_y}[:, :, 0::2] = \sin(y \cdot \text{div\_term}) $$
        $$ \text{pe\_y}[:, :, 1::2] = \cos(y \cdot \text{div\_term}) $$
        Concatenate `pe_x` and `pe_y` along the last dimension [S2, unofficial].
    *   *Method B:* Compute the product of position and frequencies:
        $$ \text{out}_{m,j} = \text{pos}_m \cdot \omega_j $$
        where $\text{pos}$ is the flattened coordinate grid [S1, unofficial].
        $$ \text{emb} = \text{concat}(\sin(\text{out}), \cos(\text{out})) $$
        [S1, unofficial].

4.  **Post-processing:**
    *   If `cat_coords` is true, concatenate the original coordinates $(x, y)$ to the embedding [S2, unofficial].
    *   If `n_cls_token > 0`, prepend $n\_cls\_token$ rows of zeros of shape $(n\_cls\_token, C)$ to the embedding [S1, unofficial].
    *   If the embedding size needs to match a specific checkpoint resolution (e.g., for interpolation), use bilinear or nearest neighbor interpolation to resize the grid embedding [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def get_2d_sincos_pos_embed(embed_dim, grid_size, n_cls_token=0, cat_coords=False):
    """
    Generates 2D sine-cosine positional embeddings for a grid.
    """
    assert embed_dim % 2 == 0, "Embed dimension must be even"
    
    # Step 1: Create 1D grids
    h, w = grid_size, grid_size
    pos_h = np.arange(h, dtype=np.float32)
    pos_w = np.arange(w, dtype=np.float32)
    
    # Step 2: Meshgrid
    grid_h, grid_w = np.meshgrid(pos_h, pos_w, indexing='ij')
    
    # Flatten and stack to get (N, 2) coordinates
    # Note: Standard implementation often processes H and W separately then combines
    # or treats them as a single 2D coordinate. The source [S1, unofficial] implies processing 
    # the grid structure directly.
    
    # Method B from notes: Compute omega
    half_dim = embed_dim // 2
    omega = np.exp(np.arange(half_dim, dtype=np.float32) * (-np.log(10000.0) / half_dim))
    
    # Process Height
    # out_h shape: (h*w, half_dim)
    out_h = np.einsum('m,d->md', grid_h.flatten(), omega)
    emb_h = np.concatenate([np.sin(out_h), np.cos(out_h)], axis=1)
    
    # Process Width
    out_w = np.einsum('m,d->md', grid_w.flatten(), omega)
    emb_w = np.concatenate([np.sin(out_w), np.cos(out_w)], axis=1)
    
    # Combine H and W embeddings
    # Source [S1, unofficial] implies splitting embed_dim in half for each dimension? 
    # Actually, standard ViT pos embed splits the *embedding vector* between H and W.
    # Let's follow the specific note [S1, unofficial]: "Split embed_dim in half... compute 1D embeddings"
    # This usually means: first half of dim comes from H, second half from W.
    
    pos_embed = np.zeros((h * w, embed_dim), dtype=np.float32)
    pos_embed[:, :half_dim] = emb_h
    pos_embed[:, half_dim:] = emb_w
    
    # Step 3: Reshape to grid
    pos_embed = pos_embed.reshape(h, w, embed_dim)
    pos_embed = np.expand_dims(pos_embed, axis=0) # (1, H, W, C)
    
    # Step 4: Prepend class tokens
    if n_cls_token > 0:
        cls_tokens = np.zeros((1, n_cls_token, embed_dim), dtype=np.float32)
        pos_embed = np.concatenate([cls_tokens, pos_embed], axis=1)
        
    return pos_embed

def get_2d_sincos_pos_embed_batched(xy, embed_dim, cat_coords=False):
    """
    Generates embeddings for batched coordinates (B, N, 2).
    """
    assert embed_dim % 2 == 0
    B, N, _ = xy.shape
    half_dim = embed_dim // 2
    
    # Compute div_term: (1, 1, half_dim)
    div_term = torch.exp(torch.arange(0, half_dim, dtype=torch.float32) * (-np.log(10000.0) / half_dim))
    div_term = div_term.view(1, 1, -1)
    
    x = xy[:, :, 0:1] # (B, N, 1)
    y = xy[:, :, 1:2] # (B, N, 1)
    
    # Compute sin/cos for x and y
    # Note: Source [S2, unofficial] uses 1000.0 / C scaling. 
    # Let's stick to the standard 10000^(i/d) as per [S1, unofficial] and common practice, 
    # but note [S2, unofficial] uses 1000.0/C. We will use the formula from [S2, unofficial] for batched if specified, 
    # but [S1, unofficial] is more standard for "2D Sine-Cosine". 
    # The prompt notes [S2, unofficial] explicitly: div_term = torch.arange(0, C, 2) * (1000.0 / C).
    # This is a linear spacing, not exponential. We must follow the source [S2, unofficial] for this variant.
    
    # Re-evaluating based on [S2, unofficial] notes:
    # div_term = torch.arange(0, embed_dim, 2) * (1000.0 / embed_dim)
    # This creates a linearly spaced frequency vector.
    
    div_term_linear = torch.arange(0, embed_dim, 2, dtype=torch.float32) * (1000.0 / embed_dim)
    div_term_linear = div_term_linear.view(1, 1, -1)
    
    pe_x = torch.zeros(B, N, embed_dim, dtype=torch.float32)
    pe_y = torch.zeros(B, N, embed_dim, dtype=torch.float32)
    
    # Even indices: sin, Odd indices: cos
    # Source [S2, unofficial]: pe_x[:, :, 0::2] = sin(x * div_term)
    # Note: div_term in [S2, unofficial] has shape (1,1,C/2) but is multiplied by x (B,N,1).
    # The arange in [S2, unofficial] is 0 to C step 2.
    
    div_term_half = torch.arange(0, embed_dim // 2, dtype=torch.float32) * (1000.0 / embed_dim)
    div_term_half = div_term_half.view(1, 1, -1)
    
    pe_x[:, :, 0::2] = torch.sin(x * div_term_half)
    pe_x[:, :, 1::2] = torch.cos(x * div_term_half)
    
    pe_y[:, :, 0::2] = torch.sin(y * div_term_half)
    pe_y[:, :, 1::2] = torch.cos(y * div_term_half)
    
    pos_embed = torch.cat([pe_x, pe_y], dim=-1)
    
    if cat_coords:
        pos_embed = torch.cat([pos_embed, xy], dim=-1)
        
    return pos_embed
```

**Parameters:**
*   `embed_dim`: Output dimension for each position. Must be even [S1, unofficial].
*   `grid_size`: Height and width of the grid (integer) [S1, unofficial].
*   `xy`: Coordinates tensor of shape $(B, N, 2)$ [S2, unofficial].
*   `C`: Embedding dimension (alias for `embed_dim`) [S2, unofficial].
*   `n_cls_token`: Number of class tokens to prepend zeros for (default 0) [S1, unofficial].
*   `cat_coords`: Flag to concatenate original coordinates (default False) [S2, unofficial].

**Edge Cases:**
*   `embed_dim` must be even. If odd, the sine/cosine pairing is incomplete [S1, unofficial].
*   Precision issues may arise with very large coordinates or very high frequencies due to floating-point limitations in trigonometric functions [G].
*   Boundary conditions for grid interpolation should be handled explicitly if resizing is required [S1, unofficial].

## Subtopics

### [Expected Sine/Cosine Encoding](12852-expected-sine-cosine-encoding.md)
Expected Sine/Cosine Encoding computes the expected value of sine and cosine functions for input coordinates treated as Gaussian random variables with a specified mean and variance. It serves as an integrated positional encoding that accounts for uncertainty or scale in the input domain, distinguishing it from standard Fourier positional encoding which assumes deterministic inputs. This method is particularly useful in neural fields where inputs represent volumes or regions rather than points, ensuring the encoded features remain statistically consistent with the underlying distribution.

#### One-blob Encoding
One-blob Encoding is a continuous variant of the one-hot encoding based on rasterizing a kernel [S4]. It achieves more accurate results than frequency encodings in bounded domains but is single-scale [S4].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot C)$ for $N$ points and embedding dimension $C$. Computation is dominated by trigonometric functions.
*   **Memory Complexity:** $O(N \cdot C)$ to store the embeddings.
*   **Strengths:** Deterministic, no training required for the encoder itself, effective at helping MLPs learn high-frequency details [S1, S2, unofficial].
*   **Limitations:** Fixed frequencies may not be optimal for all datasets; linear spacing (as in [S2, unofficial]) vs exponential spacing (as in [S1, unofficial]) can lead to different performance characteristics [S1, S2, unofficial]. Single-scale nature of variants like One-blob limits multi-resolution capability [S4].
*   **When to use:** When using MLPs for coordinate-based representations (e.g., NeRF, SIREN) and computational overhead of learning frequencies is undesirable [S1, S2, unofficial].
*   **When not to use:** When input uncertainty is significant (use Expected Sine/Cosine) or when multi-scale features are required without additional network depth [S4].

## Background
The method originates from the use of sinusoidal functions for positional encoding in sequence modeling and vision transformers, later adapted for coordinate-based neural networks. The specific implementation details for 2D grids are found in various open-source repositories [S1, S2, unofficial]. The theoretical basis for using frequency encodings in computer graphics, such as approximating visibility functions, dates back to Annen et al. [2007] and Jansen and Bavoil [2010] [S4]. Müller et al. [2019, 2020] introduced continuous variants like One-blob encoding [S4].

## Sources
[S1] cvg/NoPoSplat, https://github.com/cvg/NoPoSplat, fetched 2026-10-27 (unofficial)
[S2] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
[S3] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S4] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
