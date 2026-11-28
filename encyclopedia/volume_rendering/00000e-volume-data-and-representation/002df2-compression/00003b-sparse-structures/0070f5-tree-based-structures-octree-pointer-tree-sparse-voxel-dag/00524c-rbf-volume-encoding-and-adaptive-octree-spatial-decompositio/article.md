# RBF Volume Encoding and Adaptive Octree Spatial Decomposition

Also called: Adaptive Octree for RBFs, RBF Encoding

## Overview
This method encodes volumetric scalar data as a weighted sum of Radial Basis Functions (RBFs) to eliminate the need for large computational grids and reduce CPU-to-GPU data transfer bottlenecks [S1]. It uses an adaptive octree to store lists of relevant basis functions per spatial region, thereby reducing the number of RBF evaluations required during rendering [S1]. It performs on-the-fly reconstruction of the RBF-encoded data during the rasterization of 2D slices, allowing arbitrary visualization algorithms like cutting planes or isosurfaces [S1].

## How it works
**Inputs:** A set of volumetric scalar data points $(x_j, y_j, z_j, v_j)$ where $v_j$ is the scalar value at position $\mathbf{p}_j$ [S1].
**Outputs:** An adaptive octree data structure containing lists of RBF parameters (centers, widths, weights) and error bounds [S1].

**Data Structures:**
1. **RBF Parameter:** A tuple $(\mathbf{c}_i, \sigma_i, w_i, \mu_i)$ stored in 2D textures [S1].
    * $\mathbf{c}_i$: Center coordinates $(x, y, z)$.
    * $\sigma_i$: Width parameter.
    * $w_i$: Weight.
    * $\mu_i$: Global bias term [S1].
2. **Octree Cell:**
    * **Bounding Box:** Axis-aligned bounds defining the spatial extent of the cell [G].
    * **RBF List:** A list of indices pointing to RBF parameters whose radius of influence intersects the cell [S1].
    * **Error Bounds:** Scalar values stored at each of the 8 corner points of the cell, representing the error introduced by truncating the RBF list to the $n$ most significant functions [S1].

**Algorithm Steps:**

1. **RBF Encoding (Off-line):**
    a. Initialize the RBF representation with a global bias $\mu$ [S1].
    b. Calculate encoding errors for all input data points: $E_j = v_j - \hat{v}(\mathbf{p}_j)$, where $\hat{v}$ is the current RBF approximation [S1].
    c. Identify data points with the largest errors [S1].
    d. Add new RBFs in clusters at these high-error locations [S1].
    e. Compute individual RBF weights $w_i$ and update the global bias $\mu$ by minimizing the sum squared error over all data points using the Pseudo-inverse method [S1]. The RBF width $\sigma_i$ is determined using a hybrid gradient-descent nonlinear optimization technique (Levenberg-Marquardt method) [S1].
    f. Repeat steps b-e until the user-specified error criteria is satisfied [S1].

2. **Radius of Influence Calculation:**
    For each Gaussian RBF $i$, calculate its radius of influence $r_i$ based on a user-defined error tolerance $\epsilon$:
    $$ r_i = \sigma_i \sqrt{2 \ln \left( \frac{|w_i|}{\epsilon} \right)} $$
    If $|w_i| \le \epsilon$, the logarithm is non-positive and $r_i$ is treated as zero [S1].

3. **Adaptive Octree Construction:**
    a. Start with a root node covering the entire volume domain [G].
    b. For each cell, determine the list of contributing basis functions by checking if their radius of influence $r_i$ intersects the cell [S1].
    c. **Termination Condition:** Subdivide a cell into 8 children if the number of contributing basis functions exceeds a threshold $n$ (maximum RBFs per cell) AND further subdivision significantly reduces the number of basis functions for the eight children cells [S1].
    d. If the number of basis functions is $\le n$, the cell is a leaf [S1].

4. **Error Estimation and Storage:**
    a. For each leaf cell, sort the contributing basis functions by magnitude $|w_i|$ [S1].
    b. Calculate the error introduced by evaluating only the top $n$ most significant basis functions per fragment [S1].
    c. Store the error at each of the 8 corner points of the cell in the cell data structure [S1].

