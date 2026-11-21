# Fast Compression of 3D Gaussian Splatting (FCGS)
Also called: Fast Feedforward 3D Gaussian Splatting Compression

## Overview
FCGS is an optimization-free, feed-forward compression pipeline for 3D Gaussian Splatting (3DGS) that compresses representations in a single pass without per-scene finetuning, aiming to reduce compression time from minutes to seconds while maintaining fidelity [S1, unofficial]. It distinguishes itself from siblings like [Model Compression Pipeline for 3D Gaussian Splatting](9987-model-compression-pipeline-for-3d-gaussian-splatting.md) and [3DGS Model Compression Pipeline](10101-3dgs-model-compression-pipeline.md) by eliminating the need for iterative per-scene optimization, which typically takes 10–28 minutes, by using an autoencoder-based structure with a Multi-path Entropy Module (MEM) to balance compression size and fidelity [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: A trained 3DGS representation consisting of unstructured Gaussian blobs with attributes (position, scale, rotation, opacity, color, spherical harmonics coefficients).
- Output: A compressed bitstream representing the latent space of the Gaussian attributes.

**Data structures:**
The method treats Gaussian attributes as a set of unstructured blobs. The core data structure is the latent vector $\mathbf{z}$ derived from the analysis transform.
- Attributes $\mathbf{x}$: The raw Gaussian parameters.
- Latent $\mathbf{y}$: The output of the analysis transform $g_a(\mathbf{x})$.
- Hyperprior $\mathbf{z}$: The output of the hyperanalysis transform, used to model the entropy of $\mathbf{y}$.
- Quantized Latent $\hat{\mathbf{y}}$: The discretized latent representation.

**Method steps:**
1. **Analysis Transform:** Map input Gaussian attributes $\mathbf{x}$ into a latent space $\mathbf{y}$ using an analysis transform network $g_a$. This step is inspired by image compression autoencoders [S1, unofficial].
2. **Multi-path Entropy Module (MEM):** Determine the processing path for each attribute group based on sensitivity.
   - If an attribute is geometry-sensitive (e.g., position, scale, rotation), it bypasses the autoencoder and is directly quantized: $\mathbf{y} = \mathbf{x}$ [S1, unofficial].
   - If an attribute is color-sensitive (e.g., SH coefficients, opacity), it passes through the autoencoder: $\mathbf{y} = g_a(\mathbf{x})$ [S1, unofficial].
   - This prevents model collapse caused by geometry deviations that would break rasterization [S1, unofficial].
3. **Quantization:** Quantize the latent variables $\mathbf{y}$ to obtain $\hat{\mathbf{y}}$.
4. **Context Modeling:** Eliminate redundancies among parameters using customized context models:
   - **Inter-Gaussian Context:** Models dependencies between different Gaussians.
   - **Intra-Gaussian Context:** Models dependencies within the parameters of a single Gaussian.
   - These context models are lightweight and efficient [S1, unofficial].
5. **Hyperprior Encoding:** Encode the quantized latent $\hat{\mathbf{y}}$ into a hyperprior $\mathbf{z}$ using a hyperanalysis transform to estimate the probability distribution for entropy coding [S1, unofficial].
6. **Entropy Coding:** Encode the residuals and hyperprior using the estimated probabilities to produce the final bitstream [S1, unofficial].
7. **Decoding (Reverse):**
   - Decode the bitstream to recover $\hat{\mathbf{y}}$ and $\hat{\mathbf{z}}$.
   - Use the hyperprior $\hat{\mathbf{z}}$ to predict the distribution of $\hat{\mathbf{y}}$.
   - Apply the synthesis transform $g_s$ (if applicable for color attributes) to reconstruct the attributes.
   - Directly use the quantized geometry attributes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fcg_compress(gaussians):
    """
    Compress 3DGS using FCGS.
    gaussians: dict or object containing attributes:
        - pos: (N, 3)
        - scale: (N, 3)
        - rot: (N, 4)
        - opacity: (N, 1)
        - sh_coeffs: (N, K, 3)
    """
    # Step 1 & 2: Analysis and MEM Path Selection
    # Geometry attributes are directly quantized to preserve rasterization integrity
    pos_q = quantize(gaussians.pos)
    scale_q = quantize(gaussians.scale)
    rot_q = quantize(gaussians.rot)
    
    # Color attributes pass through the autoencoder
    # y = g_a(x) for color attributes
    color_latent = analysis_transform(gaussians.opacity, gaussians.sh_coeffs)
    color_latent_q = quantize(color_latent)
    
    # Step 3 & 4: Context Modeling
    # Inter- and intra-Gaussian context models are applied to predict probabilities
    # This step is implicit in the entropy coding process using the context models
    # h_inter and h_intra are used to condition the probability distribution
    
    # Step 5: Hyperprior
    hyperprior = hyperanalysis_transform(color_latent_q)
    hyperprior_q = quantize(hyperprior)
    
    # Step 6: Entropy Coding
    # Encode quantized values using the context models and hyperprior
    bitstream = entropy_encode(
        pos=pos_q,
        scale=scale_q,
        rot=rot_q,
        color_latent=color_latent_q,
        hyperprior=hyperprior_q,
        context_models=get_context_models() # Inter and Intra
    )
    return bitstream

def fcg_decompress(bitstream):
    """
    Decompress 3DGS using FCGS.
    """
    # Step 6 (Reverse): Entropy Decoding
    pos_q, scale_q, rot_q, color_latent_q, hyperprior_q = entropy_decode(
        bitstream,
        context_models=get_context_models()
    )
    
    # Step 2 (Reverse): MEM Path Reversal
    # Geometry attributes are directly used
    pos = dequantize(pos_q)
    scale = dequantize(scale_q)
    rot = dequantize(rot_q)
    
    # Color attributes pass through synthesis transform
    # x = g_s(y) for color attributes
    opacity, sh_coeffs = synthesis_transform(color_latent_q, hyperprior_q)
    
    return {
        'pos': pos,
        'scale': scale,
        'rot': rot,
        'opacity': opacity,
        'sh_coeffs': sh_coeffs
    }
```

**Parameters:**
- Compression Ratio: Over 20× [S1, unofficial].
- Compression Time: Seconds (compared to 10-28 minutes for optimization-based methods) [S1, unofficial].

**Edge cases and pitfalls:**
- **Geometry Sensitivity:** Directly quantizing geometry attributes is critical; processing them through the autoencoder leads to model collapse because rasterization fails due to deviations [S1, unofficial].
- **Color Attributes:** Allowing all color attributes to be processed by the autoencoder without fixing geometry can lead to densification issues and suboptimal fidelity [S1, unofficial].

## Subtopics

### FCGS with Pruning
A technique to boost compression performance by applying FCGS to 3DGS that have been pruned using techniques like Trimming or Mini-Splatting [S1, unofficial]. When combined with pruning (e.g., Mini**+Ours on Deep Blending dataset), FCGS achieves a compression ratio of 100× over vanilla 3DGS [S1, unofficial]. The size of compressed 3DGS using FCGS (at the highest rate) is only 40% that of MSC (Model Compression Pipeline for 3D Gaussian Splatting) with the same fidelity [S1, unofficial].

## Operating characteristics
- **Time Complexity:** O(1) per scene in terms of optimization iterations, as it is a single feed-forward pass. Compression time is reduced from minutes to seconds [S1, unofficial].
- **Memory Complexity:** Depends on the autoencoder and context model sizes, but described as lightweight and efficient [S1, unofficial].
- **Measured Performance:** Achieves a compression ratio of over 20× while maintaining fidelity, surpassing most SOTA per-scene optimization-based methods [S1, unofficial]. When combined with pruning, it achieves 100× compression ratio [S1, unofficial].
- **Strengths:** Fast, optimization-free, high compression ratio, compatible with pruning techniques.
- **Limitations:** May have lower fidelity than optimization-based methods if not carefully tuned, though it surpasses most SOTA methods [S1, unofficial].
- **When to use:** When rapid compression is required, or when per-scene optimization is computationally prohibitive.
- **When not to use:** When maximum possible fidelity is required and optimization time is not a constraint.

## Background
FCGS was introduced in 2024 by Yihang Chen et al. as a solution to the storage challenges posed by the vast number of Gaussians in 3DGS [S1, unofficial]. It is inspired by image compression methods, specifically autoencoder-based structures like those proposed by Ballé et al. (2018) [S1, unofficial]. The method addresses the limitation of previous compression techniques that required per-scene optimization, making them sluggish and slow [S1, unofficial].

## Sources
[S1] Fast Feedforward 3D Gaussian Splatting Compression, https://doi.org/10.48550/arxiv.2410.08017, fetched 2026-10-26 (unofficial)
