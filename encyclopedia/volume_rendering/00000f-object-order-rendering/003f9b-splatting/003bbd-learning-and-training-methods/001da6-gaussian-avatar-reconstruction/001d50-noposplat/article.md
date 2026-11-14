# NoPoSplat

Also called: No-Pose Gaussian Splatting

## Overview
NoPoSplat is a method for 3D Gaussian Splatting that eliminates the requirement for known camera poses during training [S1][S2, unofficial]. It solves the problem of pose ambiguity by anchoring the scene representation to a canonical coordinate system derived from a single input view and introducing intrinsics token embeddings to resolve scale ambiguity [S1][S2, unofficial]. This approach achieves superior novel view synthesis quality, particularly in scenarios with minimal view overlap, distinguishing it from methods that rely on pre-calibrated camera parameters [S1][S2, unofficial].

## How it works
**Inputs:**
- A set of input images (typically sparse views).
- No known camera poses or intrinsics are required as input [S1][S2, unofficial].

**Outputs:**
- A set of 3D Gaussians defined in a canonical coordinate system, including position, covariance, opacity, and spherical harmonics coefficients [S1][S2, unofficial].

**Data Structures:**
- **Canonical Coordinate System:** A fixed reference frame anchored to the geometry of the single input view used as the reference [S1][S2, unofficial].
- **Intrinsics Token Embeddings:** Learnable embeddings that encode camera intrinsic parameters, allowing the model to infer scale and projection geometry without explicit pose input [S1][S2, unofficial].

**Steps:**
1. **Anchor Initialization:** Establish a canonical coordinate system anchored to the geometry observed in a single input view [S1][S2, unofficial].
2. **Intrinsics Encoding:** Generate intrinsics token embeddings to represent camera intrinsic parameters [S1][S2, unofficial].
3. **Scale Resolution:** Use the intrinsics token embeddings to resolve the scale ambiguity inherent in monocular or sparse-view reconstruction [S1][S2, unofficial].
4. **Gaussian Optimization:** Optimize the 3D Gaussian parameters (position, covariance, opacity, SH) in the canonical coordinate system to minimize rendering error against the input views [S1][S2, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
- **Scale Ambiguity:** Without known poses, scale ambiguity is a critical issue; NoPoSplat addresses this via intrinsics token embeddings [S1][S2, unofficial].
- **Minimal View Overlap:** The method is specifically noted for superior performance in scenarios with minimal view overlap [S1][S2, unofficial].

## Subtopics
No subtopics.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
- **Strengths:** Eliminates the need for known camera poses; achieves superior novel view synthesis quality in scenarios with minimal view overlap [S1][S2, unofficial].
- **Limitations:** Not given in the sources.

**When to Use:**
Use when camera poses are unknown or unreliable, especially in sparse-view settings with minimal overlap [S1][S2, unofficial].

**When Not to Use:**
Not given in the sources.

## Background
NoPoSplat was introduced to address critical limitations in existing Gaussian Splatting methods that require known camera poses [S1][S2, unofficial]. It builds on the principles of differentiable rendering and Gaussian representation but diverges by removing the pose dependency [S1][S2, unofficial]. The method is cited in recent surveys on 3D Gaussian Splatting as a key advancement in pose-free reconstruction [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
