# Adjustable Center Point for Voronoi Partitioning


## Overview
This method modifies Voronoi partitioning within a barycentric simplex by adjusting the center point to better handle specific material distributions, such as T-intersections [S1]. Unlike standard Soft Voronoi Decomposition which assumes symmetric sector subdivision, it allows the center $c$ to be shifted to arbitrary locations within the simplex to correctly weigh materials about the junction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A barycentric simplex (triangle for 3 materials, tetrahedron for 4 materials) containing material fraction data [S1].
*   **Input:** A target adjustment for the center point $c$ within the simplex [S1].
*   **Output:** Adjusted intersection edges and vertices defining the material boundaries [S1].

**Data Structures:**
*   **Barycentric Coordinates:** For an $m$-material case, each vertex of the grid has an associated barycentric coordinate vector $\lambda = (\lambda_1, \lambda_2, \dots, \lambda_m)$ [S1].
*   **Center Point $c$:** A point within the simplex, defaulting to $c = (1/3, 1/3, 1/3)$ in the standard 3-material case [S1].

**Steps:**
1.  **Identify Material Distribution:** Determine if the material distribution deviates from the uniform assumption, such as a T-intersection where one material occupies half the volume and others one-quarter each [S1].
2.  **Sample Neighborhood:** Sample in a larger neighborhood of the specific cell to understand how to weigh the materials about the "Y point" [S1].
3.  **Adjust Center Point $c$:** Move the center point $c$ from its default position to an arbitrary location within the barycentric triangle or tetrahedron [S1].
4.  **Adjust Intersection Edges:** Modify the edges that determine the intersections between material sectors to remain consistent with the new position of $c$ [S1].
5.  **Adjust Face Centers (4+ materials):** If the simplex is a tetrahedron or higher-dimensional simplex ($m$ materials), adjusting the center implies that the "center" vertices on the faces must also be adjusted so that the separating surfaces remain planar [S1].

**Formulas:**
*   Standard 3-material center: $c = (1/3, 1/3, 1/3)$ [S1].
*   General $m$-material case: The algorithm generalizes Nielson's and Franke's algorithm [S1]. Each vertex has barycentric coordinates $\lambda$, and restricting material fractions such that exactly one $\lambda_i = 1$ yields the case where each vertex is associated with only one material [S1].

**Pitfalls:**
*   **Planarity Constraints:** In the four-material case, adjusting the center of the tetrahedron implies that the "center" vertices on the faces must also be adjusted so that the separating surfaces remain planar [S1].
*   **Complexity:** Similar considerations hold when adjusting the center of the $m$-simplex as $m$ increases [S1].

## Subtopics

### Centroid Approximation
An acceleration technique for computing Voronoi cell centroids by averaging the positions of projected and un-projected vertices associated with the cell, instead of evaluating the exact centroid [S2]. It is chosen when computational speed is prioritized over exact geometric precision, as computing the exact centroid is time-consuming [S2]. In practice, no noticeable difference in the final results was observed [S2].

## Operating characteristics
*   **Accuracy:** Improves material interface representation in non-uniform distributions (e.g., T-intersections) compared to uniform barycentric assumptions [S1].
*   **Complexity:** Requires neighborhood sampling to determine weights, adding computational overhead compared to static centers [S1].
*   **Constraints:** Requires careful adjustment of face vertices in 4+ material cases to maintain planar surfaces [S1].
*   **Usage:** Use when material boundaries cannot be detected from vertex information alone or when uniform sector subdivision fails [S1]. Avoid when exact centroid computation is required and performance constraints are strict [S2].

## Background
The method is a direct generalization of Nielson's and Franke's algorithm [S1]. It addresses limitations in isosurface-type algorithms where material boundaries in tetrahedra cannot be detected if the isosurface enters a tetrahedron but only intersects one edge [S1]. The standard assumption of three distinct sectors subdividing a triangle in a "Y" fashion with equal material distribution is often invalid in complex scenes [S1].

## Sources
[S1] Constructing Material Interfaces from Data Sets with Volume-Fraction Information, https://digital.library.unt.edu/ark:/67531/metadc739076/?q=marching cubes, fetched 2026-10-28
[S2] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
