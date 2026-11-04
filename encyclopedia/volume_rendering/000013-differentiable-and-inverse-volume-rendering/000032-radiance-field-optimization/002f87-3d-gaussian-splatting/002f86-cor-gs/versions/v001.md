# CoR-GS

Also called: Consistency Regularization for Gaussian Splatting

## Overview
CoR-GS is a method designed to improve sparse-view reconstruction in 3D Gaussian Splatting by defining two independent reconstructions and comparing them to discard inaccurate Gaussians [S1][S2, unofficial]. It addresses the ambiguity and instability inherent in reconstructing scenes from minimal view overlap, where standard optimization often yields noisy or inconsistent geometry [S1][S2, unofficial].

## How it works
CoR-GS operates by enforcing consistency between two distinct Gaussian field representations derived from the same sparse input views. The core mechanism involves generating two independent reconstructions and evaluating their agreement to identify and remove unreliable Gaussians.

**Inputs:**
- Sparse set of input views (images and camera poses).
- Initial set of 3D Gaussians (typically from a standard 3DGS initialization).

**Outputs:**
- Refined set of 3D Gaussians with improved geometric and photometric consistency.

**Data Structures:**
- Two independent sets of Gaussian parameters (position, covariance, opacity, SH coefficients) are maintained during the comparison phase.
- A mask or filter is applied to the union of Gaussians to discard those failing the consistency check.

**Steps:**
1. **Dual Reconstruction:** Train two independent Gaussian fields from the same sparse input views. These fields are initialized separately or optimized with different random seeds/initializations to ensure independence [S1][S2, unofficial].
2. **Comparison:** Render views from both reconstructions and compare the results. The comparison metric identifies discrepancies between the two fields [S1][S2, unofficial].
3. **Discard Inaccurate Gaussians:** Gaussians that contribute to inconsistent rendering results between the two fields are identified as inaccurate and removed [S1][S2, unofficial].
4. **Refinement:** The remaining Gaussians form the final, consistent representation.

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- Sparse view overlap: The method is specifically designed for scenarios with minimal view overlap, where scale ambiguity is high [S1][S2, unofficial].
- Inaccurate Gaussians: The method relies on the ability to distinguish accurate from inaccurate Gaussians via the consistency check. If both reconstructions are equally poor, the method may not improve results.

## Subtopics

### [3D Gaussian Splatting](1795-3d-gaussian-splatting.md)
The base method for which CoR-GS provides a consistency regularization strategy.

### [Density Control and Densification](5337-density-control-and-splatting.md)
CoR-GS acts as a pruning mechanism, complementing densification strategies by removing unreliable Gaussians.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Improves novel view synthesis quality in sparse-view scenarios, particularly with minimal view overlap [S1][S2, unofficial].
- **Limitations:** Requires maintaining two independent reconstructions, potentially increasing computational cost during the training phase.
- **When to use:** Sparse-view reconstruction where standard 3DGS yields inconsistent or noisy results.
- **When not to use:** Dense-view scenarios where standard 3DGS performs well.

## Background
CoR-GS was introduced in the context of improving sparse-view reconstruction for 3D Gaussian Splatting. It is cited in surveys discussing the evolution from volume rendering to 3D Gaussian Splatting, highlighting its role in addressing scale ambiguity and improving quality in challenging sparse-view settings [S1][S2, unofficial]. The method is referenced alongside other approaches like FFN-based methods and MAtCha, positioning it as a regularization technique rather than a generative model [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
