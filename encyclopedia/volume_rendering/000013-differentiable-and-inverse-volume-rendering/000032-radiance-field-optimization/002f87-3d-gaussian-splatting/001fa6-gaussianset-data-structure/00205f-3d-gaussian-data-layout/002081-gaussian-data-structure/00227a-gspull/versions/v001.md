# GSPull
Also called: GSDF / GSPull
## Overview
GSPull is a method that leverages neural signed distance functions (SDFs) for surface reconstruction to recover more detailed mesh structures from 3D Gaussian Splatting representations [S1][S2, unofficial]. It sets itself apart from standard 3DGS by integrating SDF-based regularization, inspired by NeRF-based surface reconstruction techniques, to address the lack of explicit geometry in volumetric splatting [S1][S2, unofficial].
## How it works
Inputs and outputs: The method takes 3D Gaussian parameters (position, covariance, opacity, color) as input and outputs a refined mesh structure via an SDF field [S1][S2, unofficial].
Data structures: Not given in the sources.
Numbered steps:
1. Initialize 3D Gaussians from input images [S1][S2, unofficial].
2. Integrate a neural SDF network to represent the surface geometry, regularizing the Gaussian distribution to align with the zero-level set of the SDF [S1][S2, unofficial].
3. Optimize the Gaussian parameters and SDF network jointly to minimize rendering loss and SDF consistency loss [S1][S2, unofficial].
4. Extract the mesh using marching cubes or similar algorithms on the learned SDF [S1][S2, unofficial].
Formulas: Not given in the sources.
Parameters: Not given in the sources.
Edge cases and pitfalls: Not given in the sources.
Pseudocode: Not given in the sources.
## Subtopics
## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Not given in the sources.
Scaling: Not given in the sources.
Strengths and limitations: Strengths include recovering more detailed mesh structures compared to standard 3DGS [S1][S2, unofficial]. Limitations are not specified in the sources.
When to use it and when not: Use when explicit, detailed mesh geometry is required from Gaussian splatting [S1][S2, unofficial]. Not given in the sources for when not to use.
## Background
Origin: GSPull was introduced as an extension of 3D Gaussian Splatting techniques, drawing inspiration from NeRF-based surface reconstruction methods [S1][S2, unofficial].
History and development: It follows the trend of adapting 3DGS for surface reconstruction, similar to GSDF [S1][S2, unofficial].
Where it is used: Surface reconstruction and mesh generation from 3D Gaussian representations [S1][S2, unofficial].
## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
