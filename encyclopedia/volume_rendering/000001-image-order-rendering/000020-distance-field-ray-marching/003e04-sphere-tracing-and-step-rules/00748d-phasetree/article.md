# PhaseTree

Also called: Multiphase Signed Distance Fields

## Overview
The PhaseTree is a hierarchical construction-tree representation for compactly modeling volumetric objects composed of multiple phases or materials across scales [S1]. It solves the limitation of conventional implicit surfaces, which are monolithic and encode only the signed distance to the nearest surface without differentiating between materials [S1]. Unlike standard implicit models textured with a solid texture $\phi(\mathbf{p})$ which do not provide means to compute the distance to the limit of the material, the PhaseTree explicitly models the geometry of the interface between different phases [S1]. It yields a unified multiphase signed distance representation that naturally supports complex topologies and interfaces [S1].

## How it works
The PhaseTree extends the standard Sphere Tracing algorithm to handle multiphase configurations [S1]. It defines the type of material at every point in space $\mathbf{p}$ by computing a 1-Lipschitz signed distance bound to the boundary of the phase, encoding both the signed distance to the closest interface between phases and the local phase at the query point [S1]. The multiphase signed distance function is denoted as $\Phi : \mathbb{R}^3 \rightarrow \mathbb{R}$ [S1].

**Inputs and Outputs:**
*   **Input:** A ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the normalized direction vector. A PhaseTree structure $\mathcal{T}$ representing the scene.
*   **Output:** The hit distance $t$, the hit point $\mathbf{p}$, and the identified phase ID at the surface.

**Data Structures:**
The PhaseTree is a hierarchical construction tree. Each node in the tree represents a phase-aware primitive or a composition operator. The leaves are phase-aware primitives, which may include standard SDF primitives, warping operators, or pre-existing 1-Lipschitz mono-phase neural fields [S1]. The tree structure allows for composition and warping operators to combine these primitives [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize accumulated distance $t = 0$ (or near-plane value) [G].
2.  **Marching Loop:**
    *   Evaluate the current position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.
    *   Query the PhaseTree $\mathcal{T}$ at $\mathbf{p}$ to obtain the multiphase signed distance value $d = \Phi(\mathbf{p})$ and the local phase ID [S1].
    *   **Termination Check:**
        *   If $d < \epsilon$ (convergence threshold), the ray has hit a phase interface. Return $t$, $\mathbf{p}$, and the phase ID [S1].
        *   If $t > t_{max}$ (maximum ray distance), the ray has escaped. Return failure [G].
    *   **Step:** Advance the ray by the distance value: $t \leftarrow t + d$ [S1]. This step size is guaranteed not to penetrate the surface because $\Phi$ is 1-Lipschitz [S1].
3.  **Shading:** Upon hit, compute surface properties (normal, color) based on the identified phase and lighting [S1].

**Formulas:**
The multiphase signed distance function $\Phi(\mathbf{p})$ provides a lower bound on the distance to the nearest surface of any phase [S1]. The function is constructed by combining phase-aware primitives through composition and warping operators [S1]. The specific composition operators (e.g., union, intersection) are not detailed in the provided sources, but the model ensures that the resulting $\Phi$ remains 1-Lipschitz [S1].

**Parameters:**
*   $\epsilon$: Convergence threshold. Typical value: $0.001$ [G].
*   $t_{max}$: Maximum ray distance. Scene-dependent [G].

**Edge Cases and Pitfalls:**
*   **Precision:** The 1-Lipschitz property is crucial for the correctness of the sphere tracing step. Any violation of this property may lead to penetration or missed intersections [S1].
*   **Phase Identification:** The algorithm must correctly identify the phase at the hit point. This is inherent to the PhaseTree structure [S1].
*   **Neural Fields:** When integrating pre-existing 1-Lipschitz mono-phase neural fields, ensure they strictly adhere to the 1-Lipschitz constraint to maintain the validity of the sphere tracing step [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phase_tree_tracing(ray_origin, ray_direction, phase_tree, max_steps, max_distance, epsilon):
    """
    Core PhaseTree sphere tracing algorithm.
    Args:
        ray_origin: Camera origin (vec3)
        ray_direction: Normalized ray direction (vec3)
        phase_tree: PhaseTree structure callable phase_tree.query(p) -> (distance, phase_id)
        max_steps: Maximum number of iterations
        max_distance: Maximum ray travel distance
        epsilon: Convergence threshold for surface hit
    Returns:
        hit_distance: Distance along ray to surface, or None
        hit_point: 3D position of hit, or None
        phase_id: ID of the phase at the hit point, or None
    """
    t = 0.0
    for _ in range(max_steps):
        if t > max_distance:
            break
        p = ray_origin + t * ray_direction
        d, phase_id = phase_tree.query(p)
        if d < epsilon:
            return t, p, phase_id
        t += d
    return None, None, None
```

## Operating characteristics
*   **Time Complexity:** The runtime overhead compared to single-phase Sphere Tracing is less than 25% [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Compact modeling of volumetric objects with multiple phases; explicit modeling of interfaces between phases; compatibility with accelerated Sphere Tracing schemes based on tree pruning or proxies and Sphere Carving bounding schemes [S1]; supports integration of pre-existing 1-Lipschitz mono-phase neural fields [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When modeling heterogeneous objects, composites, layered materials, or volumetric structures with distinct internal phases [S1].
*   **When not to use:** When only a single phase is required, as the overhead, though small, is unnecessary [S1].

## Background
The PhaseTree was introduced by Eric Galin, Pierre Hubert-Briere, Hugo Schott, Marie-Paule Cani, Adrien Peytavie, and Eric Guérin in 2026 in 'The PhaseTree: Multiphase Signed Distance Fields' published in ACM Transactions on Graphics, 45 (4) [S1]. It addresses the limitations of conventional implicit surfaces, which are monolithic and do not differentiate between materials or phases [S1]. The model is compatible with various implicit models such as BlobTrees, function representation, and neural models [S1]. It enables the modeling of geological structures, computer-aided design objects, and multi-phase organic shapes [S1].

## Sources
[S1] The PhaseTree: Multiphase Signed Distance Fields, https://doi.org/10.1145/3811379, fetched 2026-10-26
