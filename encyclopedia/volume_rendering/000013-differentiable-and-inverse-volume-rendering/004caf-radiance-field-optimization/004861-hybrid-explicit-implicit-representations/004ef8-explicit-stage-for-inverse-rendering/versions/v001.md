# Explicit Stage for Inverse Rendering

Also called: Explicit Stage, Mesh-based Refinement

## Overview
The Explicit Stage is the second phase of a hybrid inverse rendering pipeline, designed to refine the coarse shape and appearance reconstruction obtained from an implicit stage [S1]. It converts a continuous implicit field into a discrete mesh and jointly optimizes vertex positions and spatially varying BRDF (SVBRDF) maps [S1]. Unlike purely implicit methods, this stage uses a mesh-based framework, allowing for better handling of complex light transport and avoiding the stair-case artifacts associated with low-resolution marching cubes in implicit-only approaches [S1]. It builds on [Hybrid Explicit-Implicit Representations](12240-hybrid-explicit-implicit-representations.md).

## How it works
**Inputs:**
*   Optimized implicit geometric function $\phi(\cdot; \theta^*_g)$ and reflectance field $B(\cdot; \theta^*_a)$ from the implicit stage [S1].
*   $n$ input images $I_j$ with known illumination and viewing conditions $\theta_s$ for $j=1, \dots, n$ [S1].
*   Optional: Anti-aliased mask images $S_j$ for each input image [S1].

**Outputs:**
*   Refined mesh geometry $M$ (vertex positions) [S1].
*   Refined SVBRDF maps (diffuse albedo $k_d$, specular albedo $k_s$, surface roughness $\alpha$) [S1].
*   Optimized parameters $(\theta^*_g, \theta^*_a)$ [S1].

**Data Structures:**
*   **Mesh $M$:** Defined by vertex positions and connectivity. Features (BRDF parameters) are cached at each vertex [S1].
*   **SVBRDF Maps:** 2D texture maps generated via UV parameterization of the mesh [S1].

**Algorithm Steps:**
1.  **Iso-surface Extraction:** Convert the implicit representation $\phi(\cdot; \theta^*_g)$ into an initial explicit mesh $\tilde{M}_0$ using iso-surface extraction (e.g., marching cubes) [S1]. This step is not differentiable [S1].
2.  **UV Parameterization:** Use the Boundary First Flattening (BFF) method [SC17] to UV-parameterize the mesh $\tilde{M}_0$ [S1].
3.  **SVBRDF Initialization:** Generate initial SVBRDF maps based on the reflectance field $B(\cdot; \theta^*_a)$ from the implicit stage [S1]. Cache BRDF parameter values ($k_d, k_s, \alpha$) at each mesh vertex [S1].
4.  **Joint Optimization:** Perform a second inverse-rendering optimization using a mesh-based framework [LZBD21]. This step jointly optimizes:
    *   The position of each mesh vertex [S1].
    *   The value of each pixel in the SVBRDF maps [S1].
5.  **Vertex Update:** Update mesh vertex positions using the approach by Nicolet et al. [NJJ21] to avoid expensive continuous collision detection and handling [S1].
6.  **Mask Supervision (Optional):** If mask images $S_j$ are available, include an extra loss term $\lambda_{mask} L_{mask}(\theta_g)$ [S1].

**Formulas:**
The optimization solves for $(\theta^*_g, \theta^*_a)$:
$$ (\theta^*_g, \theta^*_a) = \arg \min_{\theta_g, \theta_a} L(\theta_g, \theta_a) $$
where the total loss $L$ is:
$$ L(\theta_g, \theta_a) = L_{img}(\theta_g, \theta_a) + \lambda_{reg} L_{reg}(\theta_g, \theta_a) $$
The image loss $L_{img}$ is:
$$ L_{img}(\theta_g, \theta_a) = \sum_{j=1}^n \| R(\theta_g, \theta_a, \theta_s^{(j)}) - I_j \|^2 $$
The regularization loss $L_{reg}$ encourages $\phi$ to be a signed-distance function (unit-length gradients):
$$ L_{reg}(\theta_g, \theta_a) = \sum_i \| \nabla_x \phi(x_i; \theta_g) \|^2 $$
If mask supervision is used, the additional term is:
$$ \lambda_{mask} L_{mask}(\theta_g) = \sum_{j=1}^n \| R_{mask}(\theta_g, \theta_s^{(j)}) - S_j \|^2 $$
where $R_{mask}$ renders only an anti-aliased mask [S1].

**Parameters:**
*   $\lambda_{reg}$: Hyper-parameter controlling the weight of the regularization term, $\lambda_{reg} \in \mathbb{R}_{\ge 0}$ [S1].
*   $\lambda_{mask}$: Weight for the mask supervision loss (if used) [S1].

**Edge Cases and Pitfalls:**
*   **Non-differentiable Extraction:** The initial iso-surface extraction does not need to be differentiable, but errors here propagate to the optimization [S1].
*   **Collision Handling:** Using older geometric processing libraries (e.g., elTopo [BB09]) requires expensive continuous collision detection. The recommended approach [NJJ21] avoids this [S1].
*   **Performance:** Caching BRDF parameters at vertices and reusing the mesh for multiple camera views in a minibatch reduces computational overhead [S1].

## Subtopics

### [Combined Implicit and Explicit Inverse Rendering Pipeline](13888-combined-implicit-and-explicit-inverse-rendering-pipeline.md)
A two-stage inverse rendering pipeline that combines implicit and explicit geometric representations to solve physics-based inverse rendering problems, supporting topology changes and differentiable rendering of complex effects.

## Operating characteristics
*   **Complexity:** The optimization jointly updates vertex positions and SVBRDF pixels, which can be computationally intensive. However, caching BRDF values and reusing meshes for minibatches amortizes overhead [S1].
*   **Strengths:** Handles complex light transport better than implicit-only methods; avoids stair-case artifacts from low-resolution marching cubes; supports topology changes during optimization [S1].
*   **Limitations:** Requires a good initial implicit reconstruction; mesh-based optimization can be sensitive to initialization; UV parameterization may introduce distortions [S1].
*   **When to use:** When high-fidelity appearance and geometry are required, and the scene allows for mesh-based representation. Particularly useful when complex light transport effects are significant [S1].
*   **When not to use:** For scenes with highly dynamic topology where mesh maintenance is prohibitive, or when computational resources are extremely limited [S1].

## Background
The method originates from the work by Cai et al. (2022) on physics-based inverse rendering using combined implicit and explicit geometries [S1]. It builds upon the implicit stage framework by Yariv et al. [YKM*20] (idr) and the mesh-based inverse rendering framework by Luan et al. [LZBD21]. The approach leverages MeshSDF [RLR*20] for differentiable iso-surface extraction and Nicolet et al. [NJJ21] for efficient vertex updates [S1].

## Sources
[S1] Physics‐Based Inverse Rendering using Combined Implicit and Explicit Geometries, https://doi.org/10.1111/cgf.14592, fetched 2026-10-26
