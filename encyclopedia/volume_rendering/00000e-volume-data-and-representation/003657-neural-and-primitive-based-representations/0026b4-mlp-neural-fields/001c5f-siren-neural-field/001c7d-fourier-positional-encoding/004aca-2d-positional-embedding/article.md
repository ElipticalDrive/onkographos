# 2D Positional Embedding

Also called: Fourier Features, Sinusoidal Positional Embedding

## Overview
2D Positional Embedding maps 2D input coordinates to a higher-dimensional space using interleaved sine and cosine functions at exponentially increasing frequencies, enabling MLPs to represent high-frequency details in 2D domains [S1, S2, unofficial]. It distinguishes itself from its parent [Fourier Positional Encoding](1267-fourier-positional-encoding.md) by explicitly handling 2D coordinate tensors $(x, y)$ and often concatenating original coordinates, whereas siblings like [Fourier Feature Networks](2765-fourier-feature-networks.md) typically treat frequency matrices as trainable parameters rather than fixed sinusoidal bases [S3].

## How it works
**Inputs and Outputs:**
- Input: A tensor `xy` of shape $(B, N, 2)$, where $B$ is batch size, $N$ is number of points, and the last dimension contains $(x, y)$ coordinates.
- Output: A tensor of shape $(B, N, C_{out})$, where $C_{out} = C + 2$ if `cat_coords` is true, else $C$. $C$ is the embedding dimension for sine/cosine components.

**Data Structures:**
- `div_term`: A 1D tensor of shape $(C/2,)$ containing frequency scaling factors.
- `pe_x`, `pe_y`: 3D tensors of shape $(B, N, C)$ storing the encoded features for x and y respectively.

**Parameters:**
- `C`: Integer, size of the embedding dimension for sine/cosine components. Must be even. [S1, unofficial]
- `cat_coords`: Boolean flag. If true, original $(x, y)$ are concatenated to the end of the embedding. [S1, unofficial]
- `base_frequency`: Constant $1000.0$ used in frequency calculation. [S1, unofficial]

**Steps:**
1. Extract $x$ and $y$ from `xy`. $x$ has shape $(B, N)$, $y$ has shape $(B, N)$.
2. Compute `div_term` as:
   $$ \text{div\_term}[i] = \frac{1000.0}{C} \cdot i \quad \text{for } i \in \{0, 1, \dots, C/2 - 1\} $$
   Reshape `div_term` to $(1, 1, C/2)$ for broadcasting. [S1, unofficial]
