# Improved Tetrahedral Mesh Generation for Voxelization
Also called: Robust Tetrahedral Voxelization, Tetrahedral Mesh Voxelization

## Overview
This method is a robust voxelization framework that converts a triangular mesh into solid binary voxel data by generating a high-quality intermediate tetrahedral mesh [S1, unofficial]. It specifically addresses the failure of existing methods to handle low-quality, non-watertight, or topologically open meshes common in internet datasets [S1, unofficial]. Unlike state-of-the-art approaches that connect triangle facets to a model centroid (producing faulty meshes for high-face models) or standard Delaunay triangulation (which suffers from boundary quality issues and high cost for sparse models), this method uses background grid insertion combined with mesh improvement to maintain quality and speed [S1, unofficial]. It achieves near 100% success rate on testing datasets compared to ~50% for other methods [S1, unofficial].

## How it works
**Inputs:**
1. A triangular mesh $M$ [S1, unofficial].
2. A user-defined error tolerance parameter $\epsilon$ [S1, unofficial].

**Outputs:**
1. A tetrahedral mesh $T$ generated within the error bound [S1, unofficial].
2. Solid binary voxel data derived from $T$ [S1, unofficial].

**Data Structures:**
* **Triangular Mesh $M$**: Defined by vertices $V$ and triangular faces $F_{tri}$ [G].
* **Tetrahedral Mesh $T$**: Defined by vertices $V_T$ and tetrahedral cells [G].
* **Background Grid**: A structured grid aligned with the bounding box of the input mesh, used as the base for insertion [S1, unofficial].

**Parameters:**
* $\epsilon$ (epsilon): User-defined tolerance for errors in tetrahedral mesh generation. It controls the magnitude of zigzag artifacts on straight lines. Typical tested values are $2 \times 10^{-3}$, $5 \times 10^{-3}$, and $1 \times 10^{-2}$ [S1, unofficial]. Errors up to $\epsilon = 10^{-2}$ (10x default) do not produce major differences in the final voxelized output [S1, unofficial].

**Algorithm Steps:**
1. **Input Validation/Preprocessing**: Accept the triangular mesh $M$ [S1, unofficial].
2. **Initial Mesh Improvement**: Perform mesh improvement processing on the input triangular mesh $M$ to enhance quality [S1, unofficial].
3. **Background Mesh Insertion**: Insert the improved triangular mesh into a background grid (background mesh insertion) [S1, unofficial]. This creates an initial tetrahedral mesh structure [G].
4. **Post-Insertion Mesh Improvement**: Perform mesh improvement processing on the resulting tetrahedral mesh to refine element quality and satisfy the error bound $\epsilon$ [S1, unofficial].
5. **Voxelization**: Rasterize the final tetrahedral mesh to produce solid binary voxel data [S1, unofficial]. This step relies on the stronger assumption of mesh validity provided by the improved tetrahedral generation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def improved_tetrahedral_voxelization(triangular_mesh, epsilon):
    # Step 1: Initial Mesh Improvement
    # Standard operation: Apply Laplacian smoothing and edge flips to improve triangle quality [G].
    improved_tri_mesh = apply_laplacian_smoothing_and_edge_flips(triangular_mesh)

    # Step 2: Background Mesh Insertion
    # Standard operation: Subdivide a uniform grid aligned with the bounding box, then insert mesh vertices and edges [G].
    initial_tet_mesh = insert_into_uniform_grid(improved_tri_mesh)

    # Step 3: Post-Insertion Mesh Improvement
    # Standard operation: Perform Delaunay refinement and edge swapping to satisfy quality and epsilon bound [G].
    final_tet_mesh = apply_delaunay_refinement_and_edge_swapping(initial_tet_mesh, epsilon)

    # Step 4: Voxelization
    # Standard operation: Evaluate each voxel center against the tetrahedral mesh to classify solid/empty [G].
    voxel_data = rasterize_tet_mesh_to_voxels(final_tet_mesh)

    return voxel_data
