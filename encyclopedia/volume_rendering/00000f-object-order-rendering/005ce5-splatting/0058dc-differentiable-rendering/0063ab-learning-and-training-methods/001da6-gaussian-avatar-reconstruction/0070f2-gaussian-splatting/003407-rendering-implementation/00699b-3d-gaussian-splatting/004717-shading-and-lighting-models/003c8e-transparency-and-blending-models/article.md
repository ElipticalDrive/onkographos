# Transparency and Blending Models
## Overview
Transparency and Blending Models define how 3D Gaussian primitives are composited into a 2D image, addressing the limitations of standard front-to-back alpha blending which requires strict depth sorting and assumes non-overlapping primitives [G]. The family includes approaches that balance performance and accuracy [3294-hybrid-transparency.md], analytic volumetric integration that eliminates projection artifacts [4440-analytic-transmittance-3dgs.md], and order-independent methods that bypass sorting entirely [7562-sparseoit-rasterizer.md].
## Shared principles
Builds on [Shading and Lighting Models](11924-shading-and-lighting-models.md).
## Subtopics

### [Hybrid Transparency](3294-hybrid-transparency.md)
Combines depth-ordered alpha blending for the foremost $K$ fragments with order-independent accumulation for the tail. Chosen when popping artifacts from approximate sorting must be mitigated without the cost of full sorting or ray tracing.

### [Analytic Transmittance 3DGS](4440-analytic-transmittance-3dgs.md)
Replaces approximate alpha computation with analytic integration of the volume rendering equation along the ray. Chosen for physically accurate transmittance and better representation of opaque surfaces, assuming non-overlapping, sorted Gaussians.

### [SparseOIT Rasterizer](7562-sparseoit-rasterizer.md)
Computes blended images using Order-Independent Transparency (OIT) without depth sorting. Chosen when sorting overhead is prohibitive, using a weighted average to approximate order-dependent results.

## Background
Originating from the need to improve view synthesis quality and rendering stability in 3D Gaussian Splatting, these methods address the linearization errors and sorting artifacts inherent in standard rasterization [G]. Analytic Transmittance derives from volumetric rendering theory, applying 1D Gaussian integration along rays [4440-analytic-transmittance-3dgs.md]. Hybrid Transparency and SparseOIT address the computational bottlenecks of sorting large numbers of fragments in real-time rendering [3294-hybrid-transparency.md, 7562-sparseoit-rasterizer.md].
## Sources