3. Initialize `pe_x` and `pe_y` as zero tensors of shape $(B, N, C)$.
4. Compute sine and cosine components for $x$:
   - For even indices $j \in \{0, 2, \dots, C-2\}$:
     $$ \text{pe\_x}[:, :, j] = \sin(x \cdot \text{div\_term}[:, :, j//2]) $$
   - For odd indices $j \in \{1, 3, \dots, C-1\}$:
     $$ \text{pe\_x}[:, :, j] = \cos(x \cdot \text{div\_term}[:, :, j//2]) $$
5. Compute sine and cosine components for $y$ using the same logic as step 4, storing results in `pe_y`.
6. Concatenate `pe_x` and `pe_y` along the last dimension to form `pe` of shape $(B, N, C)$.
7. If `cat_coords` is true, concatenate `xy` to `pe` along the last dimension.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def two_d_positional_embedding(xy: torch.Tensor, C: int, cat_coords: bool = False) -> torch.Tensor:
    """
    Computes 2D positional embedding using sinusoidal functions.
    
    Args:
        xy: Tensor of shape (B, N, 2) containing x and y coordinates.
        C: Embedding dimension for sine/cosine components (must be even).
        cat_coords: Whether to concatenate original coordinates.
        
    Returns:
        Tensor of shape (B, N, C) or (B, N, C+2).
    """
    B, N, _ = xy.shape
    x = xy[:, :, 0]  # Shape (B, N)
    y = xy[:, :, 1]  # Shape (B, N)
    
    # Compute div_term: (1, 1, C/2)
    # div_term[i] = (1000.0 / C) * i
    div_term = torch.arange(0, C, 2, dtype=xy.dtype, device=xy.device) * (1000.0 / C)
    div_term = div_term.view(1, 1, -1)  # Shape (1, 1, C/2)
    
    # Initialize embeddings
    pe_x = torch.zeros(B, N, C, dtype=xy.dtype, device=xy.device)
    pe_y = torch.zeros(B, N, C, dtype=xy.dtype, device=xy.device)
    
    # Fill even indices with sin, odd with cos
    # pe_x[:, :, 0::2] = sin(x * div_term)
    # pe_x[:, :, 1::2] = cos(x * div_term)
    pe_x[:, :, 0::2] = torch.sin(x.unsqueeze(-1) * div_term)
    pe_x[:, :, 1::2] = torch.cos(x.unsqueeze(-1) * div_term)
    
    pe_y[:, :, 0::2] = torch.sin(y.unsqueeze(-1) * div_term)
    pe_y[:, :, 1::2] = torch.cos(y.unsqueeze(-1) * div_term)
    
    # Concatenate x and y embeddings
    pe = torch.cat([pe_x, pe_y], dim=-1)  # Shape (B, N, C)
    
    # Optionally concatenate original coordinates
    if cat_coords:
        pe = torch.cat([pe, xy], dim=-1)  # Shape (B, N, C+2)
        
    return pe
```

**Edge Cases and Pitfalls:**
- `C` must be even. If odd, the indexing `0::2` and `1::2` will not cover all dimensions correctly, leading to shape mismatches or uninitialized values [S1, unofficial].
- Precision issues may arise if coordinates are very large, causing high-frequency oscillations to lose precision in floating-point representation.
- The base frequency $1000.0/C$ is fixed. For domains with different scales, this may not be optimal compared to trainable Fourier Features [S3].

## Subtopics

### [Expected Sine/Cosine Encoding](12852-expected-sine-cosine-encoding.md)
Expected Sine/Cosine Encoding computes the expected value of sine and cosine functions for input coordinates treated as Gaussian random variables with a specified mean and variance. It serves as an integrated positional encoding that accounts for uncertainty or scale in the input domain, distinguishing it from standard Fourier positional encoding which assumes deterministic inputs. This method is particularly useful in neural fields where inputs represent volumes or regions rather than points, ensuring the encoded features remain statistically consistent with the underlying distribution.

### One-blob Encoding
One-blob Encoding is a continuous variant of the one-hot encoding based on rasterizing a kernel. It achieves more accurate results than frequency encodings in bounded domains but is single-scale [S3].

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$ per point, where $N$ is the number of points and $C$ is the embedding dimension. The computation involves element-wise sine and cosine operations.
- **Memory Complexity:** $O(N \cdot C)$ to store the embedding tensor.
- **Strengths:** Simple to implement, differentiable, and effective at helping MLPs learn high-frequency functions without trainable parameters for the encoding itself [S3].
- **Limitations:** Fixed frequency spectrum may not adapt well to varying scales in the input domain. Single-scale nature of variants like One-blob Encoding limits multi-resolution representation [S3].
- **When to use:** When input coordinates are 2D and fixed sinusoidal encoding is sufficient. When computational overhead of trainable Fourier Features is undesirable.
- **When not to use:** When input uncertainty needs to be modeled (use Expected Sine/Cosine Encoding) or when multi-scale representation is critical (use variants like Integrated Positional Encoding).

## Background
2D Positional Embedding is a specific application of Fourier Positional Encoding to 2D domains. It is commonly used in neural rendering and image-based learning tasks where 2D coordinates (e.g., pixel locations) are fed into MLPs. The specific implementation details, such as the base frequency $1000.0/C$, are often derived from standard practices in deep learning libraries like PyTorch [S1, unofficial]. The broader family of Fourier Feature Networks was popularized by [S3] for improving neural network optimization in graphics.

## Sources
[S1] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
[S2] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S3] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
