# Parametric Surface Density Model

Also called: 3D object model with six parameters

## Overview
The Parametric Surface Density Model is a 3D object representation defined by six scalar quantities: three separating surfaces and the density fields defined on these interfaces [S1, unofficial]. It solves the problem of reconstructing volumetric objects from limited data by explicitly modeling the boundaries and the material properties within those boundaries, rather than storing a dense 3D grid [S1, unofficial]. Unlike its siblings which often rely on implicit distance fields or voxel grids, this method separates the geometric definition (the surfaces) from the volumetric definition (the densities), allowing for adaptive elliptic operators to fill the volume [S1, unofficial].

## How it works
**Inputs:**
*   A set of example deformations or initial surface configurations.
*   Data defining the three planar curves that generate the separating surfaces.

**Outputs:**
*   A 3D volumetric representation defined by the three separating surfaces and the interpolated density fields.

**Data Structures:**
*   **Separating Surfaces:** Represented by their planar generators (curves) rotated around an axis. Each surface is defined by a parametric curve $c_i(t)$ where $i \in \{1, 2, 3\}$.
*   **Density Fields:** Scalar fields $\rho_i(\mathbf{p})$ defined on the surface $S_i$.
*   **Volumetric Field:** A 3D scalar field $V(\mathbf{p})$ reconstructed from the surface densities.

**Steps:**
1.  **Define Separating Surfaces:** Construct three separating surfaces $S_1, S_2, S_3$ by rotating three planar curves around a common axis. These surfaces partition the 3D space into distinct regions [S1, unofficial].
2.  **Define Surface Densities:** Assign a density field $\rho_i$ to each surface $S_i$. These fields represent the material properties or intensity values at the interfaces [S1, unofficial].
3.  **Reconstruct Volumetric Field:** Fill the 3D volume with a scalar field based on the surface densities. This is achieved using an adaptive elliptic operator. The operator ensures smooth interpolation between the densities defined on the separating surfaces, effectively "filling" the volume [S1, unofficial].
    *   Let $V(\mathbf{p})$ be the reconstructed field. The operator solves a partial differential equation (PDE) such that $V(\mathbf{p})$ matches $\rho_i$ on $S_i$ and is smooth elsewhere.
    *   The specific form of the elliptic operator is adaptive, meaning its coefficients may vary based on the local geometry or density gradients [S1, unofficial].
4.  **Iterative Refinement (Optional):** For tomographic reconstruction, iterate the 1D approach across slices, adding a smoothing term between slices to ensure consistency [S1, unofficial].

**Parameters:**
*   Number of separating surfaces: 3 [S1, unofficial].
*   Number of density fields: 3 [S1, unofficial].
*   Axis of rotation: Defined by the planar curves' orientation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Curves:** If the planar curves do not generate valid closed surfaces, the separation of space may be ill-defined.
*   **Density Discontinuities:** Sharp changes in $\rho_i$ may require a higher-order elliptic operator to avoid artifacts in the reconstructed volume [S1, unofficial].
*   **Precision:** The adaptive elliptic operator requires careful numerical implementation to maintain stability, especially near the boundaries [S1, unofficial].

## Subtopics

### PCA-based Surface Deformation
This variant addresses the deformation of the separating surfaces (the planar generators) using Principal Component Analysis (PCA) [S1, unofficial]. It is chosen when the object's shape varies across a set of examples and a compact basis of deformations is needed.
*   **Definition:** Deformation is characterized by finding an optimal basis of deformations obtained via PCA on a set of examples [S1, unofficial].
*   **Steps:**
    1.  Construct a random example set where realizations are obtained by integrating the solution of a linear stochastic differential equation [S1, unofficial].
    2.  Apply PCA to the set of example deformations to obtain a basis [S1, unofficial].
    3.  Search for an optimal basis from the PCA results to characterize surface deformation [S1, unofficial].
*   **When to choose:** Use this when the object exhibits natural variations in shape that can be modeled as stochastic processes, allowing for efficient representation and generation of new shapes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The iterative reconstruction and PDE solving steps likely dominate, depending on the resolution of the adaptive elliptic operator.
*   **Memory Complexity:** Proportional to the number of parameters defining the planar curves and the density fields, plus the storage for the reconstructed 3D volume if explicitly stored. It is more memory-efficient than dense voxel grids for simple shapes but less efficient than pure implicit functions for complex scenes.
*   **Strengths:** Explicitly models boundaries and internal properties; allows for adaptive smoothing; suitable for tomographic reconstruction from few views [S1, unofficial].
*   **Limitations:** Requires defining three specific surfaces; complex to implement the adaptive elliptic operator; limited to objects that can be separated by three rotating planar curves [S1, unofficial].
*   **When to use:** Ideal for reconstructing 3D objects from limited tomographic data where the object can be approximated by three distinct regions separated by smooth surfaces [S1, unofficial].
*   **When not to use:** Not suitable for highly complex, non-separable, or topologically irregular objects that cannot be described by three rotating planar curves [S1, unofficial].

## Background
The Parametric Surface Density Model was proposed in the context of tomographic reconstruction from a small number of views [S1, unofficial]. The work, authored by an unnamed researcher (thesis/report), focuses on defining a 3D object model characterized by six quantities: three separating surfaces and density fields on these interfaces [S1, unofficial]. The primary difficulty addressed was filling a 3D field from the density fields on the interfaces, leading to the proposal of an adaptive elliptic operator [S1, unofficial]. The model also incorporates deformation via PCA on stochastic examples, allowing for the characterization of surface variations [S1, unofficial].

## Sources
[S1] Reconstruction tomographique à partir d'un petit nombre de vues, https://openalex.org/W650467298, fetched 2026-10-29 (unofficial)
