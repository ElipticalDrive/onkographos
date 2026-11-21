# Tetrahedron Property Querying via Instant-NGP


## Overview
This method queries a differentiable radiance field parameterized by an Instant-NGP data structure and the mip-NeRF 360 contraction function to produce density and appearance features for individual tetrahedra [S1, unofficial]. It solves the problem of mapping point-based neural fields to volume-based primitives during optimization, where attaching features directly to vertices fails due to topological changes in the Delaunay tetrahedralization [S1, unofficial]. It differs from standard NeRF point querying by using tetrahedron centers (centroid or circumcenter) and incorporating a radius-based downweighting to handle geometric degeneracy [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Tetrahedron $k$ defined by its vertices, circumradius $\mathbb{R}_k$, viewing direction $d_k$, and a center strategy (`centroid` or `circumcenter`) [S1, unofficial].
*   **Output:** Density $\sigma_k$, base color $\mathbf{c}_k^0$, and color gradient $\nabla \mathbf{c}_k$ [S1, unofficial].

**Data Structures**
*   **Instant-NGP Field ($G_\theta$):** A multi-resolution hash grid parameterizing the radiance field, combined with the mip-NeRF 360 contraction function [S1, unofficial].
*   **Shallow Neural Network Heads:** Four separate heads ($h_\sigma, h_{sh}, h_\Delta, h_c$) that map the bottleneck feature vector $\mathbf{b}_k$ to output values [S1, unofficial].

**Parameters**
*   **Center Strategy:** Either `centroid` or `circumcenter` [S1, unofficial].
*   **Softplus Beta:** $\beta = 10$ for color computation [S1, unofficial].
*   **Downweighting Function ($\phi$):** A function of the radius $\mathbb{R}_k$ used to scale the NGP features element-wise; the exact mathematical form is not given in the sources [S1, unofficial].

**Steps**
1.  **Compute Center:** Determine the query point $\mathbb{O}_k$ [S1, unofficial].
    *   If `centroid`: $\mathbb{O}_k$ is the arithmetic mean of the tetrahedron's vertices [S1, unofficial].
    *   If `circumcenter`: $\mathbb{O}_k$ is the center of the circumscribed sphere [S1, unofficial].
2.  **Query NGP:** Retrieve the raw feature vector from the Instant-NGP field at $\mathbb{O}_k$: $\mathbf{f}_k \leftarrow G_\theta(\mathbb{O}_k)$ [S1, unofficial].
3.  **Downweight Features:** Apply the radius-based downweighting function $\phi(\mathbb{R}_k)$ to the features using element-wise multiplication: $\mathbf{b}_k \leftarrow \phi(\mathbb{R}_k) \circ \mathbf{f}_k$ [S1, unofficial]. Here, $\mathbf{b}_k$ is the bottleneck feature vector [S1, unofficial].
4.  **Compute Density:** Pass $\mathbf{b}_k$ through the density head $h_\sigma$ and apply an exponential activation: $\sigma_k \leftarrow \exp(h_\sigma(\mathbf{b}_k))$ [S1, unofficial].
5.  **Compute Base Color:** Pass $\mathbf{b}_k$ through the spherical harmonic head $h_{sh}$, compute the dot product with spherical harmonics basis functions $Y(d_k)$ evaluated in direction $d_k$, and apply softplus: $\mathbf{c}_k^0 \leftarrow \text{softplus}_{\beta=10}(h_{sh}(\mathbf{b}_k) \cdot Y(d_k))$ [S1, unofficial].
6.  **Compute Color Gradient:** Calculate the gradient to prevent negative colors within the tetrahedron [S1, unofficial].
    *   Compute the raw gradient vector via head $h_\Delta$: $\mathbf{g}_k \leftarrow h_\Delta(\mathbf{b}_k)$ [S1, unofficial].
    *   Normalize $\mathbf{g}_k$ using spherical normalization: $\hat{\mathbf{g}}_k \leftarrow \frac{\mathbf{g}_k}{\sqrt{1 + \|\mathbf{g}_k\|_2^2}}$ [S1, unofficial].
    *   Determine the minimum color value across RGB channels: $m_k \leftarrow \min_{\mathbf{c}_k} \mathbf{c}_k$ [S1, unofficial].
    *   Scale by the inverse radius and the minimum color: $\nabla \mathbf{c}_k \leftarrow \frac{m_k}{\mathbb{R}_k} \hat{\mathbf{g}}_k$ [S1, unofficial].
    *   The gradient $\nabla \mathbf{c}_k$ is monochrome (shared across all color channels) to save 30% of memory loads in shaders [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def centroid(vertices):
    """Arithmetic mean of tetrahedron vertices."""
    return [sum(v[i] for v in vertices) / len(vertices) for i in range(3)]

def circumcenter(vertices):
    """Center of the circumscribed sphere for a 3D tetrahedron."""
    A, B, C, D = vertices
    AB, AC, AD = [B[i]-A[i] for i in range(3)], [C[i]-A[i] for i in range(3)], [D[i]-A[i] for i in range(3)]
    def cross(u, v): return [u[1]*v[2]-u[2]*v[1], u[2]*v[0]-u[0]*v[2], u[0]*v[1]-u[1]*v[0]]
    def dot(u, v): return sum(x*y for x, y in zip(u, v))
    cross1 = cross(AB, AC)
    cross2 = cross(AC, AD)
    cross3 = cross(AD, AB)
    norm = 2.0 * dot(AB, cross3)
    if abs(norm) < 1e-9: return centroid(vertices) # Fallback for degenerate cases
    A_sq = dot(A, A)
    B_sq = dot(B, B)
    C_sq = dot(C, C)
    return [A[i] + (A_sq*cross3[i] - B_sq*cross2[i] + C_sq*cross1[i]) / norm for i in range(3)]

def softplus(x, beta=10.0):
    return (1.0 / beta) * math.log(1.0 + math.exp(beta * x))

def spherical_harmonics_basis(direction, degree=3):
    """Evaluates SH basis functions for a given direction. Exact implementation depends on degree [G]."""
    return [1.0] * ((degree + 1) ** 2)

def h_sigma(b): return b # Shallow network head [G]
def h_sh(b): return b   # Shallow network head [G]
def h_delta(b): return b # Shallow network head [G]
def h_c(b): return b     # Shallow network head [G]

def downweight_function(R_k):
    """Radius-based downweighting. Exact form not given in sources [G]."""
    return 1.0 / (1.0 + R_k)

def query_tetrahedron_properties(vertices, R_k, center_method, direction_d, ngp_query_fn):
    # Step 1: Compute Center O_k
    O_k = centroid(vertices) if center_method == 'centroid' else circumcenter(vertices)
    
    # Step 2: Query NGP
    f_k = ngp_query_fn(O_k)
    
    # Step 3: Downweight Features
    phi = downweight_function(R_k)
    b_k = [phi * f for f in f_k]
    
    # Step 4: Compute Density
    sigma = math.exp(h_sigma(b_k))
    
    # Step 5: Compute Base Color
    sh_features = h_sh(b_k)
    sh_basis = spherical_harmonics_basis(direction_d)
    color_raw = sum(a * b for a, b in zip(sh_features, sh_basis))
    c0 = softplus(color_raw, beta=10.0)
    
    # Step 6: Compute Color Gradient
    g_raw = h_delta(b_k)
    norm_factor = math.sqrt(1.0 + sum(x * x for x in g_raw))
    g_normalized = [x / norm_factor for x in g_raw]
    min_color = min(c0)
    grad_c = [(min_color / R_k) * x for x in g_normalized]
    
    return sigma, c0, grad_c
```

**Edge Cases and Pitfalls**
*   **Skinny Tetrahedra:** The circumcenter can move very quickly for skinny tetrahedra, causing numerical instability [S1, unofficial]. The circumradius $\mathbb{R}_k$ is used to prefilter the NGP to address this [S1, unofficial].
*   **Continuity vs. Stability:** The circumcenter provides continuity during edge-flipping (centers are co-located at the flip moment), but the centroid has better numerical conditioning [S1, unofficial]. The centroid is shown to have better performance in ablation studies [S1, unofficial].
*   **Negative Colors:** The activation in the color gradient formula prevents the color from going negative within the tetrahedron [S1, unofficial]. Since vertices lie within a ball of radius $R_k$, combining the gradient formula into the color integration yields a maximum negative change of $-\min \mathbf{c}_k$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Handles topological changes in the Delaunay tetrahedralization gracefully by querying a continuous field [S1, unofficial]. The monochrome gradient saves 30% of memory loads in shaders [S1, unofficial].
*   **Limitations:** The circumcenter strategy is numerically unstable for skinny tetrahedra [S1, unofficial]. The centroid strategy lacks continuity at edge-flips [S1, unofficial].
*   **When to use:** When optimizing a radiance field parameterized by a tetrahedral mesh where vertex-attached features are insufficient due to mesh topology changes [S1, unofficial].

## Background
This method is described in "Radiance Meshes for Volumetric Reconstruction" [S1, unofficial]. It builds on Instant-NGP and mip-NeRF 360 [S1, unofficial]. The approach to querying tetrahedra is similar to Zip-NeRF [S1, unofficial]. The method was developed to address the failure of vertex-attached features during optimization of Delaunay tetrahedralizations [S1, unofficial].

## Sources
[S1] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
