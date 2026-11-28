# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes which geometric primitives are visible from specific regions of the viewing space, allowing the renderer to discard occluded geometry during traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scene model (e.g., mesh with millions of triangles) and a partitioning of the viewing space into regions [S1, unofficial].
*   **Output:** A visibility map associating each region with the subset of primitives visible from that region [S1, unofficial].

**Data Structures:**
*   **Region Partition:** A spatial decomposition of the camera space that defines the "from-region" scope [S1, unofficial].
*   **Visibility Data:** Per-region visibility information storing which primitives are not fully occluded [S1, unofficial].

**Steps:**
1.  **Partitioning:** Divide the 3D environment into distinct spatial regions [S1, unofficial].
2.  **Visibility Computation:** For each region, determine which triangles are visible, often employing occluder fusion to reduce the number of occluding primitives [S1, unofficial].
3.  **Storage:** Store the resulting visibility data for runtime lookup [S1, unofficial].

**Parameters:**
*   **Region Size/Count:** Not given in the sources.
*   **Occluder Fusion Threshold:** Not given in the sources.

**Pitfalls:**
*   **Preprocessing Time:** The algorithm requires significant preprocessing time. Reported times include 9 hours for an 8-million-triangle model (Wonka et al.), 33 hours for a 6-million-triangle model (Durand et al.), and 128 hours for a 13-million-triangle model on a 16-PC cluster (Chhugani et al.) [S1, unofficial].
*   **Dimensionality Limitations:** Early implementations by Wonka et al. were limited to 2.5D datasets [S1, unofficial].

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It guarantees less than one pixel of screen-space deviation and correct visibility [S1, unofficial]. For a model with 13 million triangles, using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

### Hierarchical Edge/Face Visibility Culling
A pre-processing method creating a hierarchy of mesh edges and faces to permit constant time visibility testing for silhouettes [S2]. It organizes subtrees such that each is either a cluster of front-facing edges or back-facing edges, stores anchored nodes at each node indicating viewpoints for which clusters are either all front or back-facing, and performs constant time visibility testing using the anchored nodes [S2].

## Operating characteristics
*   **Time Complexity:** Preprocessing time is high. Reported times include 9 hours for 8 million triangles, 33 hours for 6 million triangles, and 128 hours for 13 million triangles on a 16-PC cluster [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Can handle 3D environments (Durand et al.), provides strict image accuracy guarantees (Chhugani et al.), and can overlap visibility computation and rendering using multiple processors (Wonka et al.) [S1, unofficial].
*   **Limitations:** Long preprocessing times; early versions limited to 2.5D datasets [S1, unofficial].

## Background
The technique was introduced by Wonka et al. [152] as a from-region visibility preprocessing algorithm with occluder fusion, initially limited to 2.5D datasets [S1, unofficial]. Durand et al. [41] extended this to handle 3D environments, though with even longer preprocessing times [S1, unofficial]. Schaufler et al. [120] also presented a from-region 3D visibility preprocessing algorithm, but their largest test model had only 0.6 million triangles [S1, unofficial]. Chhugani et al. [20] developed a system incorporating this with levels of detail for high-accuracy rendering [S1, unofficial]. In later work, Wonka et al. [153] used a from-point approach that needed little preprocessing, but only reported results for in-core, 2.5D datasets [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
[S2] State of the Art Non-Photorealistic Rendering (NPR) Techniques, https://doi.org/10.2312/localchapterevents/tpcg/tpcg06/089-098, fetched 2026-10-26
