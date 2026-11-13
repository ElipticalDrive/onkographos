# Compact 3D Gaussian Representation (C3DGS)

## Overview
Compact 3D Gaussian Representation (C3DGS) is a framework that reduces the memory and storage footprint of 3D Gaussian Splatting (3DGS) by eliminating redundant Gaussians and compressing their attributes using shared representations [S1]. It addresses the high storage requirements of standard 3DGS, which often exceed 1GB for large scenes, by exploiting spatial redundancy in color and geometry [S1]. Unlike standard 3DGS, which stores unique high-precision values for every Gaussian, C3DGS decouples storage size from the number of Gaussians by using a grid-based neural field for view-dependent color and a codebook for geometric attributes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$ initialized from point clouds, along with camera poses and images for training.
*   **Output:** A compact scene representation consisting of a learnable Gaussian mask, a hash-based grid for colors, and a codebook for geometry, enabling real-time rendering with significantly reduced storage.

**Data Structures:**
1.  **Gaussian Mask ($M$):** A learnable scalar mask per Gaussian. If $M_i < \tau$ (threshold), the Gaussian is pruned.
2.  **Hash Grid ($\mathcal{G}$):** A multi-resolution hash table (Instant NGP style) mapping 3D coordinates to color features. It stores color attributes for the entire scene rather than per-Gaussian.
3.  **Geometry Codebook ($\mathcal{C}$):** A fixed-size table of learned geometric primitives (scale and rotation vectors). Each Gaussian stores only an index $k_i$ into this codebook.

**Numbered Steps:**

1.  **Volume-Based Masking (Pruning):**
    During training, a learnable mask $M_i$ is associated with each Gaussian. The optimization process learns to suppress $M_i$ for non-essential Gaussians. Gaussians with low mask values are removed, reducing the total count $N$ without significant quality loss. This reduces computational complexity, which is linearly proportional to $N$ [S1].

2.  **Color Compression via Hash Grid:**
    Instead of storing spherical harmonic (SH) coefficients per Gaussian, view-dependent color is queried from the hash grid $\mathcal{G}$.
    *   For a Gaussian at position $\mathbf{p}$, the color $\mathbf{c}$ is computed by querying $\mathcal{G}(\mathbf{p})$.
    *   The grid uses hash-based spatial hashing for compactness and fast access [S1].
    *   This exploits the spatial redundancy where neighboring Gaussians share similar colors.

3.  **Geometry Compression via Codebook:**
    Scale and rotation attributes are quantized using a codebook $\mathcal{C}$.
    *   The codebook contains $K$ learned geometric prototypes.
    *   Each Gaussian $i$ is assigned an index $k_i$ such that the stored geometry $\mathbf{g}_i$ is retrieved as $\mathcal{C}[k_i]$.
    *   The codebook size $K$ is small, minimizing spatial overhead while capturing the limited variation in scale and rotation across the scene [S1].

4.  **Rendering:**
    For each pixel, the visible Gaussians are sorted and splatted. The color is fetched from $\mathcal{G}$ and the geometry from $\mathcal{C}$ using the Gaussian's position and codebook index, respectively. The opacity is determined by the mask value $M_i$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
# N: number of Gaussians
# K: codebook size
# tau: pruning threshold

# Learnable parameters
mask = learnable_vector(N)          # Mask values for each Gaussian
grid = HashGrid()                   # Instant NGP hash grid for color
codebook = learnable_matrix(K, 6)   # Codebook entries for scale/rotation (6D params)

# Training Step
for batch in training_data:
    # 1. Pruning
    active_indices = [i for i in range(N) if mask[i] > tau]
    
    # 2. Attribute Retrieval
    for i in active_indices:
        pos = gaussians[i].position
        # Color from grid
        color = grid.query(pos)
        
        # Geometry from codebook
        # k_i is the index associated with Gaussian i
        k_i = codebook_indices[i]
        geometry = codebook[k_i]
        
        # 3. Rendering
        render_pixel(pos, color, geometry, opacity=mask[i])
        
    # 4. Loss and Backprop
    loss = compute_rendering_loss()
    loss.backward()
    
    # Update mask, grid, and codebook
    optimizer.step()
```

**Parameters:**
*   **Grid Type:** Hash-based grid representation (Instant NGP) is used for its compactness and speed [S1].
*   **Threshold ($\tau$):** Not given in the sources.
*   **Codebook Size ($K$):** Not given in the sources, but described as "quite small" [S1].

**Edge Cases and Pitfalls:**
*   **Pruning Threshold:** Setting $\tau$ too high may remove essential Gaussians, degrading quality. Setting it too low retains redundancy [S1].
*   **Codebook Capacity:** If $K$ is too small, the codebook may not capture geometric diversity, leading to quantization artifacts [S1].
*   **Hash Collisions:** Standard in hash grids; requires careful resolution tuning to avoid color bleeding [S1].

## Subtopics

### [Optimized Minimal 3D Gaussian Splatting (OMG)](13978-optimized-minimal-3d-gaussian-splatting-omg.md)
OMG retains per-Gaussian parameterization for geometry to handle sparsity, while using a small neural field and Sub-Vector Quantization (SVQ) for appearance features. It builds on C3DGS by addressing the weakness of neural fields in sparse settings where neighboring Gaussians are further apart.

## Operating characteristics
*   **Storage Efficiency:** Achieves approximately 15× reduced storage compared to standard 3DGS across various datasets. With post-processing (quantization and entropy coding), it achieves over 25× compression [S1]. On the Deep Blending dataset, it shows over 28× enhancement in storage efficiency [S1].
*   **Rendering Speed:** Nearly 40% increase in rendering speed on the Deep Blending dataset due to reduced Gaussian count and efficient attribute lookup [S1].
*   **Quality:** Maintains high reconstruction quality (PSNR), outperforming standard 3DGS on the Deep Blending dataset [S1].
*   **Complexity:** Spatial complexity is significantly reduced by sharing color attributes via a grid and geometry via codebooks [S1]. Computational complexity is linearly proportional to the number of active Gaussians [S1].
*   **Strengths:** High compression ratio, fast rendering, and high fidelity.
*   **Limitations:** Relies on spatial redundancy; may struggle in highly sparse or unstructured scenes where neighbors do not share attributes (addressed by OMG) [S1].

## Background
C3DGS was proposed to address the critical memory and storage issues in 3D Gaussian Splatting, which often require >1GB for large real-world scenes [S1]. It builds on the observation that standard densification algorithms produce redundant Gaussians and that spatial redundancy exists in color and geometry attributes [S1]. The method uses Instant NGP for the hash grid [S1].

## Sources
[S1] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
