# Octree-based hierarchical structures
## Overview
This family covers methods that represent volume data using recursive 3D space subdivision into octants to enable adaptive resolution [G]. It concentrates detail in regions of interest, avoiding the prohibitive storage costs of uniform grids for complex multi-scale volumetric data [G].
## Shared principles
All methods in this family share the following mechanisms and mathematical foundations:
1. **Unstructured Topology:** Data is stored in elements (tetrahedra, hexahedra, cells) defined by arbitrary vertex positions, rather than a regular lattice. [G]
2. **Adaptive Resolution:** The domain is partitioned hierarchically or locally refined based on error metrics, gradient magnitudes, or feature density. [G]
3. **Basis Function Evaluation:** Scalar values at query points are computed using basis functions defined over the elements. [G]
4. **Ray-Element Intersection:** Rendering requires determining the entry and exit points of rays through element shapes. [G]
5. **Continuity and Regularization:** In differentiable rendering contexts, maintaining mesh validity (non-inverted elements) is critical. [G]
## Subtopics

### [Adaptive Octree Construction with 2:1 Balance and Surface Propagation](830-adaptive-octree-construction-with-2-1-balance-and-surface-pr.md)
Constructs an adaptive octree for implicit surfaces, enforcing a 2:1 balance constraint and using Dual Contouring for mesh extraction. Sets itself apart by producing a watertight, manifold polygonal mesh from implicit functions, unlike KD-tree or tetrahedral AMR methods. Choose when a geometrically accurate, isotopic mesh is required.

### [Octree-based AMR Data Splitting](12260-octree-based-amr-data-splitting.md)
Partitions hierarchical AMR data into uniform, rectilinear blocks by overlaying an octree aligned to the finest grid resolution. Sets itself apart by generating a static, view-independent structure that maps directly to uniform blocks for hardware-accelerated texture uploads. Choose when parallel rendering requires uniform grid blocks from unstructured AMR data.

### [Octree AMR Model Setup](19681-octree-amr-model-setup.md)
Configures volume rendering or radiative transfer models to use adaptive octree grids with stochastic refinement criteria based on local physical properties. Sets itself apart by focusing on initialization of refinement criteria and model function signatures rather than geometric construction. Choose when initializing radiative transfer models with physics-based refinement.

## Background
Not given in the sources.
## Sources
