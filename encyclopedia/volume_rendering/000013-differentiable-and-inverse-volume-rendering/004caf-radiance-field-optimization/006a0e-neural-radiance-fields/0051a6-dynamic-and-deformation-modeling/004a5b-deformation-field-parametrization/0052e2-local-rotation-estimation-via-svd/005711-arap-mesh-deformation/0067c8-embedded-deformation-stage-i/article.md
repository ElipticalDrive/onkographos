# Embedded Deformation (Stage I)
Also called: ARAP Deformation, Deformation Graph

## Overview
Embedded Deformation (Stage I) applies As-Rigid-As-Possible (ARAP) geometric deformation to 3D Gaussian Splatting (3DGS) representations using a deformation graph of control nodes [S1]. It displaces embedded geometries (Gaussian centers and endpoints) via affine transformations to approximate per-Gaussian rotation and scaling, but visual artifacts under large deformations remain due to the mismatch between deformed geometry and the deformed radiance field, necessitating a second alignment stage [S1].

## How it works
**Inputs:**
*   Set of 3D Gaussians, each defined by center $p_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and spherical harmonics coefficients $H_i$ [S1].
*   User-defined constraints $q_l$ specifying desired deformed positions for selected control points [S1].

**Outputs:**
*   Deformed Gaussian centers $\tilde{p}_i$ [S1].
*   Deformed covariance matrices $\tilde{\Sigma}_i$ derived from rotation $r$ and scaling $s$ [S1].
*   Rotated spherical harmonics coefficients [S1].

**Data Structures:**
*   **Deformation Graph:** A graph where vertices are control points $g_j$ and edges connect neighbors via KNN [S1].
*   **Control Point Affine Transform:** Each node $j$ stores a $3 \times 3$ matrix $A_j$ and a $3 \times 1$ translation vector $t_j$ [S1].
*   **Gaussian Endpoints:** For each Gaussian, 6 endpoints located at $\pm 2\sigma$ along the three principal axes [S1].

**Steps:**
1.  **Construct Deformation Graph:** Select control points from Gaussian centers using Furthest Point Sampling (FPS) [S1]. Construct edge connections between control points using the K-Nearest Neighbors (KNN) algorithm [S1].
2.  **Define Energies:** Minimize the total energy $E_{geo}$ to find optimal $A_j$ and $t_j$ [S1]:
    $$ E_{geo} = \sum_j \left( w_{rot} E_{rot}(A_j) + w_{reg} E_{reg}(A_j) \right) + w_{con} E_{con} $$
    where:
    *   $E_{rot}(A)$ measures deviation from rigid rotation. Let $c_1, c_2, c_3 \in \mathbb{R}^3$ be the column vectors of $A$:
        $$ E_{rot}(A) = (c_1 \cdot c_2)^2 + (c_1 \cdot c_3)^2 + (c_2 \cdot c_3)^2 + (c_1 \cdot c_1 - 1)^2 + (c_2 \cdot c_2 - 1)^2 + (c_3 \cdot c_3 - 1)^2 $$
    *   $E_{reg}(A)$ penalizes sharp translational changes between adjacent control nodes $k$:
        $$ E_{reg}(A) = \sum_k \| A(g_k - g) + g + t - (g_k + t_k) \|^2 $$
        Here, $g$ and $t$ are the position and translation of the current node, and $g_k, t_k$ are those of neighbor $k$ [S1].
    *   $E_{con}$ enforces user constraints $q_l$ for deformed control point positions $\tilde{g}_l$:
        $$ E_{con} = \sum_l \| \tilde{g}_l - q_l \|^2 $$
3.  **Optimize Transformations:** Solve for $A_j$ and $t_j$ that minimize $E_{geo}$ [S1].
4.  **Displace Embedded Geometries:** Compute the deformed position $\tilde{v}_i$ for any embedded geometry point $v_i$ (e.g., Gaussian center or endpoint) using weights $w_j(v_i)$ inherited from control node $j$:
    $$ \tilde{v}_i = \sum_j w_j(v_i) \left( A_j (v_i - g_j) + g_j + t_j \right) $$
    where $g_j$ is the original position of control node $j$ [S1].
