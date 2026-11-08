# Primary Edge Sampling

Also called: Primary Boundary Integral Estimation

## Overview
Primary Edge Sampling is a method for estimating boundary integrals in differentiable rendering by sampling points from the "primary edges" of an implicit surface. These edges are defined as the intersection of the implicit surface $\phi$ and an orientation function $\psi$, where the surface normal is perpendicular to the view vector [S1][S2, unofficial]. It solves the problem of sampling visibility discontinuities for implicit surfaces, which lack a finite set of explicit edges like meshes [S2, unofficial]. It differs from sibling methods like [Distance Field Foot Point Generation](6946-distance-field-foot-point-generation.md) by explicitly targeting the silhouette curve via root-finding on the orientation function rather than projecting foot points [S1][S2, unofficial].

## How it works
**Inputs:**
*   Implicit surface function $\phi(\mathbf{p}) = 0$.
*   Camera position $\mathbf{c}$.
*   A face $F$ of the axis-aligned bounding box (AABB) of $\phi$.
*   An edge $E$ of face $F$.
*   Maximum iteration count $i_{max}$.
*   Step size $\epsilon$ for marching along the edge.

**Outputs:**
*   A point $\mathbf{p}$ on the primary edge set $S$, or `NULL` if not found.

**Data Structures:**
*   **AABB:** Axis-aligned bounding box of the implicit surface. Used for range analysis to prune search space.
*   **Affine Arithmetic:** Used to bound the values of $\phi$ and $\psi$ over an AABB to determine if it intersects the zero-level sets [S1].

**Definitions:**
*   **View Vector:** $\mathbf{v}(\mathbf{p}) = \mathbf{c} - \mathbf{p}$.
*   **Orientation Function:** $\psi_{\phi, \mathbf{c}}(\mathbf{p}) := \langle \nabla \phi(\mathbf{p}), \mathbf{v}(\mathbf{p}) \rangle$. This function is zero where the surface normal is perpendicular to the view direction [S1][S2, unofficial].
*   **Primary Edge Set:** $S := \{ \mathbf{p} \mid \phi(\mathbf{p}) = 0, \psi_{\phi, \mathbf{c}}(\mathbf{p}) = 0 \}$. This is the intersection of the surface and the orientation function [S1][S2, unofficial].

**Parameters:**
*   **AABB Size:** $6.1 \times 10^{-5}$, which corresponds to cutting a unit cube 42 times in half [S1]. This size is used in range analysis to bound $S$ tighter [S1].

**Algorithm Steps:**
1.  **Range Analysis (Preprocessing):** Cut the local space of the implicit surface into AABBs of size $6.1 \times 10^{-5}$. Apply affine arithmetic to each AABB against $\phi$ and $\psi$. Discard AABBs guaranteed not to intersect both surfaces. Keep those that possibly intersect both [S1].
2.  **Initialization:** Sample a point $\mathbf{q}$ on edge $E$. Initialize $\psi_{prev} = 0$ [S1].
3.  **Marching Loop:** For $i$ from 1 to $i_{max}$:
    a.  Intersect a ray starting from $\mathbf{q}$ along the face normal $\mathbf{n}_F$ with the implicit surface $\phi$ to get point $\mathbf{p}$ [S1][S2, unofficial].
    b.  If no hit is found, reset $\psi_{prev} = 0$ and continue to the next step (advance $\mathbf{q}$) [S1].
    c.  Evaluate $\psi_i = \psi(\mathbf{p})$ [S1].
    d.  If $\psi_{prev} \cdot \psi_i < 0$, a sign change indicates the ray crossed the orientation function $\psi$. Since $\mathbf{p}$ is on $\phi$, $\mathbf{p}$ is on $S$. Return $\mathbf{p}$ [S1].
    e.  Set $\psi_{prev} = \psi_i$ [S1].
    f.  Advance $\mathbf{q}$ along the edge normal $\mathbf{n}_E$ by step size $\epsilon$: $\mathbf{q} \leftarrow \mathbf{q} + \epsilon \cdot \mathbf{n}_E$ [S1].
4.  **Return:** If the loop completes without finding a point, return `NULL` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_primary_edge(face_normal, edge_normal, edge_start, edge_length, max_steps, step_size, phi, psi, camera_pos):
    """
    Samples a point from the primary edge S.
    face_normal: Normal vector of the proxy face F.
    edge_normal: Normal vector of the edge E within face F.
    edge_start: Starting point q on edge E.
    edge_length: Length of edge E (used to bound marching).
    max_steps: Maximum number of marching steps i_max.
    step_size: Step size epsilon.
    phi: Function returning phi(p).
    psi: Function returning psi(p) = dot(grad_phi(p), camera_pos - p).
    camera_pos: Camera position c.
    """
    # Helper to compute gradient of phi numerically or analytically
    def grad_phi(p):
        # [G] Placeholder for gradient computation. In practice, use analytical gradient or finite differences.
        pass

    # Helper to intersect ray with implicit surface phi
    def ray_intersect_phi(origin, direction, max_dist=1.0):
        # [G] Placeholder for ray marching/intersection with phi.
        # Returns point p or None.
        pass

    # Initialize
    q = edge_start
    psi_prev = 0.0
    
    # Calculate number of steps based on edge length and step size
    # [G] The source uses i_max as a hard limit, but marching along an edge implies a bound.
    # We iterate up to max_steps.
    
    for i in range(1, max_steps + 1):
        # Intersect ray from q along face normal
        p = ray_intersect_phi(q, face_normal)
        
        if p is None:
            # No hit on phi, reset psi_prev
            psi_prev = 0.0
        else:
            # Evaluate psi at intersection p
            # psi(p) = dot(grad_phi(p), camera_pos - p)
            n = grad_phi(p)
            v = camera_pos - p
            psi_i = dot(n, v)
            
            # Check for sign change
            if psi_prev * psi_i < 0:
                # Sign change detected, p is on S
                return p
            
            psi_prev = psi_i
        
        # Advance q along edge normal
        q = q + step_size * edge_normal
        
        # Optional: Check if q has exceeded the edge bounds
        # [G] Source does not explicitly check edge bounds in the loop, 
        # relying on i_max and the geometry of the proxy face.
        
    return None