```

**Edge Cases and Pitfalls:**
* **Zigzag Artifacts**: Setting $\epsilon$ too large causes visible zigzag artifacts on straight lines [S1, unofficial]. However, the impact on the final voxel output is minimal even at $\epsilon = 10^{-2}$ [S1, unofficial].
* **Low-Quality Input**: The method is designed to handle meshes that cause other methods to fail, including non-watertight or topologically open meshes [S1, unofficial].
* **Sparse Models**: Standard grid insertion is computationally expensive for sparse models; this method mitigates this by combining insertion with improvement [S1, unofficial].

## Subtopics

### Tetrahedral Mesh from Voxel Division
A method to convert a coarse voxel representation into a tetrahedral mesh by splitting each hexahedron (voxel) into 6 tetrahedra to ensure face-to-face consistency [S2, unofficial].
* **Definition**: A tetrahedron is a pyramid-like polyhedron with four vertices and four triangular faces. A hexahedron has eight vertices and six quadrilateral faces [S2, unofficial].
* **Process**: The voxel-based shape is represented as a hexahedral mesh $H = (V, F_{quad}) = (\cup_c \{v_{c,i}\}_{i=1}^8, \cup_c \{f_{c,j}\}_{j=1}^6)$, where $V$ is the vertex set, $F_{quad}$ is the quadrilateral face set, $v_{c,i}$ is the $i$-th vertex of the $c$-th cube, and $f_{c,j}$ is the $j$-th quadrilateral face of the $c$-th cube [S2, unofficial]. This is then split into a tetrahedral mesh $T = (V, F_{tri}) = (\cup_c \cup_{k=1}^6 \{v_{c,k,i}\}_{i=1}^4, \cup_c \cup_{k=1}^6 \{f_{c,k,j}\}_{j=1}^4)$, where $V$ is the same vertex set, $F_{tri}$ is the triangular face set, $v_{c,k,i}$ is the $i$-th vertex of the $k$-th tetrahedron in the $c$-th cube, and $f_{c,k,j}$ is the $j$-th triangular face of the $k$-th tetrahedron in the $c$-th cube [S2, unofficial].
* **Parameter**: Number of tetrahedra per cube is 6 [S2, unofficial].

## Operating characteristics
* **Success Rate**: Achieved near 100% availability (success rate) on testing datasets (Princeton ModelNet and Thingi10k), whereas other methods achieved around 50% [S1, unofficial].
* **Performance**: Produces much higher quality tetrahedral meshes as an intermediate outcome, allowing for a faster voxelization algorithm based on stronger assumptions [S1, unofficial].
* **Scalability**: Combines background insertion with mesh improvement to maintain process speed, addressing the computational expense of standard grid insertion for sparse models [S1, unofficial].
* **Strengths**: Robustness to low-quality, non-watertight, or topologically open meshes; high success rate; ability to use $\epsilon$ as a data augmentation technique for 3D CNNs [S1, unofficial].
* **Limitations**: Requires careful selection of $\epsilon$ to balance zigzag artifacts and accuracy, though the method is tolerant up to $\epsilon = 10^{-2}$ [S1, unofficial].

## Background
The method was proposed by researchers from National Taiwan University [S1, unofficial]. It is based on the work of [32] and [33] (cited in the source), which involve background mesh insertion with mesh improvement processing [S1, unofficial]. The framework was tested on various datasets including Princeton ModelNet and Thingi10k [S1, unofficial]. The work includes a proof that the algorithm performs better than the state-of-the-art method [1] (which connects every triangle facet to the model centroid) [S1, unofficial].

## Sources
[S1] Robust Voxelization and Visualization by Improved Tetrahedral Mesh Generation, https://doi.org/10.48550/arxiv.2106.01326, fetched 2026-10-26 (unofficial)
[S2] Neural Volumetric Mesh Generator, https://arxiv.org/abs/2210.03158v1, fetched 2026-10-27 (unofficial)