5.  **Reconstruct Gaussians:** For each Gaussian, select 6 endpoints along the three axes at twice the standard deviation from the center [S1]. Displace these endpoints using the Embedded Deformation formula above [S1]. Let $Q \in \mathbb{R}^{3 \times 6}$ be the matrix of original endpoint positions and $P \in \mathbb{R}^{3 \times 6}$ be the matrix of deformed endpoint positions [S1]. Estimate the local deformation matrix $M$:
    $$ M = P Q^\top (Q Q^\top)^{-1} $$
    Perform polar decomposition on $M$ to obtain rotation $r$ and scaling $s$ [S1]. Update the Gaussian's covariance matrix using $r$ and $s$ [G]. Rotate the spherical harmonics coefficients corresponding to the new rotation $r$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def embedded_deformation_stage_i(gaussians, constraints):
    # 1. Construct deformation graph [S1]
    control_points = furthest_point_sampling(gaussians.centers)
    graph = build_knn_graph(control_points)
    
    # 2. Optimize affine transforms [S1]
    A, t = minimize_energy(graph, constraints)
    
    # 3. Displace geometries & reconstruct Gaussians [S1]
    deformed_gaussians = []
    for g in gaussians:
        # Displace center [S1]
        center_deformed = sum(w_j(g.center) * (A_j @ (g.center - g_j) + g_j + t_j) for j in graph.neighbors(g.center))
        
        # Displace endpoints [S1]
        endpoints_orig = get_6_endpoints(g.center, g.sigma)
        endpoints_deformed = apply_deformation(endpoints_orig, A, t, graph)
        
        # Estimate local deformation [S1]
        M = endpoints_deformed @ endpoints_orig.T @ inv(endpoints_orig @ endpoints_orig.T)
        r, s = polar_decomposition(M)
        
        # Update Gaussian [S1, G]
        cov_deformed = r @ g.covariance @ r.T * s
        sh_deformed = rotate_spherical_harmonics(g.sh, r)
        deformed_gaussians.append(Gaussian(center_deformed, cov_deformed, g.opacity, sh_deformed))
    return deformed_gaussians
```

**Parameters:**
*   $w_{rot} = 1$
*   $w_{reg} = 10$
*   $w_{con} = 100$ [S1]

**Pitfalls:**
*   Direct geometric deformation causes visual artifacts under large deformations due to the mismatch between deformed Gaussians and the deformed radiance field [S1]. This method addresses the geometry but requires Stage II to align with the radiance field [S1].

## Subtopics

### [Anchor-Guided ARAP Deformation](21744-anchor-guided-arap-deformation.md)
Anchor-Guided ARAP Deformation is a variational mesh deformation method that computes a globally rigid-consistent deformation of a template mesh by minimizing a combination of As-Rigid-As-Possible (ARAP) energy and a sparse anchor fidelity term. It solves the problem of applying noisy or sparse motion flows (e.g., from voxelized geometry tracking) to a mesh without propagating noise across the entire surface. Unlike standard ARAP [SA07] which relies on fixed boundary conditions, this method uses a dynamically selected subset of high-curvature or high-motion vertices as anchors, making it resilient to input noise while preserving local geometric features.

### Neural Deformation Graph
Neural Deformation Graph is a self-supervised modeling method that uses a graph structure to model objects, assigning an SDF to each node to describe shapes [S2]. It designs multiple loss functions to constrain the graph: covering the whole body, uniform node distribution within the contour, and sparse edge connections [S2]. It takes an SDF sequence as input and evaluates deformation using the relative displacement of graph nodes between frames, which is then used to generate dynamic skinning weights [S2].

## Operating characteristics
*   **Complexity:** Depends on the number of control points, their KNN neighbors, and the number of Gaussians [S1]. Optimization of $E_{geo}$ is typically solved via least squares or iterative solvers [G].
*   **Strengths:** Provides physically plausible, rigid-like deformation for point clouds and meshes; avoids the need for explicit mesh topology in Gaussian Splatting [G].
*   **Limitations:** Does not fully correct radiance field misalignment artifacts on its own; requires a second stage for full consistency [S1].

## Background
Embedded deformation was originally introduced for meshes and point clouds [S1]. In the context of 3DGS, it is adapted to handle the particle-like nature of Gaussians [S1]. This method is part of a two-stage pipeline for ARAP deformation of Gaussian Radiance Fields [S1].

## Sources
[S1] As-Rigid-As-Possible Deformation of Gaussian Radiance Fields, https://doi.org/10.1109/TVCG.2025.3555404, fetched 2026-10-30
[S2] Human 3D Avatar Modeling with Implicit Neural Representation: A Brief Survey, https://doi.org/10.1109/icsps58776.2022.00148, fetched 2026-10-26
