# Primary Edge Sampling

Also called: Primary Boundary Integral Estimation

## Overview
Primary Edge Sampling generates sample points on the primary boundary curve $S$ of an implicit surface, which is defined as the intersection of the surface function $\phi$ and the orientation function $\psi$ [S1][S2, unofficial]. It solves the problem of sampling visibility discontinuities (silhouettes) for implicit surfaces, which lack a finite set of edges like polygonal meshes [S2, unofficial]. It sets itself apart from [Distance Field Foot Point Generation](6946-distance-field-foot-point-generation.md) by explicitly targeting the geometric condition where the surface normal is perpendicular to the view vector, rather than projecting points onto the surface [S1]. It differs from [Surface and Volume Training Position Sampling](9996-surface-and-volume-training-position-sampling.md) by focusing on boundary integrals for differentiable rendering rather than general training position generation [S1].

## How it works
**Inputs:**
*   Implicit surface function $\phi(\mathbf{p})$, where $\phi(\mathbf{p}) = 0$ defines the surface [S1][S2, unofficial].
*   Camera position $\mathbf{c}$ [S1].
*   Axis-Aligned Bounding Box (AABB) of the implicit surface [S1][S2, unofficial].
*   A face $F$ of the bounding box and an edge $E$ of that face [S1].
*   Maximum number of marching steps $i_{max}$ [S1].
*   Step size $\epsilon$ for marching along the edge [S1].

**Outputs:**
*   A point $\mathbf{p} \in S$ on the primary boundary curve [S1].

**Data Structures:**
*   **AABB:** Defined by minimum and maximum coordinates. Used for range analysis to bound the search space [S1].
*   **Orientation Function $\psi$:** Defined as $\psi_{\phi, \mathbf{c}}(\mathbf{p}) := \langle \nabla \phi(\mathbf{p}), \mathbf{v}_{\mathbf{c}}(\mathbf{p}) \rangle$, where $\mathbf{v}_{\mathbf{c}}(\mathbf{p}) = \mathbf{c} - \mathbf{p}$ is the view vector [S1]. The primary boundary $S$ is the set $\{ \mathbf{p} \mid \phi(\mathbf{p}) = 0, \psi_{\phi, \mathbf{c}}(\mathbf{p}) = 0 \}$ [S1].

**Parameters:**
*   **AABB Size:** In the source experiments, the AABB size is $6.1 \times 10^{-5}$, which corresponds to cutting a unit cube 42 times in half [S1]. This size is used in range analysis (affine arithmetic) to bound $S$ tighter [S1].
*   **Step Size $\epsilon$:** Not given in the sources.

**Algorithm Steps:**
1.  **Range Analysis (Preprocessing):** Cut the local space of the implicit surface into AABBs of size $6.1 \times 10^{-5}$ [S1]. Apply affine arithmetic to each AABB against $\phi$ and $\psi$ [S1]. Discard AABBs guaranteed not to intersect both surfaces; keep those that possibly intersect both [S1]. This bounds $S$ tighter and assumes only one intersection within the tighter bound [S1].
2.  **Proxy Selection:** Select a face $F$ of the bounding box and an edge $E$ of $F$ as a proxy surface [S1].
3.  **Initial Sample:** Sample a point $\mathbf{q}$ on edge $E$ [S1].
4.  **Initialize State:** Set $\psi_{prev} = 0$ [S1].
5.  **Marching Loop:** For $i$ from 1 to $i_{max}$:
    a. Intersect a ray starting from $\mathbf{q}$ along the face normal $\mathbf{n}_F$ with the implicit surface $\phi$ to get point $\mathbf{p}$ [S1].
    b. If no hit is found, reset $\psi_{prev} = 0$ [S1].
    c. Evaluate $\psi_i = \psi(\mathbf{p})$ [S1].
    d. If $\psi_{prev} \cdot \psi_i < 0$, return $\mathbf{p}$ as the sample point on $S$ [S1].
    e. Set $\psi_{prev} = \psi_i$ [S1].
    f. Advance $\mathbf{q}$ along the edge normal $\mathbf{n}_E$ by step size $\epsilon$ [S1].
