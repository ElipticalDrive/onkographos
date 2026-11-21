# C3DGS

Also called: Compact 3D Gaussian Splatting

## Overview
C3DGS is a compression method for 3D Gaussian Splatting that reduces storage requirements by applying residual vector quantization (R-VQ) to geometric attributes, specifically scaling and rotation [S1][S2, unofficial]. It distinguishes itself from siblings like Mini-Splatting by focusing on attribute quantization rather than count reduction via sampling [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained set of $N$ 3D Gaussians with full-precision attributes: position $\mathbf{p}$, rotation $\mathbf{R}$, scale $\mathbf{s}$, opacity $\alpha$, and spherical harmonic coefficients $\mathbf{c}_{lm}$.
*   **Output:** A compressed representation using codebooks and residuals, achieving a scene size of 48.8 MB on the MipNeRF360 dataset [S1].

**Data Structures:**
1.  **Codebook:** A finite set of representative vectors for scaling and rotation.
2.  **Indices:** Quantized indices pointing to entries in the codebook.
3.  **Residuals:** Small correction values stored for each Gaussian to recover precision.

**Method Steps:**
1.  **Attribute Selection:** Identify geometric attributes for quantization: scaling $\mathbf{s}$ and rotation $\mathbf{R}$ [S1][S2, unofficial].
2.  **Residual Vector Quantization (R-VQ):**
    *   Cluster the geometric attributes into a codebook using R-VQ [S1][S2, unofficial].
    *   For each Gaussian, find the nearest codebook entry.
    *   Compute the residual between the original attribute and the codebook entry.
    *   Store the codebook index and the residual.
3.  **Non-Quantized Attributes:** Position $\mathbf{p}$ and opacity $\alpha$ are not quantized. Sharing these attributes across Gaussians would result in overlapping Gaussians, which degrades quality [S1][S2, unofficial].
4.  **Reconstruction:** During rendering, retrieve the codebook entry and add the residual to reconstruct the full-precision attribute [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_c3dgs(gaussians):
    # gaussians: list of objects with attributes p, R, s, alpha, c_lm
    codebook_s = build_codebook([g.s for g in gaussians], method='R-VQ')
    codebook_R = build_codebook([g.R for g in gaussians], method='R-VQ')
    
    compressed_gaussians = []
    for g in gaussians:
        # Find nearest codebook entry
        idx_s = find_nearest(codebook_s, g.s)
        idx_R = find_nearest(codebook_R, g.R)
        
        # Compute residuals
        res_s = g.s - codebook_s[idx_s]
        res_R = g.R - codebook_R[idx_R]
        
        # Store compressed data
        compressed_gaussians.append({
            'p': g.p,
            'alpha': g.alpha,
            'c_lm': g.c_lm,
            'idx_s': idx_s,
            'idx_R': idx_R,
            'res_s': res_s,
            'res_R': res_R
        })
    return compressed_gaussians, codebook_s, codebook_R

def decompress_c3dgs(compressed_gaussians, codebook_s, codebook_R):
    gaussians = []
    for cg in compressed_gaussians:
        s = codebook_s[cg['idx_s']] + cg['res_s']
        R = codebook_R[cg['idx_R']] + cg['res_R']
        gaussians.append({
            'p': cg['p'],
            'R': R,
            's': s,
            'alpha': cg['alpha'],
            'c_lm': cg['c_lm']
        })
    return gaussians
```

**Parameters:**
*   Codebook size: Not given in the sources.
*   Residual precision: Not given in the sources.

**Edge Cases:**
*   Overlapping Gaussians: Avoid quantizing position and opacity to prevent this [S1][S2, unofficial].

## Operating characteristics
*   **Size:** 48.8 MB on MipNeRF360 [S1].
*   **Quality:** SSIM 0.798, PSNR 27.08, LPIPS 0.247 on MipNeRF360 [S1].
*   **Comparison:** Outperforms 3DGS (750 MB) and Compact3D (not given) in size, while maintaining competitive quality [S1].

## Background
C3DGS was introduced in [54] (cited as [S1][S2, unofficial]). It builds on the principle of vector quantization in signal processing [S1][S2, unofficial].

## Sources

[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
