# Rate-Constrained Quantization

Also called: Quantization-Aware Training with Global Codebook

## Overview
Rate-Constrained Quantization is a model compression technique for Neural Radiance Fields (NeRF) that reduces memory footprint and transmission bitrate by applying structured quantization to Tensor Train (TT) decomposed weights [S1, unofficial]. It distinguishes itself from standard quantization by using a global codebook for non-uniform clustering of core weights and a sequential fine-tuning strategy to mitigate error accumulation across layers [S1, unofficial].

## How it works
**Inputs:** A trained NeRF model with weights $\mathbf{W}_i$ decomposed into Tensor Train cores $\mathbf{Q}_{1i}$ and $\mathbf{Q}_{2i}$, and bias vectors $\mathbf{b}_i$.
**Outputs:** Quantized weights $\hat{\mathbf{Q}}_{1i}, \hat{\mathbf{Q}}_{2i}$ and biases $\hat{\mathbf{b}}_i$ ready for inference or storage.

**Data Structures:**
*   **Global Codebook $C$:** A list of $N$ centroid vectors. Each centroid has the same dimensionality as the flattened weight tensors being quantized.
*   **Quantized Weights:** Stored as indices into $C$ (for cores) or 16-bit floats (for biases).

**Parameters:**
*   $N$: Number of centroids for k-means clustering. Typical values: 32, 64, 256 [S1, unofficial].
*   Target Rank $r$: Controls the compression ratio of the TT decomposition (not part of quantization itself, but prerequisite) [S1, unofficial].

**Steps:**

1.  **Bias Quantization:**
    Quantize all bias vectors $\mathbf{b}_i$ using uniform 16-bit floating-point precision (half precision) [S1, unofficial].

2.  **Codebook Generation:**
    Collect all weight values from all TT cores $\mathbf{Q}_{1i}, \mathbf{Q}_{2i}$ across all layers that fall within the interval $[-1, 1]$. Apply k-means clustering with $N$ centroids to these values to generate the global codebook $C$ [S1, unofficial].

3.  **Sequential Quantization-Aware Training:**
    Iterate through layers $i = 1$ to $L$ (where $L$ is the total number of layers):
    a.  **Identify Active Parameters:** For the current layer $i$, identify weights $\mathbf{Q}_{1i}, \mathbf{Q}_{2i}$ within $[-1, 1]$ and those outside.
    b.  **Handle Outliers:** Parameters outside $[-1, 1]$ (typically < 0.5% of total) are **not** quantized to the codebook. They are retained in 16-bit uniform precision to prevent severe reconstruction degradation [S1, unofficial].
    c.  **Quantize Inliers:** Map parameters within $[-1, 1]$ to their nearest centroid in $C$. Let $\hat{\mathbf{Q}}_{1i}, \hat{\mathbf{Q}}_{2i}$ be the quantized values.
    d.  **Freeze and Fine-tune:** Freeze the quantized weights $\hat{\mathbf{Q}}_{1i}, \hat{\mathbf{Q}}_{2i}$ and biases $\hat{\mathbf{b}}_i$ for all layers $j \le i$. Fine-tune the remaining unfrozen layers $j > i$ by minimizing the rendering loss:
        $$ \arg\min_{\{\mathbf{W}_j, \mathbf{b}_j\}_{j > i}} ||\mathbf{c}' - \mathbf{c}_{gt}||_2^2 $$
        where $\mathbf{c}'$ is the rendered pixel value and $\mathbf{c}_{gt}$ is the ground-truth pixel value [S1, unofficial].