6.  **Return:** If the loop completes without finding a sign change, return NULL [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_primary_edge(face_normal, edge_normal, max_steps, step_size, 
                        implicit_surface_phi, orientation_function_psi, 
                        ray_intersect):
    """
    Samples a point on the primary edge S.
    
    Args:
        face_normal: Normal vector of the proxy face F.
        edge_normal: Normal vector of the edge E within the face F.
        max_steps: Maximum number of marching steps i_max.
        step_size: Step size epsilon for advancing along the edge.
        implicit_surface_phi: Function phi(p) returning the implicit surface value.
        orientation_function_psi: Function psi(p) returning the orientation function value.
        ray_intersect: Function that returns the intersection point p of a ray 
                       (origin, direction) with the implicit surface phi, 
                       or None if no hit.
                       
    Returns:
        A point p on the primary edge S, or None if not found.
    """
    # 1. Sample a point q on the edge E of the face F
    # Note: The specific sampling distribution on E is not given in the sources [G].
    q = sample_point_on_edge() 
    
    # 2. Initialize psi_prev
    psi_prev = 0.0
    
    # 3. March along the edge
    for i in range(1, max_steps + 1):
        # Intersect ray from q along face normal n_F with implicit surface phi
        p = ray_intersect(q, face_normal)
        
        if p is None:
            # No hit, reset psi_prev
            psi_prev = 0.0
        else:
            # Evaluate psi at the current intersection point
            psi_i = orientation_function_psi(p)
            
            # Check for sign change
            if psi_prev * psi_i < 0:
                # Sign change indicates intersection with psi, 
                # and p is on phi, so p is on S
                return p
            
            # Update psi_prev
            psi_prev = psi_i
            
        # Advance q along the edge normal n_E
        q = q + step_size * edge_normal
        
    return None
```

**Edge Cases and Pitfalls:**
*   **Multiple Intersections:** Rays may intersect the surface multiple times. The method uses range analysis (affine arithmetic) to bound $S$ tighter and assumes only one intersection in this tighter bound [S1]. Without this, one would need to track all intersections and march to the other edge [S1].
*   **No Intersection:** If the ray from $\mathbf{q}$ does not hit the surface, $\psi_{prev}$ is reset to 0 [S1].
*   **End of Edge:** If the marching reaches the end of the edge without finding a sign change, the algorithm returns NULL [S1].

## Subtopics

### Secondary Edge Sampling via Proxy Surface
This method samples the secondary boundary integral by first sampling a surface point using a proxy surface (e.g., a bounding box face) and then a tangent direction [S1]. It sets itself apart from [Primary Edge Sampling](7168-primary-edge-sampling.md) by transforming the integral over the implicit surface $B$ to an integral over the proxy face $F$ using Jacobians $J_{T1}$ and $J_{T2}$ [S1]. It is chosen when secondary boundary integrals need to be evaluated for implicit surfaces [S1].
*   **Definition:** Sampling the secondary boundary integral by transforming the domain from the implicit surface $B$ to a proxy face $F$ [S1].
*   **Steps:**
    1.  Define a proxy face $F$ of the bounding box [S1].
    2.  Cast a ray from $\mathbf{q} \in F$ along normal $\mathbf{n}_F$ toward the implicit surface $B$ [S1].
    3.  The ray intersects $B$ at $T_1(\mathbf{q})$ and $T_2(\mathbf{q})$ [S1].
    4.  Transform the integral over $B$ to an integral over $F$ using the Jacobians $J_{T1}$ and $J_{T2}$ [S1].
*   **Pitfall:** In general, there may be more than two intersections, and the number of intersections may vary per point. The method sums over all intersections $i$, and points with fewer than $i$ intersections contribute zero [S1].

### Primary Edge Sampling for Implicit Surfaces
This variant handles implicit surfaces defined as zero-level sets $\phi(x, \pi) = 0$, which do not offer a finite set of edges to sample from, requiring additional steps to sample visibility boundaries [S2, unofficial]. It sets itself apart from [Primary Edge Sampling](7168-primary-edge-sampling.md) by explicitly detailing the use of AABBs and range analysis for implicit surfaces in the context of differentiable rendering surveys [S2, unofficial].
*   **Definition:** Sampling the primary boundary curve $S$ for implicit surfaces [S2, unofficial].
*   **Steps:**
    1.  Introduce an AABB of $\phi$ [S2, unofficial].
    2.  Sample points on edge $E$ of the AABB [S2, unofficial].
    3.  From a sampled point $q$, cast a ray in the face normal direction to find the intersection with $S$ [S2, unofficial].
    4.  If no intersection, march $q$ perpendicular to the edge and repeat until a point in $S$ is found or the end is reached [S2, unofficial].
*   **Formula:** Primary boundaries are viewed as the intersected curve $S$ of implicit surface $\phi$ and $\psi(p) = \langle \nabla \phi(p), (p_N - p) \rangle = 0$, describing positions where the surface normal is perpendicular to the viewing direction [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The complexity depends on the number of steps $i_{max}$ and the cost of ray intersection and function evaluation [S1].
*   **Memory Complexity:** Not given in the sources. The range analysis step requires storing AABBs and their bounds [S1].
*   **Measured Performance:** The AABB size of $6.1 \times 10^{-5}$ was used in experiments, cutting a unit cube 42 times in half [S1].
*   **Strengths:** Efficiently samples boundary integrals for directly visible surfaces by identifying and sampling only silhouette edges, reducing variance compared to uniform edge sampling [S1]. The use of range analysis bounds $S$ tighter, reducing the search space [S1].
*   **Limitations:** Assumes only one intersection in the tighter bound provided by range analysis [S1]. May fail to find points if the step size is too large or the edge does not intersect the primary boundary [S1].
*   **When to Use:** When differentiable rendering of implicit surfaces is required, specifically for sampling primary boundary integrals [S1][S2, unofficial].
*   **When Not to Use:** When the surface is explicitly meshed, as polygonal edges can be sampled directly [S2, unofficial].

## Background
*   **Origin:** Introduced by Zhou et al. in "Path-Space Differentiable Rendering of Implicit Surfaces" (SIGGRAPH 2024) [S1].
*   **History:** The method extends Path-Space Differentiable Rendering (PSDR) to implicit surfaces, which do not offer a finite set of edges like meshes [S2, unofficial]. It relies on range analysis (affine arithmetic) as described by Stolfi and De Figueiredo (1997) [S1].
*   **Usage:** Used in differentiable rendering of implicit surfaces to sample primary and secondary boundary integrals [S1][S2, unofficial].

## Sources
[S1] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
[S2] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
