# Edge Chamfer Distance

Also called: ECD

## Overview
Edge Chamfer Distance (ECD) is a metric that calculates the symmetric distance between two point clouds sampled specifically from the edges of meshes, rather than their surfaces [S1]. It solves the problem of evaluating edge reconstruction quality and geometric sharpness, which standard surface-based Chamfer Distance fails to capture adequately [S1]. Unlike its parent method [Chamfer Distance](2446-chamfer-distance.md) which assesses global surface fidelity, ECD isolates the structural integrity of sharp features and boundaries.

## How it works
**Inputs:**
1.  $\mathcal{E}_r$: A set of points sampled from the edges of the reconstructed mesh.
2.  $\mathcal{E}_{\text{gt}}$: A set of points sampled from the edges of the ground truth mesh.

**Outputs:**
*   $\text{ECD}$: The symmetric Chamfer Distance value between the two edge point sets.

**Data Structures:**
*   $\mathcal{E}_r, \mathcal{E}_{\text{gt}}$: Unordered sets of 3D coordinates $(x, y, z)$. The sampling density should be consistent between reconstruction and ground truth to ensure fair comparison [S1, S2].

**Steps:**
1.  **Sampling:** Uniformly sample points from the edges of the reconstructed mesh to form $\mathcal{E}_r$ and from the ground truth mesh to form $\mathcal{E}_{\text{gt}}$ [S1, S2].
2.  **Distance Calculation:** Compute the symmetric Chamfer Distance using the loss function $\mathcal{L}_{\text{CD}}$ [S2].

**Formula:**
$$ \text{ECD} = \mathcal{L}_{\text{CD}}(\mathcal{E}_r, \mathcal{E}_{\text{gt}}) $$

Where $\mathcal{L}_{\text{CD}}$ is defined as the standard symmetric Chamfer Distance:
$$ \mathcal{L}_{\text{CD}}(A, B) = \frac{1}{|A|} \sum_{a \in A} \min_{b \in B} \|a - b\|^2 + \frac{1}{|B|} \sum_{b \in B} \min_{a \in A} \|b - a\|^2 $$

**Parameters:**
*   Sampling density: Not specified in sources [G]. Implementers must ensure equal or comparable sampling rates for $\mathcal{E}_r$ and $\mathcal{E}_{\text{gt}}$.

**Edge Cases:**
*   If either mesh has no edges (e.g., a closed manifold with no sharp features or boundaries depending on the variant), the set is empty, and the distance is undefined or zero depending on implementation convention [G].

## Subtopics

### Non-manifold Chamfer Distance
Evaluates the reconstruction quality of non-manifold structures by comparing point sets sampled from the non-manifold edges of the reconstruction ($\mathcal{E}_r^{\text{nm}}$) and the ground truth ($\mathcal{E}_{\text{gt}}^{\text{nm}}$) [S2]. It is chosen when assessing the fidelity of complex topological features like T-junctions or multi-material intersections [S2].
$$ \text{CD}_{\text{nme}} = \mathcal{L}_{\text{CD}}(\mathcal{E}_r^{\text{nm}}, \mathcal{E}_{\text{gt}}^{\text{nm}}) \quad (4) $$

### Boundary Chamfer Distance
Assesses the geometric continuity of open surfaces by comparing point sets sampled from the boundary edges of the reconstructed mesh ($\mathcal{E}_r^{\text{bd}}$) and the ground truth ($\mathcal{E}_{\text{gt}}^{\text{bd}}$) [S2]. It is used to evaluate how well open boundaries are preserved [S2].
$$ \text{CD}_{\text{bde}} = \mathcal{L}_{\text{CD}}(\mathcal{E}_r^{\text{bd}}, \mathcal{E}_{\text{gt}}^{\text{bd}}) \quad (5) $$

## Operating characteristics
*   **Time Complexity:** Dominated by the nearest-neighbor search between the two edge point sets, typically $O(N \log N)$ or $O(N^2)$ depending on the spatial indexing structure used for $\mathcal{L}_{\text{CD}}$ [G].
*   **Memory Complexity:** Linear in the number of sampled edge points $N$ [G].
*   **Strengths:** Specifically targets sharp features and structural boundaries, providing a more granular evaluation than global surface metrics [S1, S2].
*   **Limitations:** Sensitive to sampling density; uneven sampling can skew results [G]. Does not evaluate surface interior quality [S1].
*   **When to use:** When sharp edges, non-manifold connections, or open boundaries are critical to the application (e.g., CAD models, architectural geometry) [S1, S2].
*   **When not to use:** When only smooth surface fidelity is required, or when the mesh has no distinct edges [S1].

## Background
Edge Chamfer Distance was introduced as a specialized variant of Chamfer Distance to better gauge edge reconstruction quality and sharpness in 3D shape evaluation [S1]. It is particularly relevant in the context of neural implicit surface reconstruction and dual contouring methods where preserving sharp features is a known challenge [S1, S2]. The variants for non-manifold and boundary edges were formalized in the context of Dual Contouring over Expanded Cubes (DCx) for zero-level set extraction from neural unsigned distance functions [S2].

## Sources
[S1] Self-Supervised Dual Contouring, https://doi.org/10.1109/cvpr52733.2024.00448, fetched 2026-10-26
[S2] Dual Contouring over Expanded Cubes (DCx) for Zero-Level Set Extraction from Neural Unsigned Distance Functions, https://doi.org/10.1145/3811388, fetched 2026-11-06
