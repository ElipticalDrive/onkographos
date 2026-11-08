# DCCVT Projection of 0-crossing Voronoi vertices

Also called: DCCVT Voronoi Vertex Projection, Plane-fitted Voronoi Projection

## Overview
This method projects Voronoi vertices (defined as tetrahedron circumcenters) onto the Signed Distance Function (SDF) zero-level set to correct geometric errors caused by the circumcenter lying far from the tetrahedron [S1]. Unlike naive barycentric interpolation, which is unreliable for distant circumcenters, or explicit tetrahedron containment searches, which are computationally prohibitive, this approach fits a plane to the projected sites of the enclosing tetrahedron and projects the vertex onto that plane [S1]. It serves as a robust post-processing step for Dual Contouring vertices extracted from a Clipped Centroidal Voronoi Tessellation (DCCVT) [S1].

## How it works
**Inputs:**
- A set of tetrahedra $\mathcal{T}_j$ that cross the zero-level set.
- For each tetrahedron $T_j$, the four sites (vertices) $s_i$ (where $i \in \{1,2,3,4\}$).
- For each site $s_i$, the SDF value $\phi_i$ and the gradient $\nabla \phi_i$.
- The Voronoi vertex $v_j$, defined as the circumcenter of tetrahedron $T_j$.

**Outputs:**
- The projected Voronoi vertex $v'_j$ lying on the SDF zero-level set.

**Data Structures:**
- Sites: 3D vectors $\mathbf{s}_i$.
- SDF Values: Scalars $\phi_i$.
- Gradients: 3D vectors $\nabla \phi_i$.
- Covariance Matrix: $3 \times 3$ symmetric matrix.

**Steps:**

1. **Project Sites to Zero-Level Set:**
   For each of the four sites $s_i$ in tetrahedron $T_j$, compute the projected site $s'_i$ using a single Newton step based on the local SDF value and gradient [S1]:
   $$
   \mathbf{s}'_i = \mathbf{s}_i - \frac{\nabla \phi_i}{\|\nabla \phi_i\| + \epsilon} \phi_i \quad (3)
   $$
   Here, $\epsilon$ is a small constant to prevent division by zero [S1].

2. **Compute Centroid of Projected Sites:**
   Calculate the centroid $\bar{\mathbf{s}}'_j$ of the four projected sites [S1]:
   $$
   \bar{\mathbf{s}}'_j = \frac{1}{4} \sum_{i \in \mathcal{T}_j} \mathbf{s}'_i \quad (4)
   $$

