# Platonic Solid Projection

Also called: PSP
## Overview
Platonic Solid Projection (PSP) is a 3D space factorization method that projects 3D areas onto the faces of a Platonic solid, allowing anisotropic 3D areas to be represented on 2D planes with distinguishable characterization [S1]. It reduces memory consumption from $O(n^3)$ to $O(n^2)$ by replacing the orthogonal tri-plane representation with a set of planes derived from the faces of various Platonic solids (tetrahedron, cube, octahedron, dodecahedron, or icosahedron) [S1]. Unlike the orthogonal tri-plane representation, which can be derived from a regular hexahedron (cube), PSP provides a flexible trade-off between rendering quality and efficiency by selecting solids with different numbers of faces [S1].

## How it works
**Inputs:**
*   A 3D Gaussian characterized by mean $\boldsymbol{\mu} \in \mathbb{R}^3$ and covariance $\boldsymbol{\Sigma} \in \mathbb{R}^{3 \times 3}$ [S1].
*   A selected Platonic solid with $n$ faces [S1].
*   For each face $i \in \{1, \dots, n\}$:
    *   Outward normal $\boldsymbol{\phi}_i \in \mathbb{R}^3$ [S1].
    *   Ripmap Encoding feature grid [S1].

**Outputs:**
*   A final concatenated feature vector $\mathbf{f}$ for the 3D Gaussian [S1].

**Data Structures:**
*   **Platonic Solid Configuration:** Defined by the set of planes $\{P_i\}$ and their outward normals $\{\boldsymbol{\phi}_i\}$ for $i = 1, \dots, n$, where $n$ is the number of faces [S1].
*   **Local 2D Axes:** For each plane $P_i$, local axes $\mathbf{x}_i \in \mathbb{R}^3$ and $\mathbf{y}_i \in \mathbb{R}^3$ are defined perpendicular to $\boldsymbol{\phi}_i$ [S1].
*   **Projection Matrix:** $M_i \in \mathbb{R}^{3 \times 2}$ maps the 3D world coordinate system to the 2D coordinate system of plane $P_i$ [S1].

**Steps:**

1.  **Define Local Axes:**
    Given the unit vectors $\mathbf{X}, \mathbf{Y}, \mathbf{Z}$ of the world coordinate system, define the local 2D axes $\mathbf{x}_i$ and $\mathbf{y}_i$ for each plane $P_i$ [S1]:
    $$
    \mathbf{x}_i = \begin{cases} 
    \mathbf{X} & \text{if } \boldsymbol{\phi}_i = \mathbf{Z} \\
    \mathbf{Z} \times \boldsymbol{\phi}_i & \text{otherwise}
    \end{cases}
    $$
    $$
    \mathbf{y}_i = \mathbf{x}_i \times \boldsymbol{\phi}_i
    $$
    [S1]

2.  **Construct Projection Matrix:**
    Construct the projection matrix $M_i$ by concatenating the local axes as columns [S1]:
    $$
    M_i = [\mathbf{x}_i, \mathbf{y}_i]
    $$
    [S1]

3.  **Project Mean and Covariance:**
    Project the 3D Gaussian's mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$ onto plane $P_i$ to obtain the 2D mean $\boldsymbol{\mu}_{proj}^i$ and covariance $\boldsymbol{\Sigma}_{proj}^i$ [S1]:
    $$
    \boldsymbol{\mu}_{proj}^i = M_i^T \boldsymbol{\mu}
    $$
    $$
    \boldsymbol{\Sigma}_{proj}^i = M_i^T \boldsymbol{\Sigma} M_i
    $$
    [S1]

4.  **Query Feature Vector:**
    Query a feature vector $\mathbf{f}_i$ from the Ripmap Encoding of plane $P_i$ using the projected mean $\boldsymbol{\mu}_{proj}^i$ and covariance $\boldsymbol{\Sigma}_{proj}^i$ to determine the interpolation position and level [S1]. (Note: The specific interpolation formula for Ripmap Encoding is defined in the [Ripmap Encoding](8553-ripmap-encoding.md) article).

5.  **Concatenate Features:**
    Concatenate all feature vectors $\{\mathbf{f}_i \mid i = 1, \dots, n\}$ from the corresponding planes $\{P_i \mid i = 1, \dots, n\}$ to form the final feature vector $\mathbf{f}$ of the 3D Gaussian [S1]:
    $$
    \mathbf{f} = \text{concat}(\mathbf{f}_1, \dots, \mathbf{f}_n)
    $$
    [S1]

**Parameters:**
*   $n$: The number of faces of the selected Platonic solid. This determines the trade-off between rendering quality and efficiency (training time and GPU memory consumption) [S1].

**Edge Cases and Pitfalls:**
*   **Indistinguishability:** If the number of planes is insufficient or poorly oriented, different 3D ellipsoids (e.g., with major axes aligned along different body diagonals of a cube) may project to indistinguishable 2D Axis-Aligned Bounding Boxes (AABBs) under the Ripmap Encoding [S1]. Using a larger number of planes with diverse orientations mitigates this [S1].
*   **Axis Definition:** The empirical definition of local axes $\mathbf{x}_i$ and $\mathbf{y}_i$ must be consistent to ensure correct projection [S1].

## Subtopics

### [Ripmap Encoding](8553-ripmap-encoding.md)
A technique to featurize the projected anisotropic 2D areas on the faces of a Platonic solid by using a feature grid pre-filtered with anisotropic kernels, enabling precise and efficient anisotropic area-sampling.

## Operating characteristics
*   **Memory Complexity:** Reduces memory consumption from $O(n^3)$ (3D volume) to $O(n^2)$ (2D planes) [S1].
*   **Trade-off:** The choice of Platonic solid (and thus $n$) controls the trade-off between rendering quality and efficiency (training time and GPU memory consumption) [S1].
*   **Strengths:** Enables precise representation of 3D scenes with 2D feature grids; allows anisotropic 3D areas to be projected onto planes with distinguishable characterization; improves upon orthogonal tri-plane representations by providing more diverse plane orientations [S1].
*   **Limitations:** Requires careful selection of Platonic solid to avoid indistinguishable projections for certain Gaussian orientations [S1].

## Background
Platonic Solid Projection was proposed in the paper "Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids" [S1]. It addresses limitations of orthogonal tri-plane representations (derived from a cube) by using various Platonic solids to provide a flexible trade-off between rendering quality and efficiency [S1]. The method is part of the Rip-NeRF framework, which also introduces Ripmap Encoding for anti-aliasing [S1].

## Sources
[S1] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
