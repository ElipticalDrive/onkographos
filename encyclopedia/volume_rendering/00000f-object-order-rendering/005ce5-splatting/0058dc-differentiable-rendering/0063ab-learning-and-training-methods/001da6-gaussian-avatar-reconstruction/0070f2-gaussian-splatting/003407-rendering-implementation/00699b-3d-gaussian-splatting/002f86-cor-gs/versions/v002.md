# CoR-GS


## Overview
CoR-GS is a method that improves sparse-view reconstruction in 3D Gaussian Splatting by defining two independent reconstructions and comparing them to discard inaccurate Gaussians [S1][S2, unofficial].

## How it works
**Inputs:** Sparse set of input views (images and camera poses) [G].
**Outputs:** Refined set of 3D Gaussians with improved geometric and photometric consistency [G].
**Data Structures:** Two independent sets of Gaussian parameters (position, covariance, opacity, spherical harmonic coefficients) are maintained during the comparison phase [G].
**Steps:**
1. **Dual Reconstruction:** Train two independent Gaussian fields from the same sparse input views [S1][S2, unofficial].
2. **Comparison:** Render views from both reconstructions and compare the results to identify discrepancies [S1][S2, unofficial].
3. **Discard Inaccurate Gaussians:** Gaussians that contribute to inconsistent rendering results between the two fields are identified as inaccurate and removed [S1][S2, unofficial].
4. **Refinement:** The remaining Gaussians form the final, consistent representation [G].
**Parameters:** Not given in the sources.
**Edge Cases and Pitfalls:** Not given in the sources.

## Subtopics

### [3D Gaussian Splatting](1795-3d-gaussian-splatting.md)
Base method for differentiable volume rendering using 3D Gaussians. Chosen when high-fidelity novel view synthesis is required.

### [Density Control and Densification](5337-density-control-and-densification.md)
Manages Gaussian population via splitting and pruning to adapt to scene complexity. Chosen to balance detail and computational cost.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Improves sparse-view reconstruction quality by removing inconsistent Gaussians [S1][S2, unofficial].
- **Limitations:** Not given in the sources.
- **When to use:** Sparse-view reconstruction where standard 3DGS yields inconsistent or noisy results [S1][S2, unofficial].
- **When not to use:** Not given in the sources.

## Background
CoR-GS is referenced in surveys discussing the evolution from volume rendering to 3D Gaussian Splatting, positioned alongside feed-forward network (FFN)-based methods and MAtCha [S1][S2, unofficial]. It was introduced to improve sparse-view reconstruction by defining two independent reconstructions and comparing them to discard inaccurate Gaussians [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
