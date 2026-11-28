# PlenOctree Fusion

Also called: Generalized PlenOctree Fusion

## Overview
PlenOctree Fusion is a coarse-to-fine method for constructing a completed PlenOctree representation by fusing local PlenOctrees derived from multiple target views, filtering redundant leaves, and refining leaf values using transmittance-based weighting rather than gradient descent [S1]. It solves the problem of incomplete or inaccurate PlenOctree initialization from sparse views by aggregating information from dense views to produce a high-quality, sparse octree suitable for real-time rendering [S1]. Unlike sibling methods that may rely on naive gradient descent for refinement, PlenOctree Fusion accelerates the process by avoiding a tremendous amount of optimization iterations [S1].

## How it works
**Inputs:**
- Multi-view images and silhouettes (extracted via chroma key segmentation and background subtraction) [S1].
- A neural network $\Psi$ capable of predicting images, densities, and colors given positions and directions [S1].
- Parameters: Number of sparse views $N_{\text{sparse}}=6$, number of dense views $N_{\text{dense}}=100$, transmittance threshold $T_{\text{thresh}}=10^{-3}$, density threshold $\sigma_{\text{thresh}}=10^{-3}$, and initial voxel grid size $N^3$ [S1].

**Outputs:**
- A completed PlenOctree with view-independent densities $\sigma$ and Spherical Harmonics (SH) coefficients $z$ at leaf nodes [S1].

**Data Structures:**
- **PlenOctree:** A sparse octree where each leaf node stores:
  - Density $\sigma$ (scalar) [S1].
  - SH coefficients $z$ (vector) [S1].
  - Update weight $W$ (scalar) [S1].
  - Update count $C$ (integer) [S1].
- **Visual Hull:** A binary mask or set of voxels defining the coarse geometry [S1].

**Steps:**

1. **Initialization:** Initialize a coarse PlenOctree with $N^3$ voxel grids as tree leaves [S1].
2. **Sparse View Prediction:** Given multi-view images and silhouettes, use network $\Psi$ to predict images for 6 sparse views uniformly distributed around the object [S1].
3. **Coarse Geometry Estimation:** Apply Shape-From-Silhouette (SFS) to generate a coarse visual hull from the silhouettes of the 6 sparse views [S1].
4. **Coarse PlenOctree Construction:** For each leaf node inside the visual hull:
   a. Uniformly sample directions $\theta, \phi \in [0, 2\pi]$ [S1].
   b. Feed positions and directions to $\Psi$ to predict view-dependent densities $\sigma(\theta, \phi)$ and colors $c(\theta, \phi)$ [S1].
   c. Convert view-dependent predictions to view-independent values using expectation $E(\cdot)$ and SH evaluation function $SH(\theta, \phi)$ [S1]:
      $$ \sigma = E(\sigma(\theta, \phi)) $$
      $$ z = E(SH(\theta, \phi) \cdot c(\theta, \phi)) $$
      where $E(\cdot)$ is the expectation operator and $SH(\theta, \phi)$ calculates SH from a direction [S1].
5. **Refinement Setup:** Render 100 dense view images using the coarse PlenOctree [S1]. Query the points where the transmittance in volume rendering $T_i > 10^{-3}$ from $\Psi$ [S1]. The 6 sparse views are used to initialize because they query all points, whereas the 100 dense views query only about 1% of points, making them faster for refinement [S1].
6. **Fusion and Refinement:** For each of the 100 dense viewpoints $i$:
   a. Query points from $\Psi$ where transmittance $T_i > 10^{-3}$ [S1].
   b. Update density $\sigma_i$ and SH coefficients $z$ at the $i$-th viewpoint using previous values $(\sigma_{i-1}, z_{i-1})$, current transmittance $T_i$, and previous weight $W_{i-1}$ [S1]:
      $$ \sigma_i = \frac{W_{i-1}\sigma_{i-1} + T_i\sigma_{\text{pred}}}{W_{i-1} + T_i} $$
      $$ z = \frac{W_{i-1}z_{i-1} + T_i \cdot SH(\theta_i, \phi_i) \cdot c(\theta_i, \phi_i)}{W_{i-1} + T_i} $$
   c. Calculate updated weight $W_i$ and update count $C_i$ [S1]:
      $$ W_i = \frac{C_i - 1}{C_i} W_{i-1} + \frac{T_i}{C_i} $$
      $$ C_i = C_{i-1} + 1 $$
      where $C_i$ represents the number of times the leaf has been updated [S1].
