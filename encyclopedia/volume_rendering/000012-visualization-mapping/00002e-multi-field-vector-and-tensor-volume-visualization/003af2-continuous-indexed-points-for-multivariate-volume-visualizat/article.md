# Continuous Indexed Points for Multivariate Volume Visualization

Also called: Indexed Points Computation, Indexed points calculation, Parallel coordinates indexed points

## Overview
Continuous indexed points is a technique for multivariate volume visualization that encodes local correlations of multi-attribute data by representing linear structures in parallel coordinates [S1]. It improves spatial perception and interactive exploration by visualizing local linear fitting results as a density representation of indexed points in a continuous domain [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A multivariate volume dataset $\tau: \mathbb{R}^n \to \mathbb{R}^m$ (typically $n=3$ for volumetric data) [S1].
*   **Output:** A volume-rendered image with interactive multivariate transfer function widgets operating in the parallel coordinates image plane [S1].

**Data Structures:**
*   **Hierarchical Spatial Data Structure:** An octree used to accelerate local neighborhood queries for principal component analysis (PCA) [S1]. It stores reconstructed local linear information $(\xi, e_1, e_2)$ for every sample in a new multivariate volume [S1].
*   **1-flat and 2-flat Indexed Points:** Represent linear structures where a 1-flat corresponds to a line (correlation between two variables) and a 2-flat corresponds to a plane (correlation between three variables) [S1].

**Steps:**
1.  **Saliency Map and Octree Construction:** Compute a saliency map $S_f(x) = \zeta(x) * S(x)$ by convolving the gradient magnitude $S(x)$ with a low-pass filter $\zeta$ [S1]. Build an octree where a node subdivides if the value range of $S_f$ in the region exceeds threshold $t_s$ [S1].
2.  **Local Linear Fitting:** For each volume sample $\mathbf{x}$, perform local PCA on its spatial neighborhood to extract the data value $\xi = \tau(\mathbf{x})$ and eigenvector matrix $X \in \mathbb{R}^{p \times m}$ [S1]. If a leaf node's variance is below threshold $t_e$, reuse the node's linear fit for all samples within it [S1].
3.  **1-flat Indexed Points:** Compute $\eta_{1,2}, \dots, \eta_{m-1,m}$ by projecting $\xi$ and $\xi+e_1$ to all $(m-1)$ 2D subspaces spanned by adjacent attribute pairs in parallel coordinates [S1].
4.  **2-flat Indexed Points:** Compute $\eta_{1,2,3}, \dots, \eta_{m-2,m-1,m}$ by calculating the normal vector $n = e_1 \times e_2$, then projecting $\xi$ and $\xi+n$ to all $(m-2)$ 3D subspaces visualized as points on 2D image planes [S1].
5.  **Angle-Uniform Transformation:** Apply an angle-uniform transformation to bound and symmetrize the indexed points for visualization [S1].
6.  **Density Representation:** Generate a density representation using kernel density estimation (KDE) with dynamic bandwidth [S1]. The density $\rho(\boldsymbol{\tau})$ on the image domain is computed using geometric measure theory as $\rho(\boldsymbol{\tau}) = \frac{s(x)}{|\text{Vol}(Dh)(x)|}$ integrated over $h^{-1}(\boldsymbol{\tau})$ with respect to $d^{(n-2)}x$ [S1].
7.  **Rendering and Interaction:** Render the volume using an interactive occlusion shading model to facilitate spatial perception of volumetric correlation characteristics [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def continuous_indexed_points(volume_data, m, n=3, t_s=0.0, t_e=0.0):
    # Step 1: Saliency map and Octree construction
    gradient_magnitude = compute_gradient_magnitude(volume_data)
    saliency_map = convolve_low_pass(gradient_magnitude)
    octree = build_octree(saliency_map, t_s)

    indexed_points_1_flat = []
    indexed_points_2_flat = []

    # Step 2: Local PCA and reconstruction
    for leaf in octree.leaves():
        neighbors = leaf.get_spatial_neighbors()
        xi, e1, e2 = local_pca(neighbors, volume_data)
        samples = leaf.get_samples() if leaf.variance < t_e else [leaf.center]

        for x in samples:
            # Step 3: 1-flat indexed points (2 variables)
            for i in range(m - 1):
                p1 = project_to_2d_subspace(xi, i, i+1)
                p2 = project_to_2d_subspace(xi + e1, i, i+1)
                indexed_points_1_flat.append((p1, p2))

            # Step 4: 2-flat indexed points (3 variables)
            n_vec = cross_product(e1, e2)
            for i in range(m - 2):
                q1 = project_to_3d_subspace(xi, i, i+1, i+2)
                q2 = project_to_3d_subspace(xi + n_vec, i, i+1, i+2)
                indexed_points_2_flat.append((q1, q2))

    # Step 5: Angle-uniform transformation
    bounded_1_flat = angle_uniform_transform(indexed_points_1_flat)
    bounded_2_flat = angle_uniform_transform(indexed_points_2_flat)

    # Step 6: Density representation via KDE
    density_map = compute_kde_density(bounded_1_flat + bounded_2_flat, dynamic_bandwidth=True)
    # Density formula: rho(tau) = s(x) / |Vol(Dh)(x)| integrated over h^{-1}(tau) w.r.t d^{(n-2)}x
    final_density = apply_geometric_measure_theory(density_map)

    # Step 7: Rendering and interaction
    image = render_volume_with_occlusion(volume_data, final_density)
    widgets = create_parallel_coordinates_widgets(bounded_1_flat, bounded_2_flat)
    return image, widgets

# Standard math/vector operations defined for self-containment
def compute_gradient_magnitude(data): return standard_gradient(data)
def convolve_low_pass(data): return standard_convolution(data, gaussian_kernel)
def build_octree(saliency, threshold): return standard_octree_construction(saliency, threshold)
def local_pca(neighbors, data): return standard_pca(neighbors.values)
def project_to_2d_subspace(point, axis_i, axis_j): return standard_projection(point, [axis_i, axis_j])
def project_to_3d_subspace(point, axis_i, axis_j, axis_k): return standard_projection(point, [axis_i, axis_j, axis_k])
def cross_product(v1, v2): return standard_cross_product(v1, v2)
def angle_uniform_transform(points): return standard_angle_uniform_mapping(points)
def compute_kde_density(points, bandwidth): return standard_kde(points, bandwidth)
def apply_geometric_measure_theory(density): return standard_geometric_integration(density)
def render_volume_with_occlusion(vol, density): return standard_volume_rendering(vol, density, occlusion_shading=True)
def create_parallel_coordinates_widgets(p1, p2): return standard_widget_creation(p1, p2)
```

**Parameters:**
*   **Flat Dimensionality:** Supports 1-flat (two variables) and 2-flat (three variables) [S1].
*   **Spatial Neighborhood:** Size/radius for PCA neighborhood is not given in the sources [S1].
*   **Octree Subdivision Threshold ($t_s$):** A node is subdivided if the difference between the minimum and maximum values of $S_f$ in the region is greater than $t_s$ [S1].
*   **Variance Threshold ($t_e$):** If a leaf node's variance is smaller than $t_e$, the linear fit computed for the node is reused for every sample within it [S1].

**Edge Cases and Pitfalls:**
*   **False Positives:** Indexed points computed without spatial embedding result in a large number of false positives compared to the method with spatial embedding [S1].
*   **Computational Cost:** Computing local PCA for every volume sample is costly; the octree construction must consider every volume attribute because local PCA involves all $m$ dimensions of the multivariate volume [S1].

## Subtopics

### [Rendering Continuous Indexed Points](14769-rendering-continuous-indexed-points.md)
Rendering Continuous Indexed Points is a visualization technique that encodes local correlation of multivariate volume data by representing linear structures in parallel coordinates using continuous indexed points. It improves multivariate volume visualization by providing directional occlusion shading for spatial perception and allowing users to classify features based on density patterns. It differs from its sibling, [Multivariate Indexed Points Transfer Function](14772-multivariate-indexed-points-transfer-function.md), by focusing on the rendering of the indexed point layer itself within the parallel coordinate space rather than defining transfer functions for volume raycasting.

### [Multivariate Indexed Points Transfer Function](14772-multivariate-indexed-points-transfer-function.md)
This method defines the transfer function lookup mechanism for continuous indexed points, mapping multivariate volume data samples to color and opacity values via precomputed indexed point locations in image space. It solves the problem of efficiently applying multivariate transfer functions by avoiding costly on-the-fly local fitting transformations during rendering, instead relying on precomputed volumes that map spatial positions directly to indexed point coordinates. It differs from general multivariate transfer functions by utilizing a subspace-aware ID map to handle overlapping indexed points from different subspaces, distinguishing it from its sibling [Rendering Continuous Indexed Points](14769-rendering-continuous-indexed-points.md) which focuses on the rasterization of the points themselves rather than the transfer function logic.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** Not given in the sources [S1].
*   **Strengths:** Allows analysis of multivariate volume data using eigenvector information from local spatial embeddings; supports identification of correlations between two and three variables; facilitates good spatial perception via occlusion shading [S1].
*   **Limitations:** Parallel coordinates are difficult to interpret when there is a large number of attributes involved [S3, unofficial].
*   **When to use:** When analyzing local correlations in multi-attribute volume data and needing interactive exploration via parallel coordinates [S1].
*   **When not to use:** When the dataset has a very large number of attributes, making parallel coordinates hard to interpret [S3, unofficial].

## Background
The method was introduced by Liang Zhou, Xinyi Gou, and Daniel Weiskopf in December 2025 [S1]. It builds on earlier work in parallel coordinates interfaces for volume visualization, such as Pradhan, Bartz, and Mueller (2005), and extends it with dimensionality reduction techniques like LLE (Zhao and Kaufman, 2010) and MDS (Guo, Xiao, and Yuan, 2011) [S3, unofficial]. The technique is demonstrated through case studies, expert user studies, and domain expert feedback [S1].

## Sources
[S1] Continuous Indexed Points for Multivariate Volume Visualization, https://doi.org/10.26599/cvm.2025.9450496, fetched 2026-10-26
[S3] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
