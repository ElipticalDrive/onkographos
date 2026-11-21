# RBF Volume Encoding and Adaptive Octree Spatial Decomposition

Also called: Adaptive Octree for RBFs, RBF Encoding

## Overview
This method encodes volumetric scalar data as a weighted sum of Radial Basis Functions (RBFs) to eliminate the need for large computational grids and reduce CPU-to-GPU data transfer bottlenecks [S1]. It uses an adaptive octree to store lists of relevant basis functions per spatial region, thereby reducing the number of RBF evaluations required during rendering [S1]. It differs from sibling methods like [Interactive RBF Reconstruction via Volume Slicing](14698-interactive-rbf-reconstruction-via-volume-slicing.md) primarily in its reconstruction strategy: while splatting with varying footprint sizes is incompatible with hierarchical decomposition due to significant overlap of RBF influence regions, this method mandates a per-fragment procedural reconstruction approach [S1].

## How it works
**Inputs:** A set of volumetric scalar data points $(x_j, y_j, z_j, v_j)$ where $v_j$ is the scalar value at position $\mathbf{p}_j$.
**Outputs:** An adaptive octree data structure containing lists of RBF parameters (centers, widths, weights) and error bounds, ready for GPU texture storage and procedural evaluation.

**Data Structures:**
1.  **RBF Parameter:** A tuple $(\mathbf{c}_i, \sigma_i, w_i, \mu_i)$ stored in 2D textures.
    *   $\mathbf{c}_i$: Center coordinates $(x, y, z)$.
    *   $\sigma_i$: Width parameter.
    *   $w_i$: Weight.
    *   $\mu_i$: Global bias term (shared or per-RBF depending on implementation, typically global in the cited source).
2.  **Octree Cell:**
    *   **Bounding Box:** Defines the spatial extent of the cell.
    *   **RBF List:** A list of indices pointing to RBF parameters whose radius of influence intersects the cell.
    *   **Error Bounds:** Scalar values stored at each of the 8 corner points of the cell, representing the maximum error introduced by truncating the RBF list to the $n$ most significant functions.

**Algorithm Steps:**

1.  **RBF Encoding (Off-line):**
    a.  Initialize the RBF representation with a global bias $\mu$.
    b.  Calculate encoding errors for all input data points: $E_j = v_j - \hat{v}(\mathbf{p}_j)$, where $\hat{v}$ is the current RBF approximation.
    c.  Identify data points with the largest errors.
    d.  Add new RBFs in clusters at these high-error locations.
    e.  Compute individual RBF weights $w_i$ and update the global bias $\mu$ by minimizing the sum squared error over all data points. This is achieved using the Levenberg-Marquardt method for nonlinear optimization of the RBF width $\sigma_i$, and the Pseudo-inverse method for linear weights [S1].
    f.  Repeat steps b-e until the user-specified error criteria is satisfied.

2.  **Radius of Influence Calculation:**
    For each Gaussian RBF $i$, calculate its radius of influence $r_i$ based on a user-defined error tolerance $\epsilon$:
    $$ r_i = \sigma_i \sqrt{2 \ln \left( \frac{|w_i|}{\epsilon} \right)} $$
    If the term inside the logarithm is $\le 1$, $r_i$ is effectively zero or undefined for practical intersection purposes [S1].

3.  **Adaptive Octree Construction:**
    a.  Start with a root node covering the entire volume domain.
    b.  For each cell, determine the list of contributing basis functions by checking if their radius of influence $r_i$ intersects the cell's bounding box.
    c.  **Termination Condition:** Subdivide a cell into 8 children if:
        i.  The number of contributing basis functions in the cell exceeds a threshold $n$ (maximum RBFs per cell).
        ii. AND further subdivision significantly reduces the number of basis functions for the eight children cells.
    d.  If the number of basis functions is $\le n$, the cell is a leaf.

4.  **Error Estimation and Storage:**
    a.  For each leaf cell, sort the contributing basis functions by magnitude $|w_i|$.
    b.  Calculate the error introduced by evaluating only the top $n$ most significant basis functions for points within the cell.
    c.  Store the maximum estimated error at each of the 8 corner points of the cell in the cell data structure [S1].

