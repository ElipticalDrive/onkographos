# Surface-Preserving Kernel Adaptation

Also called: Laplacian-Based Deformation Adaptation

## Overview
Surface-Preserving Kernel Adaptation is a method for updating 3D Gaussian kernels after a deformation field has been applied to their means, ensuring that the kernels maintain their original surface coverage on the deformed geometry. It solves the problem of "spike-like" artifacts that arise when naive local affine transformations are estimated from partial neighborhoods, which fail to represent the transformation of the entire occupancy region [S1]. It sets itself apart from prior works that use geometric proxies like bounding scales or splitting by operating in a proxy-free scenario, directly adapting the kernel parameters to preserve the underlying surface manifold [S1].

## How it works
**Inputs:**
*   Deformed Gaussian means $\{\phi_\Delta(p_i)\}_{i=1}^N$, where $\phi_\Delta$ is the deformation map.
*   Original Gaussian means $\{p_i\}_{i=1}^N$.
*   Original Gaussian axes $\{v_{i,1}, v_{i,2}\}$ representing the major and minor axes of the occupancy region $\Omega_i$ [S1].
*   Splat graph structure defining adjacency [S1].

**Outputs:**
*   Adapted Gaussian kernels $G_{\phi_\Delta, i}$ with updated parameters (position, scale, orientation) that preserve surface coverage [S1].

**Data Structures:**
*   **Splat Graph:** An implicit or explicit graph where nodes are Gaussian kernels and edges connect adjacent kernels based on spatial proximity or splat overlap. Adjacency is determined by graph distance [S1].
*   **Inscribed Triangle $T_i$:** A set of three vertices $\{t_{i,1}, t_{i,2}, t_{i,3}\}$ representing the occupancy region $\Omega_i$ of Gaussian $G_i$ [S1].

**Steps:**

1.  **Define Inscribed Triangle Proxy:**
    For each Gaussian kernel $G_i$ with mean $p_i$ and axes $v_{i,1}$ (major) and $v_{i,2}$ (minor), define the inscribed triangle of maximum area $T_i = \{t_{i,k}\}_{k=1}^3$ as a proxy for its occupancy region $\Omega_i$ [S1]. The vertices are computed as:
    $$
    t_{i,1} = p_i + v_{i,1}
    $$
    $$
    t_{i,2} = p_i - \frac{1}{2}v_{i,1} + \frac{\sqrt{3}}{2}v_{i,2}
    $$
    $$
    t_{i,3} = p_i - \frac{1}{2}v_{i,1} - \frac{\sqrt{3}}{2}v_{i,2}
    $$
    [S1]

2.  **Identify Adjacent Gaussians:**
    For each triangle vertex $t_{i,k}$, identify a set of adjacent Gaussians $M(t_{i,k})$ from the splat graph. The selection is based on graph distance to ensure the adjacency of the surface [S1].

3.  **Transfer Displacement to Vertices:**
    The deformation map $\phi_\Delta$ is defined only on the Gaussian means. To find the deformed position of each triangle vertex $t_{i,k}$, compute the displacement using inverse distance weighting from the adjacent Gaussians in $M(t_{i,k})$ [S1]. The deformed vertex $t'_{i,k}$ is:
    $$
    t'_{i,k} = t_{i,k} + \frac{\sum_{j \in M(t_{i,k})} w_{ij} (\phi_\Delta(p_j) - p_j)}{\sum_{j \in M(t_{i,k})} w_{ij}}
    $$
    where the weight $w_{ij}$ is the inverse squared distance:
    $$
    w_{ij} = \|t_{i,k} - p_j\|^{-2}
    $$
    [S1]

