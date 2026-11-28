# Gradient-Based Mesh Optimization

Also called: nvdiffrec-style optimization

## Overview
Gradient-Based Mesh Optimization reconstructs unknown 3D geometry by optimizing a parameterized representation (typically a Signed Distance Field or SDF) to minimize geometric discrepancies between rendered multi-view images and a reference mesh [S1, unofficial]. It sets itself apart from its sibling [SDF Optimization via Finite Differences](3086-sdf-optimization-via-finite-differences.md) by operating directly on the mesh topology or SDF values to minimize multiview mask and depth losses, often employing specific regularizers for fabricability or stability [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A randomly initialized SDF field (or mesh parameters), a set of reference meshes with known camera poses, and multi-view images or masks.
- **Outputs:** An optimized SDF field or deformed mesh geometry that minimizes the difference between rendered views and the reference.

**Data Structures:**
- **SDF Field:** A grid or set of vertices where each vertex $v_i$ holds an SDF value $S_i \in \mathbb{R}$ and optionally a deformation vector $\Delta v_i \in \mathbb{R}^3$ [S3].
- **Mesh Representation:** In methods like FlexiCubes, the mesh is defined by vertices, edges, and cubes, with learnable weights (alpha, beta, gamma) controlling the interpolation and splitting of quadrilaterals into triangles [S1, unofficial].

**Steps:**
1. **Initialization:** Initialize the SDF field randomly or with a coarse approximation [S1, unofficial]. If using sphere fitting, determine initial sphere positions and radii by minimizing the volume outside the mesh while encouraging surface coverage [S2].
2. **Rendering:** Render the current geometry (SDF or mesh) from multiple camera viewpoints to generate predicted masks $M'$ and depth maps [S1, unofficial][S3].
3. **Loss Calculation:** Compute the loss between the rendered output and the ground truth reference:
   - **Mask Loss:** L1 norm between ground truth mask $M$ and rendered mask $M'$:
     $$L_{mask} = \frac{|M - M'|}{2}$$ [S3]
   - **Depth Loss:** Difference between rendered depth and reference depth [S1, unofficial].
   - **Regularization:** Add developability regularizer $L_{dev}$ to encourage fabricability from panels [S1, unofficial]. Add Implicit Geometric Regularization (IGR) as an eikonal loss to constrain the SDF gradient magnitude to 1:
     $$L_{eikonal} = E_{v_0 \in V_0} (||\nabla f_\theta (v_0 )||^2 - 1)^2$$ [S3]
4. **Optimization:** Update the SDF values, deformation parameters, or sphere parameters using gradient descent to minimize the total loss [S1, unofficial][S2, S3].
5. **Regularization Adjustment:** Incrementally increase regularization weights if artifacts appear, starting with low weights to avoid hindering convergence [S1, unofficial].

**Parameters:**
- **Regularization Weights:** Initially low; increased if artifacts appear [S1, unofficial].
- **L1 Regularizer on Flexible Weights:** Generally low impact for single shapes but stabilizes generative pipelines [S1, unofficial].
- **Progressive Position Encoding:** Frequency gradually increased after training reaches a certain stage to alleviate high-frequency noise while retaining details [S3].

**Edge Cases and Pitfalls:**
- **High-Frequency Noise:** Direct optimization of SDF values can introduce noise; mitigated by progressive position encoding [S3].
- **Convergence Hindrance:** High regularization weights early in training can prevent convergence [S1, unofficial].

## Subtopics

### [Inverse Mesh Recovery with Spatiotemporal Bilateral Gradient Filtering](13136-inverse-mesh-recovery-with-spatiotemporal-bilateral-gradient.md)
This method reconstructs unknown mesh geometry by optimizing vertex positions to minimize geometric differences between rendered multi-view images and a reference mesh. It sets itself apart from its sibling [Gradient-Based Mesh Optimization](4628-gradient-based-mesh-optimization.md) by replacing the standard Laplacian postfilter with a mesh cross-bilateral filter, which preserves sharp edges and silhouettes better than Laplacian smoothing, leading to faster convergence on thin structures.

### Gradient-Based Mesh Optimization via Sphere Fitting
Approximates non-deformed initial mesh geometry with a set of spheres to compute visibility. The mesh is deformed by translating and rotating these spheres, allowing inverse problems to supervise sphere parameters and update geometry [S2]. Initial sphere positions and radii are determined by minimizing the total volume occupied by the sphere set outside the mesh along with a term encouraging coverage of the mesh surface, optimized via gradient descent [S2]. Sphere centers are connected as an embedded graph to drive mesh deformation, with each sphere connected to its K-nearest vertices on the template [S2]. Mesh deformation is parameterized by the rotation and translation of each sphere, optimized using rendering loss during inverse problems [S2].

### DMTet Surface Reconstruction with Eikonal Regularization
Defines object geometry as a Signed Distance Field (SDF) on a tetrahedral mesh T. Each vertex $v_i$ has an SDF value $S_i \in \mathbb{R}$ and a deformation value $\Delta v_i \in \mathbb{R}^3$. The mesh is updated as $v_i' = v_i + \Delta v_i$ [S3]. Supervised by rendered masks derived from projecting the object according to camera poses, using L1 loss for training [S3]. Employs progressive position encoding to alleviate high-frequency noise while retaining details, gradually increasing encoding frequency after training reaches a certain stage [S3]. Adds Implicit Geometric Regularization (IGR) as an eikonal loss to constrain the gradient magnitude of the SDF field $f_\theta$ to be close to 1 [S3].

## Operating characteristics
- **Time Complexity:** Depends on the number of optimization steps and the complexity of the rendering pipeline (e.g., ray marching for SDFs or rasterization for meshes).
- **Memory Complexity:** Proportional to the size of the SDF grid or mesh vertex count.
- **Strengths:** Can reconstruct complex geometries from multi-view data; differentiable nature allows end-to-end optimization with rendering losses.
- **Limitations:** Sensitive to initialization; prone to high-frequency noise without regularization; computational cost of multi-view rendering.
- **When to Use:** When reconstructing geometry from multi-view images or masks where explicit mesh data is unavailable or incomplete.
- **When Not to Use:** When real-time performance is critical and the scene is simple enough for analytical solutions.

## Background
Originated from workflows for reconstructing unknown meshes by optimizing SDFs to minimize geometric differences against reference meshes [S1, unofficial]. Developed in the context of differentiable rendering and inverse graphics, allowing for the optimization of 3D structures from 2D observations [S1, unofficial][S2, S3]. Used in applications such as 3D reconstruction, shape editing, and generative modeling [S1, unofficial][S2, S3].

## Sources
[S1] GitHub - nv-tlabs/FlexiCubes, https://github.com/nv-tlabs/FlexiCubes, fetched 2026-10-27 (unofficial)
[S2] Efficient and Differentiable Shadow Computation for Inverse Problems, https://doi.org/10.1109/iccv48922.2021.01286, fetched 2026-11-01
[S3] An Index of Refraction Adaptive Neural Refractive Radiance Field for Transparent Scenes, https://doi.org/10.3390/electronics14061073, fetched 2026-10-29
