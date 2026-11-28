# Sharp Conservative Offset for Marching Cubes
Also called: Sharp Enclosure Offset

## Overview
This method determines the sharp infimum offset $\sigma_\star$ for a grid-sampled 1-Lipschitz function such that the true zero-level set is strictly contained within the interior of the Marching Cubes mesh extracted at any level $\sigma > \sigma_\star$ [S1, unofficial]. It solves the problem of ensuring strict geometric containment for safety-critical applications like collision detection, where standard Marching Cubes may fail to enclose the true surface if the offset is too small [S1, unofficial]. Unlike methods that bound the distance between a trilinearly interpolated surface and the mesh, this method proves enclosure directly for the planar Marching Cubes mesh using a convex lift of the cell corners [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 1-Lipschitz function $f: \mathbb{R}^3 \rightarrow \mathbb{R}$ sampled on a regular Cartesian grid $h\mathbb{Z}^3$ with spacing $h > 0$. The region $f < 0$ is defined as "inside" and $f > 0$ as "outside".
*   **Output:** The sharp infimum offset $\sigma_\star$ and the guarantee that for any $\sigma > \sigma_\star$, the zero-level set $f^{-1}(0)$ is contained in $\text{inside}(\mathcal{M}_\sigma)$, where $\mathcal{M}_\sigma$ is the watertight Marching Cubes mesh.

**Data Structures:**
*   **Grid:** A regular Cartesian grid with nodes $v \in h\mathbb{Z}^3$.
*   **Function Values:** Scalar values $f(v)$ stored at each grid node.
*   **Cell Corners:** For each unit cell $[0, h]^3$, the 8 corner vertices $v_i$ and their function values $f(v_i)$.

**Parameters:**
*   $h$: Grid spacing. Must be $h > 0$.
*   $\sigma$: The isosurface level. Must satisfy $\sigma > \sigma_\star$ for the enclosure guarantee.
*   $K$: Lipschitz constant. Default is $K=1$.

**Sharp Infimum Formula:**
For a uniform grid with spacing $h$ and a 1-Lipschitz function, the sharp infimum is:
$$ \sigma_\star = \frac{\sqrt{3}}{2}h $$
Generalizations:
*   Anisotropic grid spacing $h_x, h_y, h_z$: $\sigma_\star = \frac{1}{2}\sqrt{h_x^2 + h_y^2 + h_z^2}$.
*   Dimension $d$: $\sigma_\star = \frac{h\sqrt{d}}{2}$.
*   $K$-Lipschitz function: $\sigma_\star = \frac{h K \sqrt{3}}{2}$.

**Algorithm Steps:**
1.  **Sample Function:** Evaluate the 1-Lipschitz function $f$ at all nodes of the Cartesian grid $h\mathbb{Z}^3$.
2.  **Select Level:** Choose an extraction level $\sigma$ such that $\sigma > \sigma_\star$.
3.  **Extract Vertices:** For every grid edge connecting nodes $a$ and $b$:
    *   Check if the classification $f_a < \sigma$ differs from $f_b < \sigma$.
    *   If they differ, compute the linear interpolation parameter $t$ such that the vertex $q$ lies on the edge:
        $$ q = (1 - t)a + tb $$
        $$ \sigma = (1 - t)f(a) + tf(b) $$
    *   Emit vertex $q$. Node $a$ is the low-side vertex ($f_a < \sigma$) and $b$ is the high-side vertex ($f_b \geq \sigma$).
4.  **Extract Faces:** Within each grid cell, connect the emitted edge-crossing vertices into triangles according to the standard Marching Cubes case resolution. The resulting mesh $\mathcal{M}_\sigma$ is a watertight, sign-respecting separator.
5.  **Guarantee:** The interior of $\mathcal{M}_\sigma$ (the union of all per-cell low-side regions) contains the true zero-level set $f^{-1}(0)$.

**Proof Sketch (Convex Lift):**
The proof constructs a convex set $P$ for each cell, defined as the set of points $x$ that can be written as a convex combination of the 8 cell corners $v_i$ with weights $\alpha_i \geq 0$ summing to 1, such that the weighted average of the function values is at least $\sigma$:
$$ x \in P \iff \exists \alpha \text{ such that } \begin{cases} x = \sum_{i=1}^8 \alpha_i v_i, \\ \alpha_i \geq 0, \sum_{i=1}^8 \alpha_i = 1, \\ \sum_{i=1}^8 \alpha_i f(v_i) \geq \sigma. \end{cases} $$
Both the high-side cell corners and the edge-crossing vertices lie in $P$. The boundary of the high-side region $R$ (bounded by planar faces connecting these vertices) is contained in the projection of $P$. If a point $p$ with $f(p)=0$ existed in the high-side region, it would imply:
$$ \sigma \leq \sum_{i=1}^8 \alpha_i f(v_i) \leq \sum_{i=1}^8 \alpha_i \|v_i - p\| \leq \sqrt{\sum_{i=1}^8 \alpha_i \|v_i - p\|^2} \leq \frac{\sqrt{3}}{2}h $$
The last inequality follows from Jensen's inequality and the variance bound of the cube corners. Thus, if $\sigma > \frac{\sqrt{3}}{2}h$, no such point $p$ can exist in the high-side region, ensuring $f^{-1}(0)$ is inside the mesh.

**Edge Cases and Pitfalls:**
*   **Insufficient Offset:** If $\sigma < \frac{\sqrt{3}}{2}h$, the guarantee fails. For example, a signed distance function to a small sphere at the cell center with radius $r < \frac{\sqrt{3}}{2}h - \sigma$ will have all grid vertices on the high side, resulting in no triangles being emitted, while the true sphere $f=0$ exists inside the cell [S1, unofficial].
*   **Lipschitz Assumption:** The method requires $f$ to be 1-Lipschitz. If $f$ is not Lipschitz, the bound does not hold.
*   **Grid Regularity:** The formula assumes a regular Cartesian grid. For anisotropic grids, the generalized formula must be used.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of grid cells, as it requires a single pass over the grid to evaluate the function and extract the mesh.
*   **Memory Complexity:** $O(N)$ to store the grid function values and the resulting mesh.
*   **Strengths:** Provides a strict mathematical guarantee of containment for the zero-level set, which is critical for collision detection and safety-critical applications. It is computationally inexpensive as it relies on the standard Marching Cubes algorithm.
*   **Limitations:** The guarantee only holds for 1-Lipschitz functions. The required offset $\sigma_\star$ may be large relative to the grid spacing, potentially leading to a coarser representation of the surface if a small offset is desired.
*   **When to Use:** When strict containment of the true geometry is required, such as in cage construction for decimation engines or swept volume extraction for collision tasks.
*   **When Not to Use:** When the function is not Lipschitz, or when a tighter surface approximation is needed and containment is not a strict requirement.

## Background
The method was introduced by Alec Jacobson from the University of Toronto [S1, unofficial]. It addresses the limitations of standard Marching Cubes, which can fail to enclose the true zero-level set if the extraction level is too close to the grid values. The proof utilizes a convex lift of the corner samples and Jensen's inequality to establish the sharp bound. The technique is related to Lipschitz pruning in rendering, where the same bound is used to reject grid cells that do not contain the zero-level set.

## Subtopics

### [Isosurface Extraction API](836-isosurface-extraction-api.md)
Provides a standardized interface for isosurface extraction methods, including Marching Cubes.

### [VESTA](900-vesta.md)
A method for volume rendering and surface extraction, potentially using different containment guarantees.

### [Marching Squares](918-marching-squares.md)
The 2D analog of Marching Cubes, used for contour extraction in 2D grids.

### [Extended CLCIR PLIC Reconstruction](954-extended-clcir-plic-reconstruction.md)
A method for reconstructing interfaces in volume-of-fluid methods, focusing on geometric accuracy.

### [Point Cloud Extraction](1196-point-cloud-extraction.md)
Extracts surfaces from point clouds, offering an alternative to grid-based methods.

### [Canonical Enumeration](1779-canonical-enumeration.md)
A method for enumerating topological configurations in isosurface extraction.

### [Hybrid Memory Constrained Isosurface Extraction](2307-hybrid-memory-constrained-isosurface-extraction.md)
Optimizes memory usage for large-scale isosurface extraction.

### [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md)
Details the lookup tables and configurations used in Marching Cubes.

### [Surface Properties and Estimation](2790-surface-properties-and-estimation.md)
Estimates surface properties like normals and curvature from extracted meshes.

### [General-purpose Contour Extraction Routine Generator](3666-general-purpose-contour-extraction-routine-generator.md)
Generates code for contour extraction routines.

### [THREAD System](3743-thread-system.md)
A system for parallel isosurface extraction.

### [DCX Mesh Extraction via UDF and Marching Cubes](3778-dcx-mesh-extraction-via-udf-and-marching-cubes.md)
Extracts meshes from DCX files using Marching Cubes.

### [Domain Search Isosurface Extraction](3871-domain-search-isosurface-extraction.md)
Uses domain search to optimize isosurface extraction.

### [Incremental Contouring via Power Diagrams](4955-incremental-contouring-via-power-diagrams.md)
Extracts contours using power diagrams.

### [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md)
Alternative methods for isosurface extraction that can produce higher-quality meshes.

### [Neural and Learning-based Extraction](6371-neural-and-learning-based-extraction.md)
Uses neural networks for isosurface extraction.

### [Cuberille](8276-cuberille.md)
A method for extracting voxel-based surfaces.

### [Visualization and Rendering Techniques](8434-visualization-and-visualization-techniques.md)
Techniques for visualizing and rendering isosurfaces.

### [GPU and Shader-Based Extraction](9036-gpu-and-shader-based-extraction.md)
Accelerates isosurface extraction using GPUs.

### [Grid Traversal and Data Structures](10325-grid-traversal-and-data-structures.md)
Data structures for efficient grid traversal in isosurface extraction.

### [Multi-Isosurface and Generalized Contouring](10723-multi-isosurface-and-generalized-contouring.md)
Extracts multiple isosurfaces simultaneously.

### [Unstructured and Arbitrary Grid Methods](11103-unstructured-and-arbitrary-grid-methods.md)
Methods for isosurface extraction on unstructured grids.

### [Interior Volume Tetrahedralization](11141-interior-volume-tetrahedralization.md)
Tetrahedralizes the interior volume for isosurface extraction.

### [Marching Tetrahedra and Decomposition](11174-marching-tetrahedra-and-decomposition.md)
An alternative to Marching Cubes using tetrahedral decomposition.

### [Isosurface Level Configuration](11533-isosurface-level-configuration.md)
Configures the isosurface level for extraction.

### [Multiresolution and Adaptive Extraction](11893-multiresolution-and-adaptive-extraction.md)
Adapts the grid resolution for isosurface extraction.

### [Memory and Performance Optimization](12239-memory-and-performance-optimization.md)
Optimizes memory and performance for isosurface extraction.

### [Volume-to-Mesh Conversion](12671-volume-to-mesh-conversion.md)
Converts volume data to mesh representations.

### [Regular Isosurface Extraction](13552-regular-isosurface-extraction.md)
Extracts isosurfaces from regular grids.

### [n-Dimensional and Hyper-surface Methods](13662-n-dimensional-and-hyper-surface-methods.md)
Extends isosurface extraction to higher dimensions.

### [Contour Filtering](17678-contour-filtering.md)
Filters contours to remove noise.

### [Region-growing and Advancing Front](17871-region-growing-and-advancing-front.md)
Uses region-growing for isosurface extraction.

### [VolumetricFire Slice Contouring](18010-volumetricfire-slice-contouring.md)
Extracts contours from volumetric data slices.

### [Topology-Preserving and Combinatorial Methods](18063-topology-preserving-and-combinatorial-methods.md)
Preserves topology during isosurface extraction.

### [Hermite Data Generation from Voxel Grid](18125-hermite-data-generation-from-voxel-grid.md)
Generates Hermite data for smoother isosurfaces.

### [Bump-based Marching Cubes](19596-bump-based-marching-cubes.md)
Adds bumps to Marching Cubes meshes.

### [Isosurface Extraction via Ray Marching](20054-isosurface-extraction-via-ray-marching.md)
Uses ray marching for isosurface extraction.

### [Dual Isosurface Intersection](22472-dual-isosurface-intersection.md)
Extracts the intersection of two isosurfaces.

### [Surface representation](22781-surface-representation.md)
Represents surfaces for isosurface extraction.

### [Contour and Line Extraction](22941-contour-and-line-extraction.md)
Extracts contours and lines from volume data.

## Sources
[S1] A Sharp Conservative Offset for Marching Cubes, https://arxiv.org/abs/2609.13430v1, fetched 2026-10-26 (unofficial)
