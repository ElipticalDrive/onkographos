# Gaussian Scale Regularization

Also called: Circumradius Scale Regularization, GaussianSplattingSLAM / Gaussian-SLAM, GaussianSplattingSLAM regularization, Isotropic Gaussian Regularization, Phys3DGS Scale Regularizer, Scale Constraint Loss

## Overview
Gaussian Scale Regularization is a constraint term used during the optimization of 3D Gaussian Splatting to prevent the formation of excessively large Gaussians, which can degrade rendering quality and stability [S1, unofficial]. It specifically addresses issues arising from large input triangles or the unconstrained training of Gaussian scales by forcing the maximum scale of a Gaussian to remain within a threshold of the circumradius of its associated mesh triangle [S1, unofficial]. Unlike general isotropic regularization which merely encourages equal scaling in all directions, this method ties the scale limit directly to the geometric resolution of the underlying mesh [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N_G$ Gaussians, where each Gaussian $i$ has a scale vector $s^i$ (three components representing scales along principal axes) and is associated with a specific mesh triangle. The mesh triangle is defined by vertices, from which edge vectors are derived.
*   **Output:** A scalar loss value $L_{sc}$ representing the regularization penalty.

**Data Structures:**
*   **Gaussian Scale ($s^i$):** A vector of three floating-point values. In standard 3DGS, these are often stored as logarithms of the scales for optimization stability, but the loss operates on the exponential (actual) scale values.
*   **Circumradius ($r^{i}_{sc}$):** A single floating-point value representing the radius of the circle passing through the three vertices of the triangle associated with Gaussian $i$.

**Formulas:**
The circumradius $r_c$ of a triangle is computed from two edge vectors $a$ and $b$ sharing a common vertex:
$$
r_{c} = \frac {\norm {a} \norm {b} \norm {a-b}}{2\norm {a\times b}}
$$
where $\norm{\cdot}$ denotes the Euclidean norm (L2 norm) and $\times$ denotes the cross product [S1, unofficial].

The regularization loss $L_{sc}$ is defined as:
$$
L_{sc} = \sum_{i \in N_G} \max(\max(s^i) - \kappa_{r_c} r^{i}_{sc}, 0)
$$
where:
*   $N_G$ is the total number of Gaussians.
*   $s^i$ is the scale vector for Gaussian $i$.
*   $\max(s^i)$ is the largest component of the scale vector $s^i$.
*   $r^{i}_{sc}$ is the circumradius of the triangle associated with Gaussian $i$.
*   $\kappa_{r_c}$ is a constant scale factor [S1, unofficial].

**Parameters:**
*   $\kappa_{r_c}$: The threshold constant. Empirically set to $0.2$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_scale_regularization(gaussians, mesh_triangles, kappa_rc=0.2):
    """
    Computes the Gaussian Scale Regularization loss.
    
    Args:
        gaussians: List or array of Gaussian objects/structures.
                   Each element i has a 'scales' attribute (array of 3 floats).
                   Each element i has a 'triangle_index' attribute mapping to mesh_triangles.
        mesh_triangles: List or array of triangle definitions.
                        Each element j is defined by 3 vertices (v0, v1, v2).
                        Each vertex is a 3D vector (x, y, z).
        kappa_rc: Float, the scale factor (default 0.2).
        
    Returns:
        loss: Float, the total regularization loss.
    """
    total_loss = 0.0
    
    for i in range(len(gaussians)):
        # Get the scale vector for the i-th Gaussian
        # Note: If scales are stored as logs, exponentiate them here.
        s_i = gaussians[i].scales
        
        # Find the maximum scale in the three directions
        max_scale = max(s_i[0], s_i[1], s_i[2])
        
        # Get the associated triangle
        tri_idx = gaussians[i].triangle_index
        v0, v1, v2 = mesh_triangles[tri_idx].vertices
        
        # Compute edge vectors sharing a vertex (e.g., v0)
        a = v1 - v0
        b = v2 - v0
        
        # Compute circumradius r_c
        # norm(a) is sqrt(a.x^2 + a.y^2 + a.z^2)
        norm_a = sqrt(dot(a, a))
        norm_b = sqrt(dot(b, b))
        norm_ab = sqrt(dot(a - b, a - b))
        
        # Cross product a x b
        cross_ab = cross(a, b)
        norm_cross_ab = sqrt(dot(cross_ab, cross_ab))
        
        # Avoid division by zero if triangle is degenerate
        if norm_cross_ab < 1e-8:
            r_c = 0.0
        else:
            r_c = (norm_a * norm_b * norm_ab) / (2.0 * norm_cross_ab)
            
        # Compute the penalty term
        # max(max(s^i) - kappa_rc * r_c, 0)
        penalty = max_scale - kappa_rc * r_c
        if penalty > 0:
            total_loss += penalty
            
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Degenerate Triangles:** If the triangle area is zero (collinear vertices), the cross product norm is zero, leading to division by zero in the circumradius formula. A guard clause (e.g., setting $r_c = 0$ or skipping the Gaussian) is required [G].
*   **Scale Representation:** If the Gaussian scales are optimized in log-space (common in 3DGS), they must be exponentiated before computing $\max(s^i)$ to compare against the physical circumradius [G].
*   **Triangle Association:** The method assumes a one-to-one or clear mapping between Gaussians and mesh triangles. In SLAM contexts where Gaussians may not be strictly tied to a single triangle, the association logic must be defined [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_G)$, where $N_G$ is the number of Gaussians. Each Gaussian requires a constant number of vector operations to compute the circumradius and the loss term.
*   **Memory Complexity:** $O(1)$ additional memory per Gaussian for the loss computation, assuming triangle data is pre-accessed.
*   **Strengths:** Effectively prevents "exploding" Gaussians caused by large input mesh features or unconstrained scale optimization. It stabilizes localization and mapping in SLAM applications by encouraging more isotropic and geometrically consistent Gaussian sizes [S2, unofficial].
*   **Limitations:** Ties the Gaussian scale strictly to the mesh resolution. If the mesh is coarse, the Gaussians are forced to be small, potentially losing fine detail. It does not directly enforce isotropy (equal scales in all directions) but rather limits the maximum scale; however, it is often used in conjunction with other terms to encourage isotropy [S2, unofficial].
*   **When to Use:** When using a mesh-based initialization or hybrid model (e.g., Phys3DGS) where large triangles could lead to poor normal estimation or rendering artifacts. Also useful in SLAM systems to stabilize scale drift [S1, unofficial][S2, unofficial].
*   **When Not to Use:** When the scene is represented purely by point clouds without an underlying mesh structure, as the circumradius cannot be computed.

## Background
The method was introduced in **Phys3DGS** (Physically-based 3D Gaussian Splatting for Inverse Rendering) by F. Author et al. in 2024 [S1, unofficial]. It was proposed to address two main issues: (1) large triangles in the input mesh causing large Gaussians, and (2) the unconstrained training of triangle-mapped Gaussian scales. The authors noted that while opacity regularization could reduce large triangles, it did not solve the scale training issue. The circumradius-based constraint was chosen because it provides a geometrically meaningful upper bound for the Gaussian size relative to the local mesh resolution [S1, unofficial]. The approach has since been adopted in various SLAM frameworks (e.g., GaussianSplattingSLAM, Gaussian-SLAM) to stabilize localization and mapping by encouraging isotropic Gaussians [S2, unofficial].

## Sources
[S1] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
