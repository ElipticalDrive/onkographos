# 6-Separating Surface Voxelization

Also called: Thinner Connected Voxelization, Thinner Surface Voxelization

## Overview
6-Separating Surface Voxelization is a surface voxelization criterion that produces a "thinner" set of voxels compared to conservative voxelization, while remaining gap-free and 26-connected [S1, S2]. Unlike conservative voxelization, which is 26-separating (and thus 6-connected), this method ensures that the resulting voxel set is 6-separating, meaning any two voxels in the set are connected via a path of voxels sharing a face, edge, or corner, but the separation property guarantees no diagonal gaps exist [S1]. It is derived by adapting the triangle/box overlap test used in conservative voxelization [S2].

## How it works
**Inputs:** A triangle defined by vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ and a voxel grid with spacing $\Delta \mathbf{p}$.
**Outputs:** A set of voxels intersected by the triangle, satisfying the 6-separating criterion.

**Data Structures:**
*   Triangle vertices: $\mathbf{v}_i = (x_i, y_i, z_i)$ for $i \in \{0, 1, 2\}$.
*   Normal vector: $\mathbf{n} = (n_x, n_y, n_z)$, normalized.
*   Voxel spacing: $\Delta \mathbf{p} = (\Delta x, \Delta y, \Delta z)$.

**Steps:**
1.  **Compute Normal:** Calculate the triangle normal $\mathbf{n} = (\mathbf{v}_1 - \mathbf{v}_0) \times (\mathbf{v}_2 - \mathbf{v}_0)$ and normalize it.
2.  **Determine Dominant Axis:** Identify the axis $k \in \{x, y, z\}$ with the largest absolute component in $\mathbf{n}$, i.e., $k = \arg \max_{c \in \{x,y,z\}} |n_c|$ [S1]. This optimization reduces the number of projection tests required [S1].
3.  **Plane Test (6-Separating):** Determine if the triangle intersects the slab of voxels along the dominant axis. The offsets for the plane test are modified from the conservative case [S1]:
    $$d_1 = \langle \mathbf{n}, \frac{1}{2} \Delta \mathbf{p} - \mathbf{v}_0 \rangle + \frac{1}{2} \Delta \mathbf{p}_{\perp} |\mathbf{n}_{\perp}|$$
    $$d_2 = \langle \mathbf{n}, \frac{1}{2} \Delta \mathbf{p} - \mathbf{v}_0 \rangle - \frac{1}{2} \Delta \mathbf{p}_{\perp} |\mathbf{n}_{\perp}|$$
    where $\Delta \mathbf{p}_{\perp}$ and $\mathbf{n}_{\perp}$ are the components of $\Delta \mathbf{p}$ and $\mathbf{n}$ perpendicular to the dominant axis $k$ [S1]. The term with the plus sign is chosen for the specific offset calculation based on the axis that maximizes the offset [S1]. Specifically, the selection of $d_1$ or $d_2$ depends on $\arg \max_{|\mathbf{n}_{\perp}| = x, y, z} d_2$ [S1].
4.  **2D Projection Tests:** For the two axes perpendicular to the dominant axis, perform 2D projection overlap tests. Adopt the criterion for 4-separating line pixelization from Huang et al. [S1]. For an edge $e_i$ projected onto the $xy$-plane, the offsets are:
    $$de^{xy}_i = \langle \mathbf{n}_{\mathbf{e}_i}^x, \frac{1}{2} \Delta \mathbf{p}_{xy} - \mathbf{v}_{i,xy} \rangle + \frac{1}{2} \Delta \mathbf{p}_{\mathbf{e}_i}^x |\mathbf{n}_{\mathbf{e}_i}^x|$$
    where the selection is based on $\arg \max_{\mathbf{p} \in \mathbf{p}_{xy}} |\mathbf{n}_{\mathbf{e}_i}^x|$ [S1]. Similar tests are performed for the other perpendicular projection.
5.  **Voxel Selection:** Mark voxels that satisfy the 6-separating plane test and the 2D projection tests. This results in a thinner set than conservative voxelization but maintains connectivity [S2].

**Parameters:**
*   $\Delta \mathbf{p}$: Voxel size. Typical values depend on the application resolution.

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** If the triangle area is zero, the normal is undefined. Handle by skipping or treating as a line/point.
*   **Precision:** Floating-point precision issues may arise in the dot products and comparisons. Use epsilon values for comparisons if necessary [G].
*   **Dominant Axis Selection:** If two components of the normal are equal in magnitude, the choice of dominant axis may affect the result slightly, but the method should remain valid [G].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels intersected by the triangle. The use of the dominant axis optimization reduces the number of projection tests [S1].
*   **Memory Complexity:** Proportional to the number of voxels in the output set. Since it is thinner than conservative voxelization, it uses less memory for the same mesh [S2].
*   **Performance:** Faster than previous conservative surface voxelization solutions, up to one order of magnitude [S2].
*   **Strengths:** Produces a gap-free, connected surface voxelization that is thinner than conservative voxelization, reducing memory usage and potentially improving rendering quality for thin surfaces [S2].
*   **Limitations:** May not be suitable for applications requiring strict 6-connectivity (e.g., certain morphological operations) as it produces 26-connected voxels [S1].

## Background
6-Separating Surface Voxelization was introduced by Michael Schwarz et al. in their 2010 paper "Fast Parallel Surface and Solid Voxelization on GPUs" [S1, S2]. It was developed to address the high memory consumption of conservative voxelization while maintaining connectivity and gap-free properties. The method adapts the triangle/box overlap test from conservative voxelization by using criteria from Huang et al. for plane and line voxelization [S1]. It is used in GPU-based voxelization pipelines for volume rendering and other graphics applications [S2].

## Sources
[S1] Fast parallel surface and solid voxelization on GPUs, https://doi.org/10.1145/1882261.1866201, fetched 2026-11-03
[S2] PDF Fast Parallel Surface and Solid Voxelization on GPUs, https://www.michael-schwarz.com/research/publ/files/vox-siga10.pdf, fetched 2026-10-27
