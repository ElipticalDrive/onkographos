# Disk-Based Ambient Occlusion

Also called: Bunnel's Disk-Based AO, Disk Approximation AO

## Overview
Disk-Based Ambient Occlusion approximates surface visibility by modeling mesh vertices as collections of disk-shaped elements, allowing for the analytical calculation of occlusion factors without ray casting [S1, unofficial]. It solves the problem of real-time ambient occlusion for dynamic geometry where precomputation is infeasible, distinguishing itself from SDF-based methods by operating directly on mesh topology rather than a volumetric grid [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of mesh vertices, each associated with a disk element (defined by position, normal, and radius).
*   **Output:** An occlusion factor for each vertex/disk, ranging from 0 (fully occluded) to 1 (fully visible).

**Data Structures:**
*   **Disk Element:** A structure containing the vertex position $\mathbf{p}$, surface normal $\mathbf{n}$, and radius $r$.
*   **Hierarchical Tree:** A spatial partitioning structure (e.g., BVH or octree) built over the disk elements to optimize culling of distant disks [S1, unofficial].

**Algorithm Steps:**
The method uses a two-pass approach to mitigate "double shadowing," where summing occlusion from all disks leads to overly dark results because occluders behind other occluders are counted multiple times [S1, unofficial].

1.  **Pass 1: Naive Summation**
    For each disk $i$, compute the occlusion factor $O_i$ by summing the occlusion contributions $C_{ij}$ from all other disks $j$:
    $$ O_i = \sum_{j \neq i} C_{ij} $$
    Here, $C_{ij}$ is the analytical occlusion cast by disk $j$ onto disk $i$. This pass accumulates double shadowing errors.

2.  **Pass 2: Correction**
    Reduce each disk's occlusion contribution by its own accumulated occlusion factor from Pass 1. This effectively normalizes the result to account for the fact that if disk $i$ is already heavily occluded, it cannot occlude others as effectively (or rather, the shadow it casts is attenuated by the fact that it is in shadow itself, depending on the specific interpretation of the correction). The final occlusion factor $AO_i$ is derived from $O_i$ and the correction step [S1, unofficial].

**Analytical Occlusion Calculation:**
The occlusion $C_{ij}$ cast by disk $j$ onto disk $i$ is calculated analytically. While the exact formula for two disks is complex, it relies on the geometric relationship between the two disk planes and their relative positions. The hierarchical tree allows the algorithm to skip pairs $(i, j)$ where disk $j$ is too distant or poorly aligned to significantly affect disk $i$, reducing the complexity from $O(n^2)$ to $O(n \log n)$ [S1, unofficial].

**Parameters:**
*   **Disk Radius:** Typically related to the local triangle size or a fixed fraction of the mesh scale. Not given in the sources.
*   **Tree Depth:** Depends on the scene complexity. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Double Shadowing:** As noted, naive summation produces incorrect, overly dark results. The two-pass correction is essential [S1, unofficial].
*   **Performance:** Without the hierarchical tree, the $O(n^2)$ complexity makes the method unusable for large scenes [S1, unofficial].
*   **Precision:** Analytical calculations involving dot products and cross products of normals and vectors must handle degenerate cases (e.g., parallel disks, coplanar disks) to avoid division by zero or NaNs [G].

## Subtopics

### Analytic Polygon Occlusion
A variant that calculates ambient occlusion by a single polygon $P$ on an infinitesimal patch $X$ with normal $\hat{n}$, defined as the form factor for diffuse radiative transfer between $P$ and $X$ [S2, unofficial]. The occlusion is computed as:
$$ \text{AO}_P(\hat{n}) = \frac{1}{2\pi} \sum_{i=0}^{k-1} \cos^{-1} \left( \frac{\vec{p}_i \cdot \vec{p}_j}{\|\vec{p}_i\| \|\vec{p}_j\|} \right) \hat{n} \cdot \frac{\vec{p}_i \times \vec{p}_j}{\|\vec{p}_i \times \vec{p}_j\|} $$
where $j = (i+1) \mod k$, and vertices $\{\vec{p}_0, \dots, \vec{p}_{k-1}\}$ lie in the positive half-plane $\vec{p} \cdot \hat{n} \ge 0$ [S2, unofficial]. This approach costs 1 arccosine, 15 multiply-add, and 2 reciprocal square root operations per edge [S2, unofficial]. It is used when extending the disk approximation to arbitrary polygonal geometry in deferred rendering contexts [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(n^2)$ for naive implementation; $O(n \log n)$ with hierarchical tree optimization [S1, unofficial].
*   **Memory Complexity:** $O(n)$ for storing disk elements and tree structures [S1, unofficial].
*   **Strengths:** Real-time capable for dynamic geometry; no ray casting required; provides bent normals in addition to AO [S1, unofficial].
*   **Limitations:** Approximation quality depends on disk radius and distribution; hierarchical tree construction adds overhead; double shadowing requires careful handling [S1, unofficial].
*   **When to use:** Dynamic scenes with mesh-based geometry where precomputed AO is not feasible [S1, unofficial].
*   **When not to use:** Static scenes where precomputed AO is better; scenes with extremely high polygon counts where tree construction is prohibitive [S1, unofficial].

## Background
The method was introduced by Bunnel [S1, unofficial]. It was later improved in image quality by Hoberock [S1, unofficial]. The underlying analytical concept for polygon occlusion was first introduced to graphics by Baum et al. [BRW89] in the context of radiosity algorithms [S2, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Ambient Occlusion Volumes, https://research.nvidia.com/sites/default/files/pubs/2010-06_Ambient-Occlusion-Volumes/McGuire10AOV.pdf, fetched 2026-10-28 (unofficial)