```

**Edge Cases and Pitfalls:**
*   **Multiple Intersections:** Rays may intersect the surface multiple times. The method assumes only one intersection in the tighter bound provided by range analysis [S1].
*   **No Intersection:** If the ray from $\mathbf{q}$ does not hit $\phi$, $\psi_{prev}$ is reset to 0 [S1].
*   **Range Analysis Dependency:** The efficiency relies on affine arithmetic to prune AABBs. If the AABB size is too large, the bound is loose, and the assumption of a single intersection may fail [S1].

## Subtopics

### [Silhouette-based Edge Sampling for Directly Visible Surfaces](8426-silhouette-based-edge-sampling-for-directly-visible-surfaces.md)
A method to efficiently sample boundary integrals for directly visible surfaces by identifying and sampling only silhouette edges, reducing variance compared to uniform edge sampling [8426-silhouette-based-edge-sampling-for-directly-visible-surfaces.md].

### Secondary Edge Sampling via Proxy Surface
Sampling the secondary boundary integral by first sampling a surface point using a proxy surface (e.g., bounding box face) and then a tangent direction [S1].
*   **Definition:** Uses a proxy face $F$ of the bounding box. Casts a ray from $\mathbf{q} \in F$ along normal $\mathbf{n}_F$ toward the implicit surface $B$. The ray intersects $B$ at $T_1(\mathbf{q})$ and $T_2(\mathbf{q})$. The integral over $B$ is transformed to an integral over $F$ using Jacobians $J_{T_1}$ and $J_{T_2}$ [S1].
*   **Formula:** $\int_B h(\mathbf{p}) d\mathbf{p} = \int_F \sum_{i=1}^{2} h(T_i(\mathbf{q})) J_{T_i}(\mathbf{q}) d\mathbf{q}$ [S1].
*   **Pitfall:** There may be more than two intersections, and the number may vary per point. The method sums over all intersections $i$, and points with fewer than $i$ intersections contribute zero via $g_i(\mathbf{q}) = h(T_i(\mathbf{q})) J_{T_i}(\mathbf{q})$ if defined, else 0 [S1].

### Primary Edge Sampling for Implicit Surfaces
Handles implicit surfaces defined as zero-level sets $\phi(x, \pi) = 0$, which do not offer a finite set of edges to sample from, requiring additional steps to sample visibility boundaries [S2, unofficial].
*   **Definition:** Primary boundaries are viewed as the intersected curve $S$ of implicit surface $\phi$ and $\psi(p) = \langle \nabla \phi(p), (p_N - p) \rangle = 0$, describing positions where surface normal is perpendicular to viewing direction [S2, unofficial].
*   **Formula:** $h(p)dA(p) := \int_B F'(\omega, p) d\theta(\omega) dA(p)$ where $B$ is union of material-form surfaces, $S$ is set of tangent directions, $F'$ is integrand after reparameterization, $\theta$ is angle measure [S2, unofficial].
*   **Steps:** (1) Introduce AABB of $\phi$, (2) sample points on edge $E$ of AABB, (3) from sampled point $q$, cast ray in face normal direction to find intersection with $S$, (4) if no intersection, march $q$ perpendicular to edge and repeat until point in $S$ found or end reached [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of AABBs visited and the cost of ray intersection and gradient evaluation. Range analysis reduces the search space [S1].
*   **Memory Complexity:** Low, as it operates on a per-ray or per-edge basis without storing large structures, aside from the AABB tree if used for acceleration [S1].
*   **Strengths:** Enables differentiable rendering of implicit surfaces by accurately sampling boundary integrals [S1][S2, unofficial].
*   **Limitations:** Requires gradient computation for $\phi$ [S1]. Relies on range analysis for efficiency; poor bounds can lead to missed intersections or false positives [S1].

## Background
Originated in "Path-Space Differentiable Rendering of Implicit Surfaces" by Zhou et al. (SIGGRAPH 2024) [S1]. It extends previous work on differentiable rendering of meshes to implicit surfaces, where explicit edges are not available [S2, unofficial]. The method uses affine arithmetic for range analysis to bound the primary edges [S1].

## Sources
[S1] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
[S2] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