5.  **Interactive Reconstruction (On-line):**
    a.  Generate slice polygons by intersecting a plane with the volume bounding box.
    b.  For each fragment (pixel) on the slice:
        i.  Traverse the octree to find the leaf cell containing the fragment's 3D position.
        ii. Retrieve the list of RBF indices and the corner error values for that cell.
        iii. Interpolate the corner error values to estimate the local error bound for the fragment.
        iv. Evaluate the RBF sum using only the $n$ stored basis functions:
        $$ \hat{v}(\mathbf{p}) = \mu + \sum_{k=1}^{n} w_k \phi(\|\mathbf{p} - \mathbf{c}_k\|, \sigma_k) $$
        where $\phi(r, \sigma) = e^{-(r/\sigma)^2}$ for Gaussian RBFs.
        v.  Use the interpolated error bound to potentially adjust opacity or color mapping, or to trigger refinement if higher accuracy is required [S1].

**Parameters:**
*   $\epsilon$: User-defined error tolerance for calculating radius of influence. Typical values depend on the dynamic range of the data.
*   $n$: Maximum number of RBFs per cell. Typical values are small integers (e.g., 1-8) to maintain interactivity.
*   $\sigma_i$: RBF width, optimized via Levenberg-Marquardt [S1].

**Edge Cases and Pitfalls:**
*   **Linear Artifacts:** With very small values of $n$, linear interpolation of the error bounds at cell corners may introduce visible artifacts [S1].
*   **Precision:** The calculation of $r_i$ involves logarithms and square roots; care must be taken with numerical stability when $|w_i|$ is close to $\epsilon$.
*   **Boundary Conditions:** The radius of influence formula assumes the RBF decays to zero at $r_i$. Outside this radius, the RBF is ignored, which is valid given the error tolerance $\epsilon$.

## Subtopics

### [RBF Encoding of Volume Data](14685-rbf-encoding-of-volume-data.md)
A method for encoding volumetric scalar data using a compact set of Radial Basis Functions (RBFs) to eliminate the need for large computational grids and reduce CPU-to-GPU data transfer bottlenecks. It differs from sibling methods like [Interactive RBF Reconstruction via Volume Slicing](14698-interactive-rbf-reconstruction-via-volume-slicing.md) primarily in its reconstruction strategy: while splatting with varying footprint sizes is incompatible with hierarchical decomposition due to significant overlap of RBF influence regions, this method mandates a per-fragment procedural reconstruction approach.

### [Interactive RBF Reconstruction via Volume Slicing](14698-interactive-rbf-reconstruction-via-volume-slicing.md)
A visualization approach that eliminates storing volume data in 3D textures, instead using the GPU fragment processor to perform on-the-fly reconstruction of RBF-encoded volumetric data during the rasterization of each slice.

## Operating characteristics
*   **Time Complexity:** Encoding is $O(N \cdot M)$ where $N$ is the number of data points and $M$ is the number of RBFs added, dominated by the Levenberg-Marquardt optimization and Pseudo-inverse calculations [S1]. Rendering is $O(1)$ per fragment relative to the total number of RBFs, as only $n$ RBFs are evaluated per fragment, where $n$ is a small constant [S1].
*   **Memory Complexity:** Storage is proportional to the number of RBFs $M$ plus the octree structure. The octree size is proportional to the number of leaf cells, which depends on the spatial distribution of RBF centers and the threshold $n$ [S1].
*   **Measured Performance:** The method allows for interactive visualization by eliminating the need to store volume data in 3D textures, relying instead on compact RBF parameters in 2D textures [S1].
*   **Strengths:** Significant reduction in data transfer size; adaptive resolution via octree; procedural generation allows infinite resolution within the RBF support.
*   **Limitations:** Encoding phase can be computationally expensive; linear interpolation of error bounds can cause artifacts for very low $n$ [S1]; not suitable for non-scalar data without extension.

## Background
The method was introduced by Jang, Weiler, Hopf, Huang, Ebert, Gaither, and Ertl in their paper "Interactively Visualizing Procedurally Encoded Scalar Fields" [S1]. It builds on earlier work on RBF interpolation [PFTV92, Alb72] and adapts it for interactive volume rendering by combining it with an adaptive octree spatial decomposition to manage the evaluation cost.

## Sources
[S1] Interactively Visualizing Procedurally Encoded Scalar Fields, https://doi.org/10.2312/vissym/vissym04/035-044, fetched 2026-10-26
