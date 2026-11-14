# GSPull

Also called: GSDF / GSPull
## Overview
GSPull is a technique that leverages neural Signed Distance Functions (SDFs) or Unsigned Distance Functions (UDFs) to recover detailed mesh structures from 3D Gaussian Splatting representations [S1][S2, unofficial]. It addresses the limitation of standard 3DGS, which produces volumetric splats rather than explicit surfaces, by integrating distance-field supervision to refine geometry [S1][S2, unofficial]. Unlike methods that directly pull Gaussians to the zero-level set, GSPull uses dense queries around the zero-level set as strong geometric cues to refine the surface conservatively and gradually, preventing error accumulation from ambiguous pulling directions [S3].

## How it works
GSPull integrates a neural distance field (SDF/UDF) with the Gaussian splatting optimization process. The method jointly finetunes the Gaussian model and the distance field model in a progressive manner [S3].

**Inputs:**
- 3D Gaussian parameters (position, covariance, opacity, color) initialized from standard 3DGS reconstruction.
- Multi-view images for supervision.
- A neural network parameterizing the SDF or UDF.

**Outputs:**
- Refined 3D Gaussians with improved geometric alignment.
- A detailed mesh extracted from the converged distance field zero-level set.

**Data Structures:**
- **Gaussian Set:** Standard 3DGS layout (position $\mathbf{p}$, scale $\mathbf{s}$, rotation $\mathbf{q}$, opacity $\alpha$, spherical harmonics coefficients).
- **Distance Field Network:** A Multi-Layer Perceptron (MLP) mapping 3D coordinates to scalar distance values $d(\mathbf{x})$ [S3].

**Steps:**
1. **Initialization:** Initialize Gaussians using standard 3DGS. Initialize the distance field network [S3].
2. **Joint Optimization:** Perform gradient descent on both the Gaussian parameters and the distance field network parameters [S3].
3. **Dense Querying:** Instead of pulling Gaussians directly to the zero-level set $d(\mathbf{x})=0$, query the distance field at dense points around the current Gaussian positions or surface estimates [S3].
4. **Conservative Refinement:** Use the dense queries as geometric cues to update Gaussian positions and orientations. The refinement is applied gradually to prevent error accumulation [S3].
5. **Progressive Finetuning:** Continuously improve the Gaussian distribution and the distance field representation over time [S3].
6. **Mesh Extraction:** Extract the mesh from the final distance field zero-level set [S1][S2, unofficial].

**Formulas:**
- Distance Field: $d(\mathbf{x})$ represents the signed or unsigned distance from point $\mathbf{x}$ to the surface [S3].
- Loss Function: Combines rendering loss (photometric) and distance field regularization (e.g., Eikonal loss, volume rendering loss for the field) [S3]. Specific loss weights are not given in the sources.

**Edge Cases:**
- **Ambiguous Pulling Directions:** Direct pulling to the zero-level set can cause error accumulation if the gradient direction is ambiguous. GSPull mitigates this by using dense queries [S3].
- **Precision:** The accuracy of the distance field network impacts the final mesh quality [S3].

## Subtopics

### Comparison with Zero-Level Pulling
Unlike methods that directly pull Gaussians to the UDF zero-level set, GSPull leverages dense queries around the zero-level set as strong geometric cues. It refines the surface in a conservative and gradual manner, effectively preventing error accumulation caused by ambiguous pulling directions [S3].

## Operating characteristics
**Time and Memory Complexity:**
- Adds the computational cost of evaluating the neural distance field network for dense queries during optimization [S3].
- Memory usage increases due to the storage of the distance field network parameters [S3].

**Strengths and Limitations:**
- **Strengths:** Recovers more detailed mesh structures compared to standard 3DGS [S1][S2, unofficial]. Prevents error accumulation from ambiguous pulling directions [S3].
- **Limitations:** Requires joint optimization of Gaussians and a neural network, which may be slower than standard 3DGS [S3].

**When to Use:**
- When explicit, high-quality mesh extraction is required from 3DGS [S1][S2, unofficial].
- When standard zero-level pulling leads to geometric artifacts [S3].

## Background
GSPull was introduced as an extension of 3D Gaussian Splatting to enable surface reconstruction. It was inspired by similar approaches used in NeRF-based methods [S1][S2, unofficial]. The technique is often cited alongside GSDF as a method that adopts neural SDFs/UDFs for recovering detailed mesh structures [S1][S2, unofficial]. It is part of the broader effort to enhance 3DGS with geometric priors [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
