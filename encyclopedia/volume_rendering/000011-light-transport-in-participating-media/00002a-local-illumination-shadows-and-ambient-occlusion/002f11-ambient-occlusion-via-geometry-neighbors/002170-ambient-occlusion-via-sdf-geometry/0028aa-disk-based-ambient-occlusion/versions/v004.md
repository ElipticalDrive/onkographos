# Disk-Based Ambient Occlusion

Also called: Bunnel's Disk-Based AO, Disk Approximation AO

## Overview
Disk-Based Ambient Occlusion approximates surface geometry as a collection of disk-shaped elements located at mesh vertices to compute ambient occlusion and bent normals analytically, avoiding the need for expensive ray casting [S1, unofficial]. It distinguishes itself from SDF-based methods by using explicit geometric primitives (disks) rather than implicit distance fields, and resolves the "double shadowing" artifact inherent in naive disk summation through a two-pass accumulation scheme [S1, unofficial].

## How it works
**Inputs:** A set of mesh vertices, each associated with a surface normal $\mathbf{n}_i$ and a disk radius $r_i$.
**Outputs:** An ambient occlusion factor $AO_i \in [0, 1]$ and optionally a bent normal $\mathbf{b}_i$ for each vertex.

**Data Structures:**
*   **Disk List:** An array of $N$ disks. Each disk $i$ is defined by:
    *   Center position $\mathbf{c}_i$ (vertex position).
    *   Normal $\mathbf{n}_i$ (vertex normal).
    *   Radius $r_i$ (parameter, typically derived from local geometry or fixed).
    *   Current occlusion weight $w_i$ (initialized to 1.0).

**Algorithm Steps:**

1.  **Initialization:**
    For each vertex $i$ from $0$ to $N-1$:
    *   Set initial occlusion weight $w_i = 1.0$.
    *   Set accumulated occlusion $A_i = 0.0$.

2.  **Pairwise Occlusion Calculation (Naive):**
    For each vertex $i$ from $0$ to $N-1$:
        For each vertex $j$ from $0$ to $N-1$:
            If $i \neq j$:
                Calculate the occlusion factor $f_{ij}$ that disk $j$ casts on disk $i$.
                *Note: The specific analytic formula for $f_{ij}$ is not provided in the sources, but it relies on the analytical occlusion between two disks [S1, unofficial].*
                Add $f_{ij}$ to $A_i$.

3.  **Pass 1: Simple Summation:**
    The first pass computes the raw sum of occlusion factors. This sum includes "double shadowing," where disks behind other occluders still contribute to the darkness, leading to overly dark results [S1, unofficial].
    $$ A_i^{(1)} = \sum_{j \neq i} f_{ij} $$

4.  **Pass 2: Correction:**
    Reduce each disk's occlusion contribution by its own occlusion factor from the previous pass to mitigate double shadowing [S1, unofficial].
    $$ A_i^{(2)} = \sum_{j \neq i} f_{ij} \cdot (1 - w_j^{(1)}) $$
    Where $w_j^{(1)}$ is the normalized occlusion from Pass 1 for disk $j$.
    *Note: The exact normalization and reduction formula is not fully detailed in the sources, but the principle is reducing contribution by the occluder's own occlusion [S1, unofficial].*

5.  **Normalization:**
    Normalize the final accumulated occlusion to the range $[0, 1]$.
    $$ AO_i = \min(1.0, A_i^{(2)}) $$

**Optimization:**
A naive implementation has $O(N^2)$ runtime complexity, computing all disk pairs [S1, unofficial]. To make this practical, a hierarchical tree (e.g., BVH or octree) is used to optimize computations against distant disks, reducing complexity to $O(N \log N)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Double Shadowing:** Merely summing occlusion from all disks leads to an overly dark image because disks behind other disks still count towards occlusion [S1, unofficial]. The two-pass approach is required to solve this.
*   **Self-Intersection:** As with all geometry-based AO, a small bias offset along the normal $\mathbf{n}$ is required before querying geometry to prevent self-intersection artifacts (acne) [Parent].

## Subtopics

### Analytic Polygon Occlusion
Calculates the ambient occlusion by a single polygon $P$ on an infinitesimal patch $X$ with normal $\hat{n}$, defined as the form factor for diffuse radiative transfer between $P$ and $X$ [S2, unofficial]. It is chosen when high-precision analytic occlusion from specific polygonal geometry is needed, rather than disk approximations [S2, unofficial].
**Formula:**
$$ \text{AO}_P(\hat{n}) = \frac{1}{2\pi} \sum_{i=0}^{k-1} \cos^{-1} \left( \frac{\vec{p}_i \cdot \vec{p}_j}{\|\vec{p}_i\| \|\vec{p}_j\|} \right) \hat{n} \cdot \frac{\vec{p}_i \times \vec{p}_j}{\|\vec{p}_i \times \vec{p}_j\|} \quad (4) $$
where $j = (i+1) \mod k$, and $P$ has vertices $\{\vec{p}_0, \dots, \vec{p}_{k-1}\}$ in the positive half plane $\vec{p} \cdot \hat{n} \ge 0$ [S2, unofficial].
**Parameters:** Implementation cost is 1 arccosine, 15 multiply-add, and 2 reciprocal square root operations per edge [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Naive implementation is $O(N^2)$ [S1, unofficial]. With hierarchical tree optimization, it reduces to $O(N \log N)$ [S1, unofficial].
*   **Memory Complexity:** $O(N)$ to store the disk list and occlusion weights [S1, unofficial].
*   **Strengths:** Analytical calculation removes the need for ray casting, allowing for real-time performance with optimization [S1, unofficial]. It can compute bent normals simultaneously with AO [S1, unofficial].
*   **Limitations:** The naive approach is too slow for large scenes without optimization [S1, unofficial]. The two-pass correction adds computational overhead [S1, unofficial].
*   **When to use:** When real-time AO is needed and ray casting is too expensive, and the geometry can be approximated by disks [S1, unofficial].
*   **When not to use:** For scenes with highly complex, non-mesh geometry where disk approximation is inaccurate, or when extremely high precision is required without the overhead of hierarchical structures [S1, unofficial].

## Background
The method was presented by Bunnel [S1, unofficial]. It models surfaces as collections of disk-shaped elements located on mesh vertices to approximate ambient occlusion and bent normals analytically [S1, unofficial]. Hoberock improved upon this method's image quality, but at a performance cost tradeoff [S1, unofficial]. The analytic polygon occlusion variant was introduced to graphics by Baum et al. [BRW89] in the context of the radiosity algorithm [S2, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Ambient Occlusion Volumes, https://research.nvidia.com/sites/default/files/pubs/2010-06_Ambient-Occlusion-Volumes/McGuire10AOV.pdf, fetched 2026-10-28 (unofficial)
