# Bardett Cone Filter Voxelization

Also called: Cone filter voxelization, Filtered voxelization with Bardett cone filter

## Overview
Bardett Cone Filter Voxelization is a volume sampling technique that uses a cone-shaped filter to approximate object details during the voxelization of parametric surfaces [S1]. It is specifically preferred for objects with sharp details, such as small features, where preserving edge sharpness is critical [S1]. Unlike the Oriented Box Filter, which provides linear density profiles, the Bardett cone filter introduces a nonlinear density profile [S1]. This nonlinearity causes errors in surface normal estimation, leading to more serious artifacts than slightly smoother edges, making it less suitable for general objects where normal accuracy is paramount [S1].

## How it works
**Inputs:** Parametric surface definition; sampling points on the surface.
**Outputs:** Voxel grid with density values.

**Data Structures:**
*   **Voxel Grid:** 3D array of floating-point density values.
*   **Sampling Points:** Set of 3D coordinates $(x, y, z)$ on the surface.

**Parameters:**
*   **Filter Radius ($r$):** The support radius of the cone filter.
    *   Typical value: Not explicitly given for the cone filter alone, but the oriented box filter optimal radius is 1.8 [S1].
    *   For thin objects: 0.8 [S1].

**Steps:**
1.  **Surface Sampling:** Uniformly sample the parametric surface. An adaptive technique may be used to ensure sufficient sampling density [S1].
2.  **Filter Application:** For each sample point, apply the Bardett cone filter. The filter is oriented based on the surface normal at that point [S1].
3.  **Contribution Aggregation:** Unlike standard filtering which sums contributions, the Bardett cone filter technique described in the source uses only the **minimum contribution** of all samples within the filter domain into account [S1]. This step is crucial to fulfill the condition of linearity in the surface transition area [S1].
4.  **Voxel Update:** Update the voxel grid with the computed minimum contribution values [S1].

**Formulas:**
The surface gradient is not explicitly defined for the cone filter in the provided text, but the oriented box filter uses forward and backward differences [S1]. For the cone filter, the key operation is the selection of the minimum contribution:
$$ V_{voxel} = \min_{i \in \text{samples in domain}} (C_i) $$
where $C_i$ is the contribution of sample $i$ [S1].

**Edge Cases and Pitfalls:**
*   **Nonlinear Density Profile:** The cone filter introduces a nonlinear density profile, which causes error in surface normal estimation [S1].
*   **Artifacts:** This normal estimation error leads to more serious artifacts than slightly smoother edges [S1].
*   **Thin Objects:** For thin objects like curves, normal precision estimation is less critical, allowing the use of smaller support filters (radius 0.8) [S1].

## Subtopics

### Oriented Box Filter Voxelization
The voxelization method of choice for objects where details can be approximated by a sphere of radius not smaller than 2 voxel units [S1]. It uses forward and backward differences to compute the surface gradient, fitting the maximum of both into account [S1]. The optimal filter radius is 1.8 for general objects and 0.8 for thin objects like curves [S1]. Using the maximum of forward and backward differences deteriorates performance for thick transient regions [S1]. For a sphere with radius 15, using an oriented box filter with radius 0.8 and adaptive normal estimation, the surface position error is below 0.1 voxel unit and the surface normal is estimated with error lower than 10 degrees [S1].

## Operating characteristics
*   **Strengths:** Provides better results for small details and sharper edges compared to box filters [S1].
*   **Limitations:** Introduces nonlinear density profiles, leading to significant errors in surface normal estimation and more serious artifacts than slightly smoother edges [S1].
*   **When to use:** Objects with sharp details where edge sharpness is prioritized over normal accuracy, or thin objects like curves where normal precision is less critical [S1].
*   **When not to use:** General objects where accurate surface normal estimation is required, as the oriented box filter is preferred in those cases [S1].

## Background
The technique is described in the context of voxelization of parametric surfaces, where distance to the surface is estimated [S1]. The method was proposed to fulfill the condition of linearity in the surface transition area [S1]. It is part of a broader work on voxelization of volumetric data by ray tracing [S1].

## Sources
[S1] Object voxelization by filtering, https://doi.org/10.1145/288126.288178, fetched 2026-11-03
