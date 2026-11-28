# GSPull

Also called: GSDF / GSPull
## Overview
GSPull is a technique that leverages neural distance functions (SDFs or UDFs) to recover detailed mesh structures from 3D Gaussian Splatting representations [S1][S2, unofficial]. It addresses the limitation of standard 3DGS, which produces volumetric splats rather than explicit surfaces, by integrating distance-field supervision to refine geometry [S1][S2, unofficial]. Unlike methods that directly pull Gaussians to the zero-level set, GSPull uses dense queries around the zero-level set as strong geometric cues to refine the surface conservatively and gradually, preventing error accumulation from ambiguous pulling directions [S3].

## How it works
**Inputs:** 3D Gaussian parameters (position, covariance, opacity, color) initialized from standard 3DGS reconstruction [G]. Multi-view images for supervision [G]. A neural network parameterizing the distance field [G].
**Outputs:** Refined 3D Gaussians with improved geometric alignment [G]. A detailed mesh extracted from the converged distance field zero-level set [G].
**Data Structures:** The Gaussian set follows the standard 3DGS layout containing position $\mathbf{p}$, scale $\mathbf{s}$, rotation $\mathbf{q}$, opacity $\alpha$, and spherical harmonics coefficients [G]. The distance field is stored as a neural network mapping 3D coordinates to scalar distance values $d(\mathbf{x})$ [G].
**Steps:** 1. Initialize Gaussians using standard 3DGS and initialize the distance field network [G]. 2. Jointly optimize the Gaussian parameters and distance field network parameters via gradient descent [S3]. 3. Query the distance field at dense points around the current Gaussian positions or surface estimates instead of pulling directly to the zero-level set $d(\mathbf{x})=0$ [S3]. 4. Use the dense queries as geometric cues to update Gaussian positions and orientations, applying the refinement gradually [S3]. 5. Progressively finetune the Gaussian distribution and distance field representation over time [S3].
**Formulas:** The distance field $d(\mathbf{x})$ represents the signed or unsigned distance from point $\mathbf{x}$ to the surface [G].
**Parameters:** Not given in the sources.
**Edge Cases:** Ambiguous pulling directions can cause error accumulation if gradients are unclear; GSPull mitigates this by relying on dense queries rather than direct zero-level set pulling [S3].

## Subtopics

### Comparison with Zero-Level Pulling
Unlike methods that directly pull Gaussians to the UDF zero-level set, GSPull leverages dense queries around the zero-level set as strong geometric cues [S3]. It refines the surface in a conservative and gradual manner, effectively preventing error accumulation caused by ambiguous pulling directions [S3].

## Operating characteristics
**Time and Memory Complexity:** Evaluating the neural distance field network for dense queries adds computational cost during optimization [G]. Memory usage increases due to the storage of the distance field network parameters [G]. Measured performance, hardware specifics, resolution scaling, and scene dependencies are Not given in the sources.
**Strengths and Limitations:** GSPull recovers more detailed mesh structures compared to standard 3DGS [S1][S2, unofficial]. It prevents error accumulation from ambiguous pulling directions [S3]. The method requires joint optimization of Gaussians and a neural network [S3].
**When to Use:** Use when explicit, high-quality mesh extraction is required [S1][S2, unofficial]. Use when standard zero-level pulling leads to geometric artifacts [S3].

## Background
GSPull was introduced as an extension of 3D Gaussian Splatting to enable surface reconstruction [S1][S2, unofficial]. It was inspired by similar approaches used in NeRF-based methods [S1][S2, unofficial]. The technique is often cited alongside GSDF as a method that adopts neural distance functions for recovering detailed mesh structures [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
