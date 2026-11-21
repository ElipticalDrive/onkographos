# Tetrahedralization Schemes
## Overview
Tetrahedralization Schemes define how a cubic voxel cell is decomposed into tetrahedra to facilitate isosurface extraction, addressing the topological ambiguities and artifacts inherent in standard cube-based methods like Marching Cubes. Unlike sibling methods that may use fixed lookup tables or specific lattice constraints, this family focuses on the geometric partitioning strategy itself—determining vertex counts, tetrahedron shapes, and neighbor dependencies. It sets itself apart by offering alternatives to the standard 5-tetrahedron or 6-tetrahedron decompositions, such as schemes using 12 or 24 tetrahedra, or those based on non-cubic lattices like BCC or Diamond, to improve surface smoothness or reduce lookup table memory.
## Shared principles
The core mechanism involves replacing the standard cubic voxel with a set of non-overlapping tetrahedra that fill the same volume. For each tetrahedron, the scalar field values at its four vertices are used to determine the intersection with an isosurface threshold $T$. The algorithm checks the sign of $(S_i - T)$ for each vertex $i$. If the signs differ across an edge, a linear interpolation is performed to find the intersection point $P$:

$$ P = V_1 + \frac{T - S_1}{S_2 - S_1} (V_2 - V_1) $$

where $V_1, V_2$ are the vertex positions and $S_1, S_2$ are their scalar values. The collection of these intersection points forms the vertices of the output triangle mesh. The specific scheme dictates the connectivity of these tetrahedra, whether new "fabricated" points (like cell centers) are introduced, and how adjacent cells share boundaries to ensure a manifold mesh.
## Subtopics

### [BCC Marching Tetrahedra with Permutation-Based Lookup](3414-bcc-marching-tetrahedra-with-permutation-based-lookup.md)
Extracts isosurfaces from Body-Centered Cubic (BCC) datasets by decomposing cells into tetrahedra and using permutation transformations to map a single reference tetrahedron's connectivity to all six cell types, significantly reducing GPU memory overhead for lookup tables compared to naive approaches. Choose this when working with BCC data and needing to minimize memory usage for large-scale GPU rendering while maintaining connected mesh structures.

### [Marching Tetrahedra 3 (DLC-Plane)](3632-marching-tetrahedra-3-dlc-plane.md)
Uses a diamond crystal lattice decomposition to split cells into exactly 5 tetrahedra using only 5 vertex points, eliminating fabricated center points and the associated creases/indentations, while processing data plane-by-plane to optimize point handling. Choose this for smooth surface extraction in void areas and when avoiding the artifacts of center-point synthesis is critical.

### Centered Cubic Lattice
A method that splits the cube cell into 24 tetrahedra by introducing a new data value at the center of gravity of the processed cell, computed as the arithmetic or weighted mean of the cell's vertices [S1, unofficial]. All 24 tetrahedra are identical in dimensions, similar to the MT6 method, but require interpolation on new edges among adjacent central points [S1, unofficial]. This scheme provides better detail in some aspects but has unacceptable computational time and memory requirements compared to 6-tetrahedra schemes [S2, unofficial].

### 12 Tetrahedra Scheme
A decomposition scheme using 12 tetrahedra per voxel, based on a body-centred cubic lattice with two neighboring voxels, where all tetrahedra are identical with edge lengths $a$, $a\sqrt{3}$, $a$, $a\sqrt{3}$ if $2a$ is the voxel's edge length [S2, unofficial]. It requires considering more neighbors for iso-surface extraction than the body-centred tetrahedronal scheme [S2, unofficial].

### Diamond Lattice Cell (DCL) Tetrahedra
An alternative cube slicing method where cubical cells are meshed into 5 tetrahedra using a (diamond cubic) lattice as a basis, mating cubes with opposite alignments to provide additional planes of symmetry [S3]. It tiles cells such that odd/even position sums determine orientation, ensuring near cells use consistent diagonals. It involves additional calculations for diagonals across cube-faces compared to Dual Contouring and does not address cases where two near 'inside' points are a combined distance < 1 from the surface [S3].

## Background
The need for tetrahedralization arose from the topological ambiguities in Marching Cubes, where certain configurations could produce non-manifold surfaces or artifacts. Early work by Baxley and others explored various decompositions [S2, unofficial]. The Centered Cubic Lattice method was compared in studies of fundamental iso-surface extraction methods [S1, unofficial]. The Diamond Lattice Cell approach is discussed in the context of alternative cube slicing and its relationship to Dual Contouring [S3]. The BCC permutation-based lookup is a modern GPU-optimized variant addressing memory constraints [S1, unofficial].

## Sources
[S1] A Comparison of Fundamental Methods for Iso-surface Extraction, https://doi.org/10.48550/arxiv.2301.01715, fetched 2026-10-29 (unofficial)
[S2] Two Methods for Iso-Surface Extraction from Volumetric Data and Their Comparison, https://arxiv.org/abs/2201.03446v1, fetched 2026-10-26 (unofficial)
[S3] Marching tetrahedra - HandWiki, https://handwiki.org/wiki/Marching_tetrahedra, fetched 2026-10-29
