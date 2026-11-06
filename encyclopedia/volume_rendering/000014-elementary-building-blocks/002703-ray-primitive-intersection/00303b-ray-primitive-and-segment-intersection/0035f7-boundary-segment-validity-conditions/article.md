# Boundary Segment Validity Conditions


## Overview
Boundary Segment Validity Conditions define the geometric constraints required for a line segment connecting two points on a surface mesh to be classified as a valid "boundary segment" in path-space differentiable rendering. These conditions ensure that the segment represents a direct, unobstructed path along the surface boundary, which is necessary for correctly computing boundary integrals and Jacobians during variable changes. They distinguish valid boundary interactions from penetrative or invalid geometric configurations by enforcing intersection consistency and non-penetration.

## How it works
The method validates a candidate boundary segment defined by endpoints $\mathbf{x}_{0S}$ and $\mathbf{x}_{0D}$ on a mesh $M(\pi)$, mediated by a boundary point $\mathbf{x}_B$ and direction $\boldsymbol{\omega}_B$.

**Inputs:**
- Mesh geometry $M(\pi)$ (polygonal mesh).
- Candidate boundary point $\mathbf{x}_B$ on an edge of $M(\pi)$.
- Direction vector $\boldsymbol{\omega}_B$ at $\mathbf{x}_B$.

**Outputs:**
- Boolean validity flag.
- Intersection points $\mathbf{x}_{0S}$ and $\mathbf{x}_{0D}$ if valid.

**Data Structures:**
- Mesh $M(\pi)$: List of faces, each with vertex positions and normal vectors.
- Edge data: For $\mathbf{x}_B$, identify the shared edge and the two adjacent face normals $\mathbf{n}$ and $\mathbf{n}'$.

**Steps:**
1. **Ray Tracing Validation**: Perform ray tracing from $\mathbf{x}_B$ in direction $-\boldsymbol{\omega}_B$ to find intersection $\mathbf{x}_{0S}$ with $M(\pi)$. Perform ray tracing from $\mathbf{x}_B$ in direction $\boldsymbol{\omega}_B$ to find intersection $\mathbf{x}_{0D}$ with $M(\pi)$. If either ray fails to hit the mesh, the segment is invalid [S1, unofficial].
2. **Non-Penetration Check**: Verify that the segment $\mathbf{x}_{0S}\mathbf{x}_{0D}$ does not penetrate the mesh interior. For a point $\mathbf{x}_B$ on an edge shared by faces with normals $\mathbf{n}$ and $\mathbf{n}'$, the direction $\boldsymbol{\omega}_B$ must satisfy:
   $$ (\boldsymbol{\omega}_B \cdot \mathbf{n}) (\boldsymbol{\omega}_B \cdot \mathbf{n}') < 0 $$
   This condition ensures the segment exits one face and enters the adjacent face without crossing the mesh volume [S1, unofficial].
3. **Light Source Constraint (Sampling Context)**: In practice, when sampling $\boldsymbol{\omega}_B$, restrict directions such that $\mathbf{x}_{0S}$ lies on a light source to separate direct and indirect boundary paths [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_valid_boundary_segment(mesh, x_B, omega_B):
    # Step 1: Ray trace to find endpoints
    # ray_trace(origin, direction, mesh) returns intersection point or None
    x_0S = ray_trace(x_B, -omega_B, mesh)
    x_0D = ray_trace(x_B, omega_B, mesh)
    
    if x_0S is None or x_0D is None:
        return False, None, None

    # Step 2: Check non-penetration via edge normals
    # Identify the edge containing x_B and its two adjacent face normals
    n, n_prime = get_edge_normals(mesh, x_B)
    
    # Condition: (omega_B . n) * (omega_B . n') < 0
    dot1 = dot(omega_B, n)
    dot2 = dot(omega_B, n_prime)
    
    if dot1 * dot2 >= 0:
        return False, None, None

    return True, x_0S, x_0D
```

## Subtopics

### Change of Integration Variable for Boundary Segments
This variant describes the change of variables from boundary segment endpoints $(\mathbf{p}_S, \mathbf{p}_D)$ to a surface point $\mathbf{p}_B$ and tangent direction $\boldsymbol{\omega}_B$. This transformation avoids silhouette detection for non-camera endpoints. The Jacobian of this change involves projecting $\mathbf{p}_S$ and $\mathbf{p}_D$ onto the silhouette $\Sigma_B$ of $\mathbf{p}_B$ as seen from $\mathbf{p}_S$. The variation of $\mathbf{p}_S$ along the projection of $\boldsymbol{\omega}_B$ onto the tangent plane maps to the variation of $\mathbf{p}_B$ along $\boldsymbol{\omega}_B$ [S2].

## Operating characteristics
- **Complexity**: Dominated by ray-triangle intersection tests for $\mathbf{x}_{0S}$ and $\mathbf{x}_{0D}$. Complexity is $O(N)$ for naive mesh traversal, or $O(\log N)$ with acceleration structures.
- **Strengths**: Enables correct computation of boundary integrals in differentiable rendering by ensuring geometric consistency of boundary paths.
- **Limitations**: Requires explicit mesh representation and robust ray tracing. The non-penetration condition is sensitive to numerical precision at edges.

## Background
Originated in the context of path-space differentiable rendering, specifically for handling boundary integrals on polygonal meshes. The validity conditions ensure that the change of variables in the rendering equation correctly accounts for surface boundaries without self-intersection or penetration artifacts.

## Sources
[S1] Path-Space Differentiable Rendering, https://core.ac.uk/download/541724175.pdf, fetched 2026-10-29 (unofficial)
[S2] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
