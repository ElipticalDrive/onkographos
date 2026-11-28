# LightGaussian / CompGS / RDOGS

Also called: Compact3D, Compressing Gaussian Splatting Radiance Field Models with Vector Quantization, Deng et al, End-to-End Rate-Distortion Optimized 3D Gaussian Representation, Pruning and Quantization for 3DGS, RDO-Gaussian, RDOGS

## Overview
LightGaussian (also known as CompGS) is a compression method for 3D Gaussian Splatting that reduces storage per primitive by pruning redundant Gaussians and quantizing parameters using learned codebooks [S1][S4, unofficial]. It combines dynamic pruning during training with entropy-constrained vector quantization to achieve flexible rate control and minimal quality loss [S2][S4, unofficial].

## How it works
**Inputs and outputs:**
- Input: A trained 3D Gaussian Splatting scene containing position, scale, rotation, opacity, and spherical harmonics (SH) coefficients [S1][S2].
- Output: A pruned Gaussian set, a learned codebook, and compressed indices [S1][S2].

**Data structures:**
- Gaussian attributes: Each Gaussian stores quantized attributes $\theta_j$ approximated by codebook entries $q_{z_j}$, where $z_j \in \{1, \dots, K\}$ is the index into a codebook of size $K$ [S1].
- Index storage: Indices are sorted by a quantized parameter and compressed using a run-length-encoding-like scheme [S2].

**Steps:**
1. **Pruning via sliding window mask:** During training, a sliding window mask prevents redundant Gaussian splitting in already covered regions [S4, unofficial].
2. **Dynamic pruning and importance scoring:** Low-contribution primitives are eliminated using dynamic pruning and global significance scoring based on intersected pixels [S2].
3. **Vector quantization:** Gaussian attributes (covariance, color/SH, scale) are discretized via entropy-constrained vector quantization (ECVQ) or K-means clustering, storing only the codebook and per-Gaussian indices [S1][S2].
4. **Reconstruction:** Rendering looks up quantized parameters from the codebook using the stored indices [S1][S2].

**Formulas:**
- Attribute approximation: $\theta_j \approx q_{z_j}$, where $\theta_j$ are Gaussian attributes, $q_{z_j}$ are codebook entries, and $z_j$ is the codebook index [S1].

**Parameters:**
- Codebook size: Not given in the sources.
- Sliding window size: Not given in the sources.

**Edge cases and pitfalls:**
- Quantization precision loss may degrade high-frequency details [G].
- The sliding window mask requires careful tuning to avoid over-pruning essential geometry [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_and_compress(gaussians, codebook_size, window_size, iterations):
    # Initialize codebook with random Gaussian parameters [G]
    codebook = random_sample(gaussians.attributes, codebook_size)
    indices = [0] * len(gaussians)
    
    for i in range(iterations):
        # 1. Pruning via sliding window mask [S4, unofficial]
        gaussians = apply_sliding_window_mask(gaussians, window_size)
        
        # 2. Dynamic pruning based on importance scoring [S2]
        gaussians = prune_low_importance(gaussians)
        
        # 3. Vector quantization (ECVQ/K-means) [S1][S2]
        for j, attr in enumerate(gaussians.attributes):
            best_idx = argmin([norm(attr - c) for c in codebook])
            indices[j] = best_idx
            gaussians.attributes[j] = codebook[best_idx]
            
        # 4. Update codebook via K-means step [S2]
        codebook = update_codebook(gaussians.attributes, indices, codebook_size)
        
        # 5. Standard 3DGS training step: render, compute loss, backpropagate [G]
        rendered_image = render(gaussians, camera)
        loss = compute_loss(rendered_image, ground_truth)
        update_parameters(gaussians, loss)
        
    # 6. Index compression via sorting and RLE-like encoding [S2]
    sorted_indices = sort_indices_by_attribute(indices)
    compressed_indices = run_length_encode(sorted_indices)
    
    return gaussians, codebook, compressed_indices
```

## Subtopics

### SOG (Self-Organizing Gaussian Grids)
SOG organizes 3DGS parameters into a 2D grid to exploit perceptual redundancies and drastically reduce storage requirements [S2]. It uses the PLAS sorting algorithm to arrange parameters while maintaining local neighborhood structure [S2]. A smoothness loss is applied during training to enforce grid smoothness for better compressibility [S2]. Post-processing includes quantization and entropy coding for further compression [S2].

## Operating characteristics
- **Storage:** Achieves approximately $10\times$ smaller storage compared to the original 3DGS representation [S1][S3, unofficial].
- **Quality:** Minimal quality loss is reported [S1][S3, unofficial].
- **Performance:** Not given in the sources.
- **Strengths:** Significant storage reduction while preserving visual fidelity [S1][S3, unofficial].
- **Limitations:** Requires joint optimization of rate and distortion during training, and additional codebook learning steps [S2].

## Background
The method originates from multiple concurrent compression efforts: Deng et al. [188] introduced the sliding window mask to avoid redundant splitting alongside vector quantization [S4, unofficial]. Compact3D [58] formalized vector quantization of Gaussian attributes using K-means and index sorting [S1][S2]. RDO-Gaussian [49] framed the problem as end-to-end rate-distortion optimization using dynamic pruning and ECVQ [S2]. The method is part of the broader effort to make 3DGS practical for storage and transmission [S1][S2].

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29
[S3] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S4] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