4.  **Recover Adapted Kernel Parameters:**
    Compute the Steiner circumellipse of the deformed triangle $T'_i = \{t'_{i,1}, t'_{i,2}, t'_{i,3}\}$ [S1]. The parameters of the adapted Gaussian kernel $G_{\phi_\Delta, i}$ (position, scale, and orientation) are uniquely recovered from this ellipse to preserve the original surface coverage [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_inscribed_triangle(mean, major_axis, minor_axis):
    """
    Computes the vertices of the inscribed triangle of maximum area
    for a Gaussian kernel defined by its mean and axes.
    
    Args:
        mean: np.array, shape (3,), the Gaussian mean p_i.
        major_axis: np.array, shape (3,), the major axis v_i,1.
        minor_axis: np.array, shape (3,), the minor axis v_i,2.
        
    Returns:
        np.array, shape (3, 3), vertices t_i,1, t_i,2, t_i,3.
    """
    t1 = mean + major_axis
    t2 = mean - 0.5 * major_axis + (np.sqrt(3) / 2) * minor_axis
    t3 = mean - 0.5 * major_axis - (np.sqrt(3) / 2) * minor_axis
    return np.array([t1, t2, t3])

def inverse_distance_weighted_displacement(vertex, adjacent_means, deformed_means, graph_adjacency):
    """
    Computes the displaced position of a triangle vertex using inverse distance weighting
    from adjacent Gaussians.
    
    Args:
        vertex: np.array, shape (3,), the original triangle vertex t_i,k.
        adjacent_means: list of np.array, shape (3,), means p_j of adjacent Gaussians.
        deformed_means: list of np.array, shape (3,), deformed means phi_delta(p_j).
        graph_adjacency: list of bool/int, indicating which adjacent Gaussians are in M(t_i,k).
                         In practice, this filters the lists above to only include valid neighbors.
                         
    Returns:
        np.array, shape (3,), the deformed vertex position t'_i,k.
    """
    # Filter to only adjacent Gaussians based on graph distance
    valid_indices = [j for j, is_adj in enumerate(graph_adjacency) if is_adj]
    
    if not valid_indices:
        # Fallback if no neighbors found [G]
        return vertex
        
    neighbors_p = [adjacent_means[j] for j in valid_indices]
    neighbors_p_prime = [deformed_means[j] for j in valid_indices]
    
    displacements = [np.array(p_prime) - np.array(p) for p, p_prime in zip(neighbors_p, neighbors_p_prime)]
    
    weights = []
    for p in neighbors_p:
        dist_sq = np.sum((vertex - p) ** 2)
        # Avoid division by zero [G]
        if dist_sq < 1e-8:
            dist_sq = 1e-8
        weights.append(1.0 / dist_sq)
        
    weights = np.array(weights)
    weights_sum = np.sum(weights)
    
    if weights_sum == 0:
        return vertex
        
    weighted_displacement = np.zeros(3)
    for w, disp in zip(weights, displacements):
        weighted_displacement += w * disp
        
    return vertex + weighted_displacement / weights_sum

def steiner_circumellipse_parameters(vertices):
    """
    Recovers Gaussian kernel parameters (position, scale, orientation) from the
    vertices of the Steiner circumellipse.
    
    Note: The exact mathematical derivation for recovering Gaussian parameters
    from the Steiner circumellipse vertices is referenced as [47] in the source.
    This implementation assumes a standard geometric reconstruction.
    For a 2D Gaussian in 3D space, this typically involves finding the center,
    principal axes, and eigenvalues of the ellipse defined by the three points.
    
    Args:
        vertices: np.array, shape (3, 3), deformed triangle vertices t'_i,1, t'_i,2, t'_i,3.
        
    Returns:
        dict with keys 'mean', 'major_axis', 'minor_axis'.
    """
    # Compute centroid as the new mean [G]
    new_mean = np.mean(vertices, axis=0)
    
    # Compute covariance-like matrix to find axes [G]
    # Center the vertices
    centered = vertices - new_mean
    cov = np.dot(centered.T, centered)
    
    # Eigen decomposition to find orientation and scale
    eigenvalues, eigenvectors = np.linalg.eigh(cov)
    
    # Sort by eigenvalue magnitude
    idx = eigenvalues.argsort()[::-1]
    eigenvalues = eigenvalues[idx]
    eigenvectors = eigenvectors[:, idx]
    
    # Scale factors proportional to sqrt(eigenvalues)
    # Note: Exact scaling factor for Steiner circumellipse to Gaussian parameters
    # depends on the specific definition in [47]. Here we use a proportional scaling.
    scale1 = np.sqrt(eigenvalues[0])
    scale2 = np.sqrt(eigenvalues[1])
    
    # Ensure real positive scales
    scale1 = max(scale1, 1e-6)
    scale2 = max(scale2, 1e-6)
    
    return {
        'mean': new_mean,
        'major_axis': eigenvectors[:, 0] * scale1,
        'minor_axis': eigenvectors[:, 1] * scale2
    }

def surface_preserving_kernel_adaptation(gaussians, deformation_map, splat_graph):
    """
    Main adaptation routine.
    
    Args:
        gaussians: list of dicts with 'mean', 'major_axis', 'minor_axis'.
        deformation_map: function phi_delta that takes a mean and returns deformed mean.
        splat_graph: object or structure providing get_adjacent_gaussians(vertex, gaussians).
        
    Returns:
        list of adapted Gaussian parameters.
    """
    adapted_gaussians = []
    
    for i, g in enumerate(gaussians):
        p_i = g['mean']
        v1 = g['major_axis']
        v2 = g['minor_axis']
        
        # Step 1: Define inscribed triangle
        triangle = compute_inscribed_triangle(p_i, v1, v2)
        
        # Step 2 & 3: Transfer displacement to vertices
        deformed_triangle = np.zeros_like(triangle)
        for k in range(3):
            t_k = triangle[k]
            # Get adjacent Gaussians for this vertex
            # splat_graph.get_adjacent_gaussians should return list of indices or objects
            # For this pseudocode, we assume it returns a list of adjacent Gaussian indices
            adj_indices = splat_graph.get_adjacent_gaussians(t_k, gaussians)
            
            if not adj_indices:
                deformed_triangle[k] = t_k # No change if no neighbors [G]
                continue
                
            adj_means = [gaussians[j]['mean'] for j in adj_indices]
            adj_deformed_means = [deformation_map(gaussians[j]['mean']) for j in adj_indices]
            
            # Create adjacency mask (all returned are adjacent)
            adj_mask = [True] * len(adj_indices)
            
            deformed_triangle[k] = inverse_distance_weighted_displacement(
                t_k, adj_means, adj_deformed_means, adj_mask
            )
            
        # Step 4: Recover adapted kernel
        params = steiner_circumellipse_parameters(deformed_triangle)
        adapted_gaussians.append(params)
        
    return adapted_gaussians
```

**Parameters:**
*   $M(t_{i,k})$: Set of adjacent Gaussians to vertex $t_{i,k}$, chosen based on graph distance on the splat graph [S1].

**Edge Cases and Pitfalls:**
*   **Naive Affine Estimation Failure:** Estimating a local affine transformation from a neighborhood that only partially covers a subset of a Gaussian's occupancy region $\Omega$ causes the kernel to deviate from the underlying surface, leading to spike-like artifacts [S1].
*   **No Adjacent Neighbors:** If a vertex has no adjacent Gaussians in the splat graph, the displacement cannot be computed via weighting. A fallback (e.g., identity) is required [G].
*   **Precision:** Inverse distance weighting involves division by squared distances; small distances can lead to numerical instability if not clamped [G].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $N$ and the average degree of the splat graph. For each Gaussian, 3 vertices are processed, and each vertex queries adjacent Gaussians. If $K$ is the average number of adjacent Gaussians per vertex, complexity is $O(N \cdot K)$.
*   **Memory Complexity:** $O(N)$ to store the Gaussians and the splat graph structure.
*   **Strengths:** Preserves geometric structure and visual fidelity by maintaining original surface coverage; avoids spike-like artifacts common in naive deformation approaches; works in a proxy-free scenario [S1].
*   **Limitations:** Relies on the quality of the splat graph for adjacency; computationally more expensive than simple affine transformation if the graph is dense; requires the deformation map to be defined on means [S1].
*   **When to Use:** When deforming 3D Gaussian Splatting representations where preserving surface fidelity is critical and proxy-based methods (bounding scales, splitting) are not applicable or desired [S1].
*   **When Not to Use:** If computational efficiency is paramount and some artifact tolerance is acceptable; if the splat graph is poorly constructed or sparse [S1].

## Background
The method was introduced in "Proxy-Free Gaussian Splats Deformation with Splat-Based Surface Estimation" [S1]. It addresses limitations in existing deformation techniques for 3D Gaussian Splatting, particularly the failure of naive local affine transformations to preserve surface coverage. The approach builds on the foundation of 2D Gaussian Splatting, which encourages each kernel to act as a local patch covering the underlying surface [S1]. It references Steiner circumellipse [47] and inverse distance weighting [48] as mathematical foundations. The method was evaluated using Laplacian-based deformation techniques such as As-Rigid-As-Possible (ARAP) [1] and Bounded Biharmonic Weights (BBW) [2] on synthetic objects from ShapeNet, Objaverse, and Sketchfab [S1].

## Sources
[S1] Proxy-Free Gaussian Splats Deformation with Splat-Based Surface Estimation, https://doi.org/10.1109/3DV69130.2026.00155, fetched 2026-10-30
