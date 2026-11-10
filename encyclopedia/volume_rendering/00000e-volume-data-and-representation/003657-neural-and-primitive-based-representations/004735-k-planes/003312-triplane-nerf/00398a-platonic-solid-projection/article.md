# Platonic Solid Projection

Also called: PSP

## Overview
Platonic Solid Projection is a 3D space factorization method that projects 3D areas onto the faces of a Platonic solid, allowing anisotropic 3D areas to be represented on 2D planes with distinguishable characterization [S1]. It reduces memory consumption from $O(n^3)$ to $O(n^2)$ by replacing 3D volumes with 2D feature grids [S1]. Unlike the orthogonal tri-plane representation (derived from a regular hexahedron/cube), this method uses various Platonic solids to provide a flexible trade-off between rendering quality and efficiency by selecting solids with different numbers of faces [S1].

## How it works
**Inputs:**
- A 3D Gaussian characterized by mean $\boldsymbol{\mu} \in \mathbb{R}^3$ and covariance $\boldsymbol{\Sigma} \in \mathbb{R}^{3 \times 3}$ [S1].
- A selected Platonic solid with $n$ faces [S1].
- For each face $i \in \{1, \dots, n\}$:
  - Outward normal $\boldsymbol{\phi}_i \in \mathbb{R}^3$ [S1].
  - Local 2D axes $\mathbf{x}_i, \mathbf{y}_i \in \mathbb{R}^3$ perpendicular to $\boldsymbol{\phi}_i$ [S1].
  - A Ripmap Encoding (a feature grid pre-filtered with anisotropic kernels) associated with face $i$ [S1].

**Outputs:**
- A concatenated feature vector $\mathbf{f} \in \mathbb{R}^k$ formed by concatenating feature vectors $\mathbf{f}_i$ queried from each plane [S1].

**Data Structures:**
- **Platonic Solid Configuration:** A set of $n$ planes $\{P_i\}$ and their normals $\{\boldsymbol{\phi}_i\}$. The choice of solid determines $n$ (e.g., tetrahedron: 4, cube: 6, octahedron: 8, dodecahedron: 12, icosahedron: 20) [S1].
- **Ripmap Encoding:** A 2D hierarchical grid structure for each face $P_i$, enabling anisotropic area-sampling [S1].

**Steps:**
1. **Define Local Axes:** For each face $i$, define the local 2D basis vectors $\mathbf{x}_i$ and $\mathbf{y}_i$ based on the world coordinate system unit vectors $\mathbf{X}, \mathbf{Y}, \mathbf{Z}$ and the face normal $\boldsymbol{\phi}_i$ [S1]:
   $$
   \mathbf{x}_i = \begin{cases} 
   \mathbf{X} & \text{if } \boldsymbol{\phi}_i = \mathbf{Z} \\
   \mathbf{Z} \times \boldsymbol{\phi}_i & \text{otherwise}
   \end{cases}, \quad
   \mathbf{y}_i = \begin{cases} 
   \mathbf{Y} & \text{if } \boldsymbol{\phi}_i = \mathbf{Z} \\
   \mathbf{x}_i \times \boldsymbol{\phi}_i & \text{otherwise}
   \end{cases}
   $$
   [S1]

2. **Construct Projection Matrix:** Form the projection matrix $M_i \in \mathbb{R}^{3 \times 2}$ by concatenating the local axes as columns:
   $$
   M_i = [\mathbf{x}_i, \mathbf{y}_i]
   $$
   [S1]

3. **Project Mean and Covariance:** Project the 3D Gaussian's mean $\boldsymbol{\mu}$ and covariance $\boldsymbol{\Sigma}$ onto the 2D plane $P_i$ to obtain the projected mean $\boldsymbol{\mu}_{proj}^i$ and covariance $\boldsymbol{\Sigma}_{proj}^i$ [S1]:
   $$
   \boldsymbol{\mu}_{proj}^i = M_i^T \boldsymbol{\mu}
   $$
   $$
   \boldsymbol{\Sigma}_{proj}^i = M_i^T \boldsymbol{\Sigma} M_i
   $$
   [S1]

4. **Query Feature Vector:** Query a feature vector $\mathbf{f}_i$ from the Ripmap Encoding of plane $P_i$ using the projected parameters $\boldsymbol{\mu}_{proj}^i$ and $\boldsymbol{\Sigma}_{proj}^i$ to determine the interpolation position and level [S1].

5. **Concatenate Features:** Concatenate all feature vectors $\{\mathbf{f}_i \mid i = 1, \dots, n\}$ from the corresponding planes to form the final feature vector $\mathbf{f}$ of the 3D Gaussian [S1].

**Parameters:**
- $n$: The number of faces of the selected Platonic solid. This determines the trade-off between rendering quality and efficiency (training time and GPU memory consumption) [S1].

**Edge Cases and Pitfalls:**
- **Indistinguishability:** If 3D ellipsoids have major axes aligned along different body diagonals of a cube, they may project to indistinguishable 2D Axis-Aligned Bounding Boxes (AABBs) under isotropic sampling or insufficient plane diversity [S1]. Using a larger number of planes (faces) with diverse orientations resolves this [S1].
- **Axis Definition:** The empirical definition of local axes $\mathbf{x}_i, \mathbf{y}_i$ must ensure they are perpendicular to $\boldsymbol{\phi}_i$ and consistent across faces to maintain coordinate system integrity [S1].

## Subtopics

### [Ripmap Encoding](8553-ripmap-encoding.md)
Ripmap Encoding is a featurization method for anisotropic 2D areas projected onto the faces of a Platonic solid, enabling precise and efficient anisotropic area-sampling in neural rendering. It distinguishes itself from isotropic area-sampling methods like Tri-MipRF by using anisotropic kernels to characterize ellipsoidal footprints, preventing ambiguities and blurriness that arise when different viewing cones project to the same isotropic area. It builds on [Platonic Solid Projection](8518-platonic-solid-projection.md) by providing the hierarchical data structure required to resolve the anisotropy introduced by the projection.

## Operating characteristics
- **Memory Complexity:** Reduces memory consumption from $O(n^3)$ (3D volume) to $O(n^2)$ (2D planes) [S1].
- **Efficiency Trade-off:** The choice of Platonic solid (number of faces $n$) controls the trade-off between rendering quality and efficiency (training time and GPU memory consumption) [S1].
- **Strengths:** Enables precise representation of 3D scenes with 2D feature grids; allows anisotropic 3D areas to be projected onto planes with distinguishable characterization; supports flexible trade-offs via different Platonic solids [S1].
- **Limitations:** Requires careful selection of Platonic solid to avoid indistinguishable projections for certain Gaussian orientations; relies on Ripmap Encoding for effective anisotropic featurization [S1].

## Background
Platonic Solid Projection was proposed in Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids [S1]. It addresses limitations of orthogonal tri-plane representations (derived from a cube) by using various Platonic solids to provide more diverse plane orientations, improving the distinguishability of projected anisotropic 3D areas [S1].

## Sources
[S1] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