**Pitfalls:**
*   **Truncation Error:** Do not truncate parameters outside $[-1, 1]$ to fit within the codebook range. These outliers are essential for reconstruction quality [S1, unofficial].
*   **Error Accumulation:** Simultaneous quantization of all layers without fine-tuning leads to significant quality loss due to inter-layer dependencies [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rate_constrained_quantization(model, N=256):
    """
    Applies rate-constrained quantization to a NeRF model with TT cores.
    model: Object containing layers with attributes .Q1, .Q2, .bias
    N: Number of centroids for global codebook
    """
    # Step 1: Quantize biases to 16-bit float
    for layer in model.layers:
        layer.bias = layer.bias.to(torch.float16)

    # Step 2: Generate Global Codebook
    # Collect all core weights within [-1, 1] from all layers
    inlier_values = []
    for layer in model.layers:
        q1_flat = layer.Q1.flatten()
        q2_flat = layer.Q2.flatten()
        # Filter for values in [-1, 1]
        mask = (q1_flat >= -1) & (q1_flat <= 1)
        inlier_values.extend(q1_flat[mask].tolist())
        mask = (q2_flat >= -1) & (q2_flat <= 1)
        inlier_values.extend(q2_flat[mask].tolist())
    
    # Apply k-means clustering to generate global codebook C
    # C is a list of N centroids, each of dimension D (flattened core size)
    C = kmeans(inlier_values, n_clusters=N)

    # Step 3: Sequential Quantization-Aware Training
    for i, layer in enumerate(model.layers):
        # Quantize Q1 and Q2
        q1_quantized = torch.zeros_like(layer.Q1)
        q2_quantized = torch.zeros_like(layer.Q2)
        
        # Handle outliers: keep >1.0 or <-1.0 in float16
        # Handle inliers: map to nearest centroid in C
        q1_inlier_mask = (layer.Q1 >= -1) & (layer.Q1 <= 1)
        q2_inlier_mask = (layer.Q2 >= -1) & (layer.Q2 <= 1)
        
        # Quantize Q1
        q1_inliers = layer.Q1[q1_inlier_mask]
        q1_outliers = layer.Q1[~q1_inlier_mask]
        
        # Map inliers to nearest centroid indices
        q1_indices = find_nearest_centroids(q1_inliers, C)
        q1_quantized[q1_inlier_mask] = C[q1_indices]
        q1_quantized[~q1_inlier_mask] = q1_outliers.to(torch.float16)
        
        # Quantize Q2
        q2_inliers = layer.Q2[q2_inlier_mask]
        q2_outliers = layer.Q2[~q2_inlier_mask]
        
        q2_indices = find_nearest_centroids(q2_inliers, C)
        q2_quantized[q2_inlier_mask] = C[q2_indices]
        q2_quantized[~q2_inlier_mask] = q2_outliers.to(torch.float16)
        
        # Freeze current layer's quantized weights
        layer.Q1 = q1_quantized.detach()
        layer.Q2 = q2_quantized.detach()
        layer.bias = layer.bias.detach()
        
        # Fine-tune subsequent layers
        if i < len(model.layers) - 1:
            fine_tune_model(model, frozen_layers_up_to=i)

def find_nearest_centroids(values, codebook):
    # Compute distances to all centroids and return indices of nearest
    # values: tensor of shape (M, D)
    # codebook: tensor of shape (N, D)
    # Returns: tensor of shape (M,) with indices in [0, N)
    pass

def fine_tune_model(model, frozen_layers_up_to):
    # Set requires_grad=False for layers <= frozen_layers_up_to
    # Run optimization step using rendering loss ||c' - c_gt||^2
    pass
```

## Operating characteristics
*   **Compression:** Significantly reduces model size compared to full-precision NeRF, especially when combined with Tensor Train decomposition [S1, unofficial].
*   **Quality:** Maintains high reconstruction quality by preserving outliers in 16-bit and fine-tuning sequentially [S1, unofficial].
*   **Complexity:** Codebook generation is $O(M \cdot N \cdot D)$ where $M$ is number of inlier values, $N$ centroids, $D$ dimension. Fine-tuning adds computational cost proportional to the number of layers and training iterations [S1, unofficial].
*   **When to use:** When model size and transmission bitrate are critical constraints, such as in light field compression [S1, unofficial].
*   **When not to use:** When maximum possible reconstruction fidelity is required without any compression overhead, or when training resources are extremely limited for the fine-tuning phase.

## Background
Rate-Constrained Quantization was proposed by J. Shi et al. in the context of distilling low-rank Neural Radiance Fields for light field compression [S1, unofficial]. It addresses the large model size of initial NeRF implementations, which made transmitting the network less efficient than transmitting compressed light field views [S1, unofficial]. The method builds on Tensor Train decomposition [S1, unofficial] and employs a global codebook strategy distinct from per-layer local codebooks [S1, unofficial].

## Sources
[S1] Distilled Low Rank Neural Radiance Field with Quantization for Light Field Compression, https://core.ac.uk/works/391353499, fetched 2026-10-30 (unofficial)
