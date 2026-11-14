# Sobierajski’s method

Also called: Hierarchical Zonal Method, Sobierajski's Hierarchical Zonal Method

## Overview
Sobierajski’s method is a hierarchical volumetric radiosity technique for calculating multiple scattering in participating media [S1][S2, book]. It extends the Zonal Method by representing volumetric data as voxels that model Lambertian surfaces, isotropic media, or a combination thereof [S1][S2, book]. Unlike classical hierarchical radiosity which relies on pre-computed explicit links between elements, this method finds the best highest possible levels of interaction on-the-fly during each shooting iteration, avoiding the quadratic cost of the initial link phase [S1][S2, book]. It builds on [Zonal Method](1891-zonal-method.md) by introducing a voxel-based hierarchy and a Push-Pull procedure to maintain energy conservation across levels [S1][S2, book].

## How it works
**Inputs:** A scene containing participating media represented as a set of global volumes, where each volume is discretized into a 3D grid of voxels. Each voxel is characterized by its geometry, albedo, and scattering properties.
**Outputs:** The radiosity (diffuse and isotropic components) for all voxels in the hierarchy.

**Data Structures:**
1.  **Voxel:** Represents the base unit of the hierarchy.
    *   `pos`: Center position of the voxel.
    *   `size`: Side length of the voxel (assumed cubic).
    *   `albedo_diffuse`: Coefficient for ideal diffuse reflection.
    *   `albedo_isotropic`: Coefficient for isotropic scattering.
    *   `radiosity_diffuse`: Current diffuse radiosity value.
    *   `radiosity_isotropic`: Current isotropic radiosity value.
    *   `parent`: Pointer to the parent voxel in the hierarchy.
    *   `children`: Array of 8 child voxels (if not a leaf).
2.  **Hierarchy:** A 3D octree structure.
    *   Level 0: Individual voxels.
    *   Level $k$: Formed by combining 8 neighboring voxels from level $k-1$.
    *   The hierarchy is built in a preprocessing step.

**Formulas:**
*   **Voxel BRDF:** The Bidirectional Reflectance Distribution Function of a voxel is the sum of ideal diffuse reflection and isotropic scattering [S1][S2, book]:
    $$ f_r = f_{diffuse} + f_{isotropic} $$
    Where $f_{diffuse}$ corresponds to Lambertian reflection and $f_{isotropic}$ corresponds to uniform scattering in all directions.
*   **Radiosity Decomposition:** Each voxel has two radiosity components [S1][S2, book]:
    $$ B_{voxel} = B_{diffuse} + B_{isotropic} $$
*   **Interaction Criterion:** An interaction between two nodes is considered if the amount of radiosity shot from one element to another exceeds a given threshold $\epsilon$ [S2, book]. Alternatively, an estimated form factor can be used to check if the estimated energy arriving is below a threshold, which is particularly useful for optically thick media [S2, book].

**Algorithm Steps:**
1.  **Preprocessing (Hierarchy Construction):**
    1.  Discretize the participating media into a 3D grid of voxels (Level 0) [S1][S2, book].
    2.  Recursively combine 8 neighboring voxels at level $k$ to form one parent voxel at level $k+1$ [S1][S2, book].
    3.  For each parent voxel, compute its averaged radiosity and albedo values from its children [S1].
    4.  Define a criterion to decide if a parent voxel is a good approximation of its descendents (e.g., based on variance of radiosity or albedo) [S1][S2, book].

2.  **Iterative Shooting:**
    1.  Initialize all voxel radiosity values (diffuse and isotropic) based on light sources and initial emission [S1][S2, book].
    2.  Repeat until convergence or maximum iterations:
        a.  **Select Source:** Choose a voxel (or node) with significant unshot radiosity [S1][S2, book].
        b.  **Find Interaction Levels:** For each target node in the scene, determine the highest level in the hierarchy where the source and target can interact. This is done on-the-fly without explicit links [S1][S2, book].
            *   Check if the estimated energy transfer between the source node and the target node (at the determined level) exceeds the threshold $\epsilon$ [S2, book].
            *   If the energy is below $\epsilon$, skip the interaction [S2, book].
        c.  **Shoot Energy:** Transfer energy from the source node to the target node based on the form factors between the diffuse and isotropic components [S1][S2, book].
            *   Update the radiosity of the target node [S1][S2, book].
        d.  **Push-Pull Procedure:** After each shooting iteration, perform a Push-Pull procedure to ensure the correct representation of energies at all levels of the hierarchy [S1][S2, book].
            *   **Push:** Propagate radiosity values from children to parents (averaging) [S1][S2, book].
            *   **Pull:** Propagate radiosity values from parents to children (distribution) [S1][S2, book].
            *   *Note:* The Push-Pull procedure as described in the sources limits the method to media with constant albedo. Pushing irradiance instead of radiosity would solve this drawback [S2, book].

**Pitfalls and Edge Cases:**
*   **Self-Refinement:** Participating media are modeled as global volumes. Each global volume is refined against other volumes but not against itself [S2, book]. Therefore, explicit links must be set between each pair of the smallest volume elements within a given global volume to correctly compute self-illumination [S2, book]. A self-refinement strategy would reduce the number of interactions but is not implemented in the basic method [S2, book].
*   **Push-Pull Limitation:** The Push-Pull procedure may not correctly set radiosity values at all levels if the albedo is not constant [S2, book].
*   **Optically Thick Media:** The standard interaction criterion may result in too many interactions. Using an estimated form factor to check if arriving energy is below a threshold can dramatically decrease interactions [S2, book].
*   **Non-Convex Objects:** Self-lighting in non-convex solid objects may be missed if self-interaction is not properly handled [S2, book].

## Operating characteristics
*   **Time Complexity:** The method avoids the quadratic cost of the initial link phase of classical hierarchical methods by finding interactions on-the-fly [S1][S2, book]. However, the cost of finding interaction levels and performing the Push-Pull procedure after each shooting iteration can be high [S2, book].
*   **Memory Complexity:** The hierarchy requires storing radiosity values for all nodes at all levels. The memory usage is proportional to the number of voxels and the depth of the hierarchy [S1][S2, book].
*   **Strengths:**
    *   Efficient handling of multiple scattering in participating media [S1][S2, book].
    *   Avoids pre-computation of explicit links, reducing initial setup cost [S1][S2, book].
    *   Can model both Lambertian surfaces and isotropic media within the same framework [S1][S2, book].
*   **Limitations:**
    *   The Push-Pull procedure is limited to media with constant albedo [S2, book].
    *   Self-illumination in non-convex volumes requires explicit links between smallest elements, which can be costly [S2, book].
    *   The method is an iterative shooting algorithm, which may converge slowly for some scenes [S2, book].
*   **When to use:** Suitable for scenes with participating media where multiple scattering is significant and the media can be approximated as isotropic [S1][S2, book].
*   **When not to use:** Not suitable for media with highly anisotropic scattering or varying albedo if the Push-Pull limitation is not addressed [S2, book].

## Background
Sobierajski’s method was introduced by Sobierajski [S1][S2, book]. It is an adaptation of hierarchical radiosity techniques, such as those by Sillion [S1][S2, book], to include isotropic volumes. The method was developed to address the computational challenges of rendering participating media with multiple scattering, extending the principles of diffuse radiosity to volumetric elements [S1][S2, book].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