5. **Interactive Reconstruction (On-line):**
    a. Generate slice polygons by intersecting a plane with the volume bounding box [S1].
    b. For each fragment on the slice:
        i. Traverse the octree to find the leaf cell containing the fragment's 3D position [G].
        ii. Retrieve the list of RBF indices and the corner error values for that cell [S1].
        iii. Interpolate the corner error values to estimate the local error bound for the fragment [S1].
        iv. Evaluate the RBF sum using only the $n$ stored basis functions:
        $$ \hat{v}(\mathbf{p}) = \mu + \sum_{k=1}^{n} w_k \phi(\|\mathbf{p} - \mathbf{c}_k\|, \sigma_k) $$
        where $\phi(r, \sigma) = e^{-(r/\sigma)^2}$ for Gaussian RBFs [G].
        v. Use the interpolated error bound to adjust visualization parameters [S1].

**Parameters:**
* $\epsilon$: User-defined error tolerance for calculating radius of influence [S1].
* $n$: Maximum number of RBFs per cell [S1].
* $\sigma_i$: RBF width, optimized via Levenberg-Marquardt [S1].

**Edge Cases and Pitfalls:**
* **Linear Artifacts:** With very small values of $n$, linear interpolation of the error bounds at cell corners may introduce visible artifacts [S1].
* **Precision:** The calculation of $r_i$ involves logarithms and square roots; care must be taken with numerical stability when $|w_i|$ is close to $\epsilon$ [G].
* **Boundary Conditions:** The radius of influence formula assumes the RBF decays to zero at $r_i$. Outside this radius, the RBF is ignored, which is valid given the error tolerance $\epsilon$ [G].

## Subtopics

### [RBF Encoding of Volume Data](14685-rbf-encoding-of-volume-data.md)
A method for encoding volumetric scalar data using a compact set of Radial Basis Functions (RBFs) to eliminate the need for large computational grids and reduce CPU-to-GPU data transfer bottlenecks. It mandates a per-fragment procedural reconstruction approach rather than splatting, making it suitable for hierarchical decomposition. Choose this method when interactive visualization of procedurally encoded fields is required without pre-rasterizing to 3D textures.

### [Interactive RBF Reconstruction via Volume Slicing](14698-interactive-rbf-reconstruction-via-volume-slicing.md)
A visualization approach that eliminates storing volume data in 3D textures, instead using the GPU fragment processor to perform on-the-fly reconstruction of RBF-encoded volumetric data during the rasterization of each slice. It sets itself apart by computing field values procedurally per fragment, supporting arbitrary visualization algorithms like cutting planes or isosurfaces. Choose this method when hardware-accelerated procedural evaluation is preferred over texture-based storage.

## Operating characteristics
* **Memory Complexity:** Storage is proportional to the number of RBFs plus the octree structure. The octree size depends on the spatial distribution of RBF centers and the threshold $n$ [S1].
* **Measured Performance:** The method allows for interactive visualization by eliminating the need to store volume data in 3D textures, relying instead on compact RBF parameters in 2D textures [S1].
* **Strengths:** Significant reduction in data transfer size; adaptive resolution via octree; procedural generation allows infinite resolution within the RBF support [S1].
* **Limitations:** Encoding phase can be computationally expensive; linear interpolation of error bounds can cause artifacts for very low $n$ [S1]; not suitable for non-scalar data without extension [S1].

## Background
The method was introduced by Jang, Weiler, Hopf, Huang, Ebert, Gaither, and Ertl in their paper "Interactively Visualizing Procedurally Encoded Scalar Fields" [S1]. It builds on earlier work on RBF interpolation [PFTV92, Alb72] and adapts it for interactive volume rendering by combining it with an adaptive octree spatial decomposition to manage the evaluation cost [S1].

## Sources
[S1] Interactively Visualizing Procedurally Encoded Scalar Fields, https://doi.org/10.2312/vissym/vissym04/035-044, fetched 2026-10-26
