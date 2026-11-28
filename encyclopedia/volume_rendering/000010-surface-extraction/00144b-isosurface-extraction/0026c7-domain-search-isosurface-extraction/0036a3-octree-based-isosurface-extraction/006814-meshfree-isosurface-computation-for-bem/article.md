# Meshfree Isosurface Computation for BEM

Also called: Meshfree BEM post-processing, Octree-based isovalue search with FMM

## Overview
This method computes isosurfaces for scalar fields (e.g., electric potential) derived from Boundary Element Method (BEM) data without generating an auxiliary post-processing volume mesh [S1]. It combines an adaptive octree scheme for isovalue search with FMM-accelerated BEM to compute field values directly via series expansions, avoiding dense volume meshes [S1]. Unlike classical approaches that rely on dense volume meshes for interpolation, this method uses higher-order Taylor series expansions to determine intersection points with high accuracy, enabling nearly arbitrarily adjustable resolution [S1].

## How it works
**Inputs:**
- Boundary element mesh (surface elements defining the domain boundary).
- Scalar field values at boundary element nodes (e.g., surface charge density or potential).
- Target isovalue $u_{iso}$.
- Parameters: Expansion order $L$ (typically 9–12), outer domain depth factor $n$ (positive integer).

**Outputs:**
- Triangular mesh representing the isosurface.

**Data Structures:**
1.  **Inner Octree ($T_i$):**
    -   Root cube $C_{ri}$: Centered at the center of the bounding box $B_e$ of all boundary elements. Edge length $l_{ri}$ equals the largest edge length of $B_e$ [S1].
    -   Subdivision: Standard FMM octree rules. Empty cubes are stored but only subdivided if boundary elements are assigned to neighbor cubes at a finer level [S1].
2.  **Outer Octree ($T_o$):**
    -   Root cube $C_{ro}$: Center coincides with $C_{ri}$. Edge length $l_{ro} = n \cdot l_{ri}$ [S1].
    -   Subdivision: Subdivided into empty child cubes until the level matches the child cubes of $C_{ri}$. Cubes coinciding with $T_i$ are omitted to ensure unique assignment of evaluation points [S1].
3.  **FMM Coefficients:**
    -   Local coefficients $L_n^m$ for cubes in $T_i$ [S1].
    -   Multipole coefficients $M_n^m$ for the outer domain [S1].

**Steps:**

1.  **Domain Subdivision:**
    Divide total space into an inner domain $\Omega_i$ (containing boundary elements) and an outer domain $\Omega_o$ (surrounding free space) [S1].

2.  **Octree Construction:**
    -   Create $T_i$ root $C_{ri}$ from $B_e$ [S1].
    -   Subdivide $C_{ri}$ and extend $T_i$ with empty cubes as needed for neighbor assignments [S1].
    -   Create $T_o$ root $C_{ro}$ with $l_{ro} = n \cdot l_{ri}$ [S1].
    -   Subdivide $T_o$ down to the resolution of $T_i$'s children, omitting overlaps with $T_i$ [S1].

3.  **FMM Expansion Computation:**
    Compute multipole expansions for the outer domain and local expansions for cubes in $T_i$ analogous to the classical FMM algorithm [S1].
    -   **Local Expansion (Far-field potential in $C_{rep}$):**
        $$u_f(r_{rep}) = \frac{1}{4\pi} \sum_{n=0}^{L} \sum_{m=-n}^{n} L_n^m \frac{Y_n^m(q, j)}{r^{n+1}}$$
        Where $L_n^m$ are local coefficients, $L$ is the expansion order, and $r, q, j$ are spherical coordinates of evaluation point $r_{rep}$ relative to the cube center [S1].
    -   **Multipole Expansion (Outer domain):**
        $$u_f(r_{rep}) = \frac{1}{4\pi} \sum_{n=0}^{L} \sum_{m=-n}^{n} M_n^m \frac{Y_n^m(q, j)}{r^{n+1}}$$
        Origin is the center of $C_{ri}$ [S1].

4.  **Value Range Determination:**
    -   Compute $u(r)$ at all nodes of the boundary elements [S1].
    -   For each octree cube, determine the value range of $u(r)$ using:
        -   Values of $u(r)$ on boundary elements inside the cube.
        -   Values of $u(r)$ at the cube corners.
        -   Boundary elements from neighbor cubes that extend into the cube [S1].

5.  **Cube Filtering:**
    Filter relevant cubes using the hierarchical tree structure based on their value ranges. If a cube and its neighbors contain no boundary elements, use the local expansion (Eq. 3) to compute $u(r)$ directly [S1].

6.  **Intersection Point Calculation:**
    -   Determine intersections of the isosurface with octree cube edges.
    -   Unlike linear interpolation, use the higher-order Taylor series expansion (Eq. 3) for $u(r)$ within the cube [S1].
    -   Apply an optimization method (e.g., bisectional line search or gradient-based search) to find precise intersection points [S1].

7.  **Refinement (Optional):**
    Compute additional intersection points inside cubes by evaluating along lines parallel to the cube edges to increase resolution [S1].

8.  **Mesh Construction:**
    Construct triangular elements of the isosurface using an adapted advancing front meshing algorithm applied to the computed intersection nodes [S1].

**Pitfalls and Edge Cases:**
-   **Singular/Near-Singular Integrals:** In the near-field, direct evaluation of the BEM integral is required for high accuracy. The method handles this by evaluating the integral directly for a small number of nearby boundary elements, while using series expansions for the far-field [S1].
-   **Separation Condition:** FMM series expansions require definite separation between sources and evaluation points. This is handled by circumscribing elements with spheres, resulting in an extra cube layer between corresponding cubes in practical implementations [S1].

## Operating characteristics
-   **Accuracy:** High accuracy is achieved because the method uses higher-order polynomials (Taylor series) to approximate the field, whereas linear volume elements would require a large number of elements for comparable accuracy [S1].
-   **Resolution:** The resolution of the isosurface is nearly arbitrarily adjustable through refinement steps (additional intersection points) [S1].
-   **Computational Cost:** Avoids expensive computations of field values in nodes of a dense volume mesh. The FMM acceleration ensures efficient field value computations in relatively large domains using only a small number of coefficients [S1].
-   **Memory:** Requires storage for two octrees ($T_i$ and $T_o$) and FMM coefficients. The number of cube layers in $\Omega_o$ is arbitrarily adjustable, allowing trade-offs between memory usage and domain coverage [S1].

## Background
This method was presented by A. Buchau et al. in 2017 [S1]. It addresses the limitations of classical isosurface computation methods that rely on auxiliary post-processing volume meshes, which are expensive to generate and compute [S1]. The approach couples established isovalue search methods with fast boundary element method (BEM) techniques and advancing front meshing algorithms [S1]. It is particularly useful for visualizing scalar fields (like electric potential) in virtual reality environments or for applications requiring high-accuracy isosurfaces without the overhead of volume meshing [S1].

## Sources
[S1] A meshfree isosurface computation method for boundary element methods, https://doi.org/10.2495/cmem-v5-n5-647-658, fetched 2026-10-26