3. **Fit Plane via Covariance Eigenvector:**
   Compute the centered coordinates $\Delta \mathbf{s}'_i$ for each projected site [S1]:
   $$
   \Delta \mathbf{s}'_i = \mathbf{s}'_i - \bar{\mathbf{s}}'_j
   $$
   Construct the $3 \times 3$ centered covariance matrix $C$ of the projected sites [S1]:
   $$
   C = \frac{1}{4} \sum_{i \in \mathcal{T}_j} \Delta \mathbf{s}'_i \Delta \mathbf{s}'_i^\top
   $$
   Find the normal vector $\mathbf{n}_j$ of the fitted plane. This vector corresponds to the eigenvector associated with the smallest eigenvalue of $C$ [S1]. This minimizes the average squared distance from the projected sites to the plane passing through the centroid [S1]:
   $$
   \mathbf{n}_j = \arg \min_{\mathbf{n}} \frac{1}{4} \sum_{i \in \mathcal{T}_j} \left( \frac{\mathbf{n}}{\|\mathbf{n}\|} \cdot \Delta \mathbf{s}'_i \right)^2 \quad (5)
   $$
   Note: The source identifies $\mathbf{n}_j$ as the "first eigenvector" in the context of minimizing distance, which mathematically corresponds to the eigenvector of the smallest eigenvalue [S1].

4. **Project Voronoi Vertex:**
   Project the original Voronoi vertex $v_j$ onto the fitted plane defined by normal $\mathbf{n}_j$ and centroid $\mathbf{c}_j = \bar{\mathbf{s}}'_j$ [S1]:
   $$
   \mathbf{v}'_j = \mathbf{v}_j - [(\mathbf{v}_j - \mathbf{c}_j) \cdot \mathbf{n}_j] \mathbf{n}_j \quad (6)
   $$
   The result $\mathbf{v}'_j$ is the final projected vertex.

**Parameters:**
- $\epsilon$: Small constant for numerical stability in site projection. Value not given in sources [G]. Typical value: $10^{-6}$ or $10^{-8}$.

**Edge Cases and Pitfalls:**
- **Distant Circumcenters:** The circumcenter of a tetrahedron can lie far from the tetrahedron itself, making straightforward barycentric interpolation of SDF values and gradients unreliable [S1]. This method avoids that issue by using local site projections [S1].
- **Containment Search:** Explicitly searching for the containing tetrahedron for a given Voronoi vertex is computationally prohibitive [S1]. This method assumes the Voronoi vertex is associated with the tetrahedron formed by its four defining sites, avoiding the search [S1].
- **Degenerate Tetrahedra:** If the four sites are coplanar, the covariance matrix will be singular. The smallest eigenvalue will be zero, and the corresponding eigenvector will define the plane normal. This is handled naturally by the eigendecomposition [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def project_voronoi_vertex(sites, sdfs, gradients, voronoi_vertex, epsilon=1e-8):
    """
    Projects a Voronoi vertex onto the SDF zero-level set using the DCCVT method.
    
    Args:
        sites: Array of shape (4, 3), the four sites s_i of the tetrahedron.
        sdfs: Array of shape (4,), the SDF values phi_i at each site.
        gradients: Array of shape (4, 3), the gradients nabla phi_i at each site.
        voronoi_vertex: Array of shape (3,), the circumcenter v_j of the tetrahedron.
        epsilon: Small constant for numerical stability.
        
    Returns:
        projected_vertex: Array of shape (3,), the projected Voronoi vertex v'_j.
    """
    # Step 1: Project sites to zero-level set
    # s'_i = s_i - (nabla phi_i / (||nabla phi_i|| + epsilon)) * phi_i
    norms = np.linalg.norm(gradients, axis=1, keepdims=True)
    # Avoid division by zero if gradient is zero
    norms = np.maximum(norms, epsilon)
    projected_sites = sites - (gradients / norms) * sdfs[:, np.newaxis]
    
    # Step 2: Compute centroid of projected sites
    centroid = np.mean(projected_sites, axis=0)
    
    # Step 3: Fit plane via covariance eigenvector
    # Center the projected sites
    centered_sites = projected_sites - centroid
    
    # Compute covariance matrix C = (1/4) * sum(delta_s'_i * delta_s'_i^T)
    # Using matrix multiplication for efficiency: C = (1/4) * centered_sites.T @ centered_sites
    covariance_matrix = (centered_sites.T @ centered_sites) / 4.0
    
    # Find the eigenvector corresponding to the smallest eigenvalue
    # np.linalg.eigh returns eigenvalues in ascending order
    eigenvalues, eigenvectors = np.linalg.eigh(covariance_matrix)
    normal = eigenvectors[:, 0]  # First eigenvector (smallest eigenvalue)
    
    # Normalize normal vector (should already be normalized by eigh, but for safety)
    normal = normal / np.linalg.norm(normal)
    
    # Step 4: Project Voronoi vertex onto the fitted plane
    # v'_j = v_j - ((v_j - c_j) . n_j) * n_j
    dot_product = np.dot(voronoi_vertex - centroid, normal)
    projected_vertex = voronoi_vertex - dot_product * normal
    
    return projected_vertex
```

## Subtopics

### DCCVT Projection of 0-crossing edge midpoints
This variant projects the midpoints of tetrahedron edges that cross the zero-level set onto the SDF zero-level set to provide additional constraints ensuring Voronoi faces remain well aligned with the SDF and target point cloud [S1]. For each edge between adjacent sites $s_i$ and $s_j$ crossing the zero-level set, the SDF value $\phi(b_{ij})$ and gradient $\nabla \phi(b_{ij})$ at the midpoint $b_{ij}$ are approximated by linear interpolation: $\phi(b_{ij}) \approx \frac{1}{2}(\phi_i + \phi_j)$ and $\nabla \phi(b_{ij}) \approx \frac{1}{2}(\nabla \phi_i + \nabla \phi_j)$ [S1]. The midpoint is then projected onto the zero-level set using a Newton step: $b'_{ij} = b_{ij} - \frac{\nabla \phi(b_{ij})}{\|\nabla \phi(b_{ij})\| + \epsilon} \phi(b_{ij})$ [S1]. This method is chosen when higher fidelity to the SDF is required for mesh faces, not just vertices [S1].

## Operating characteristics
- **Time Complexity:** $O(1)$ per Voronoi vertex, as it involves a fixed number of arithmetic operations and a $3 \times 3$ eigendecomposition.
- **Memory Complexity:** $O(1)$ auxiliary memory per vertex.
- **Strengths:** Robust against degenerate tetrahedra where circumcenters are far from the mesh; avoids expensive tetrahedron containment searches; uses only local site information [S1].
- **Limitations:** Relies on the accuracy of the SDF gradient at the sites; assumes the tetrahedron is well-defined [S1].
- **When to use:** When extracting meshes from DCCVT where vertex accuracy on the zero-level set is critical and standard barycentric interpolation fails due to large circumcenter distances [S1].

## Background
The DCCVT (Differentiable Clipped Centroidal Voronoi Tessellation) method was introduced to address challenges in meshing unoriented 3D point clouds using SDFs [S1]. The projection of 0-crossing Voronoi vertices was proposed as a specific technique within this framework to ensure that the extracted mesh vertices lie accurately on the zero-level set of the inferred SDF, overcoming the limitations of naive interpolation methods [S1].

## Sources
[S1] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
