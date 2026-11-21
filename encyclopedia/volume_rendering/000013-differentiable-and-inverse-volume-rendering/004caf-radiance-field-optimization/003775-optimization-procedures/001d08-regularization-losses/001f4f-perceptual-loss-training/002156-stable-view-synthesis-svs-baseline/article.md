# Stable View Synthesis (SVS) Baseline

Also called: SVS Baseline, SVS*, Stable View Synthesis (SVS)

## Overview
Stable View Synthesis (SVS) is a view synthesis method that blends training images together to render novel views, relying on proxy geometry from an external system [S1]. It is distinguished from its siblings by its explicit dependence on external proxy geometry (typically piecewise planar meshes) and access to all training images, whereas [High-Frequency Aware Loss](7589-high-frequency-aware-loss.md) and [WD-Regularized Loss](15613-wd-regularized-loss.md) operate within different optimization frameworks without this specific blending-and-proxy requirement [S1].

## How it works
SVS is trained using a perceptual loss [S1]. The method operates by blending training images together to render novel views [S1].

**Inputs and Outputs:**
*   **Inputs:** A set of training images; proxy geometry produced by an external system [S1].
*   **Outputs:** Rendered novel views [S1].

**Data Structures:**
Not given in the sources.

**Steps:**
1.  Obtain proxy geometry from an external system [S1].
2.  Train the model using a perceptual loss [S1].
3.  Render novel views by blending training images together [S1].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
SVS may fail when the external proxy geometry is incorrect [S1]. It struggles particularly with distant or unobserved content, such as trees, where the proxy geometry reconstruction (e.g., via COLMAP) is poor [S1].

**Pseudocode:**
Not given in the sources.

## Subtopics

### High-Frequency Aware Loss
Improves perceptual quality by targeting high-frequency details. Choose when SVS fails on natural content due to proxy geometry errors. See [High-Frequency Aware Loss](7589-high-frequency-aware-loss.md).

### WD-Regularized Loss
Uses weight decay regularization for stability. Choose when SVS's reliance on external geometry is unavailable. See [WD-Regularized Loss](15613-wd-regularized-loss.md).

## Operating characteristics
SVS requires multiple large CNNs and access to all training images to render novel views [S1]. It is better suited to large piecewise planar objects (e.g., vehicles) due to its reliance on piecewise planar mesh proxy geometry, whereas it performs poorly on scenes with natural content (trees, grass, flowers) [S1].

## Background
Stable View Synthesis was introduced by Riegler and Koltun in CVPR 2021 [S2, unofficial].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