7. **Filtering:** Filter out leaves with density $\sigma < 10^{-3}$ to save storage and computational costs [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def plenoctree_fusion(multi_view_images, silhouettes, network_psi, N_sparse=6, N_dense=100, T_thresh=1e-3, sigma_thresh=1e-3, N_init=10):
    # Step 1: Initialize coarse PlenOctree with N^3 voxel grids as leaves
    plen_octree = []
    for x in range(N_init):
        for y in range(N_init):
            for z in range(N_init):
                plen_octree.append(Leaf(pos=(x, y, z), sigma=0.0, z=[], W=0.0, C=0))

    # Step 2: Predict sparse views
    sparse_silhouettes = []
    for i in range(N_sparse):
        img = network_psi.predict_image(multi_view_images, view_index=i)
        sil = extract_silhouette(img)
        sparse_silhouettes.append(sil)

    # Step 3: Generate visual hull
    visual_hull = shape_from_silhouette(sparse_silhouettes)

    # Step 4: Construct coarse PlenOctree
    for leaf in plen_octree:
        if leaf.pos in visual_hull:
            directions = uniform_sample_sphere(num_samples=32)
            sigma_samples = []
            z_samples = []
            for theta, phi in directions:
                sigma_pred = network_psi.predict_density(leaf.pos, theta, phi)
                c_pred = network_psi.predict_color(leaf.pos, theta, phi)
                sigma_samples.append(sigma_pred)
                sh_basis = evaluate_sh(theta, phi)
                z_samples.append(sh_basis * c_pred)
            leaf.sigma = mean(sigma_samples)
            leaf.z = mean(z_samples)
            leaf.W = 0.0
            leaf.C = 0

    # Step 5 & 6: Refine with dense views
    for i in range(N_dense):
        query_points = get_query_points(plen_octree, network_psi, view_index=i, T_thresh=T_thresh)
        for point in query_points:
            leaf = plen_octree.get_leaf(point.pos)
            theta_i, phi_i = get_view_direction(i)
            sigma_pred = network_psi.predict_density(point.pos, theta_i, phi_i)
            c_pred = network_psi.predict_color(point.pos, theta_i, phi_i)
            T_i = network_psi.get_transmittance(point.pos, theta_i, phi_i)

            new_sigma = (leaf.W * leaf.sigma + T_i * sigma_pred) / (leaf.W + T_i)
            sh_basis_i = evaluate_sh(theta_i, phi_i)
            new_z = (leaf.W * leaf.z + T_i * sh_basis_i * c_pred) / (leaf.W + T_i)

            new_C = leaf.C + 1
            new_W = ((new_C - 1) / new_C) * leaf.W + T_i / new_C

            leaf.sigma = new_sigma
            leaf.z = new_z
            leaf.W = new_W
            leaf.C = new_C

    # Step 7: Filter leaves
    plen_octree = [leaf for leaf in plen_octree if leaf.sigma >= sigma_thresh]
    return plen_octree

# Helper definitions
def extract_silhouette(image): return chroma_key_segment(image) [S1]
def shape_from_silhouette(silhouettes): return compute_visual_hull(silhouettes) [S1]
def uniform_sample_sphere(num_samples): return standard_uniform_sphere_sampling(num_samples) [G]
def mean(values): return sum(values) / len(values) [G]
def evaluate_sh(theta, phi): return spherical_harmonics_basis(theta, phi) [G]
def get_query_points(octree, psi, view_index, T_thresh): return ray_march_query(octree, psi, view_index, T_thresh) [G]
def get_view_direction(view_index): return uniform_view_direction(view_index) [G]
```

## Subtopics

### Topology-Aligned Octree Fusion
A two-stage fusion process (topology alignment and feature aggregation) to merge per-view octrees into a single global octree $V_{\text{fusion}}$ with a compatible hierarchy [S2, unofficial]. Topology alignment constructs the finest common refinement by iterating levels from $L_{\min}$ to $L_{\max}-1$, detecting regions where some octrees are coarser, and recursively subdividing voxels into eight children at $L+1$ when another octree has finer voxels with coinciding Morton prefixes [S2, unofficial]. Feature aggregation groups voxels by their Morton path and level, then averages zero-degree SH coefficients (colors) and higher-degree coefficients: $\bar{h}_{0p,\ell} = \frac{1}{|V_{p,\ell}|} \sum h_{0i,k}$ and $\bar{h}_{\text{sh}p,\ell} = \frac{1}{|V_{p,\ell}|} \sum h_{\text{sh}i,k}$ for $v_{i,k} \in V_{p,\ell}$ [S2, unofficial]. The fused octree is defined by unique cells $\{(p, \ell)\}$ and aggregated attributes $\{\bar{h}_{0p,\ell}, \bar{h}_{\text{sh}p,\ell}\}$ [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The method is significantly faster than naive gradient descent optimization [S1]. Generation time for a static scene PlenOctree is approximately 60 seconds [S1].
- **Memory Complexity:** Depends on the sparsity of the final PlenOctree after filtering [S1]. Filtering leaves with $\sigma < 10^{-3}$ reduces storage and computational costs [S1].
- **Strengths:** Accelerates refinement by avoiding a tremendous amount of optimization iterations; generates a completed PlenOctree quickly [S1].
- **Limitations:** Relies on the accuracy of the initial coarse PlenOctree and the network $\Psi$ for predictions; requires multi-view images and silhouettes [S1].
- **When to use:** When fast generation of a PlenOctree for real-time rendering is required, especially for static scenes [S1].

## Background
PlenOctree Fusion was introduced in the paper "Fourier PlenOctrees for Dynamic Radiance Field Rendering in Real-time" [S1]. It is part of the Fourier PlenOctree framework, which extends PlenOctree to dynamic scenes by compressing time-variant information in the frequency domain [S1]. The method was developed to address the slow generation speed of previous PlenOctree representations, improving it to 60 seconds for static scenes [S1].

## Sources
[S1] Fourier PlenOctrees for Dynamic Radiance Field Rendering in Real-time, https://doi.org/10.1109/cvpr52688.2022.01316, fetched 2026-10-26
[S2] Advancing Structured Priors for Sparse-Voxel Surface Reconstruction, https://doi.org/10.48550/arxiv.2601.17720, fetched 2026-10-26 (unofficial)
