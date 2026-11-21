# Probing for Surface Mesh Generation through Delaunay Refinement

Also called: Delaunay Refinement Surface Meshing, Probing Delaunay Refinement

## Overview
This method generates a surface mesh from an implicit surface without requiring an initial point set, using Delaunay refinement combined with a probing procedure to ensure all connected components of the surface are captured [S1, unofficial]. It guarantees topological equivalence to the surface and approximation properties defined by the refinement algorithm [S1, unofficial]. Unlike standard Delaunay refinement which requires seed points, this approach autonomously discovers isolated components through spatial probing of the domain [S1, unofficial]. It builds on [Delaunay Cover Refinement](13422-delaunay-cover-refinement.md) by adding a termination condition based on domain coverage rather than just local feature size satisfaction.

## How it works

### Inputs and Outputs
**Inputs:**
*   Implicit surface function $f(\mathbf{x}) = 0$ defining the geometry [S1, unofficial].
*   Bounding volume $\Omega$ enclosing the surface [S1, unofficial].
*   Parameter $\varepsilon$: The maximum distance from any point in the domain to the set of probes, defining the radius of the ball included in each connected component's volume [S1, unofficial].
*   Optional parameter $r$: A positive number such that any connected component contains at least one ball of radius $r$. If known, probes only need to be within distance $r$ of the surface, reducing search space [S1, unofficial].

**Outputs:**
*   A 3D triangulation (surface mesh) that is topologically equivalent to the surface [S1, unofficial].

### Data Structures
*   **Triangulation:** A Delaunay triangulation of the current set of probes (vertices) [S1, unofficial].
*   **Voronoi Diagram:** The dual of the triangulation, used to identify infinite edges and probe cells [S1, unofficial].
*   **Probes:** The set of points sampled from the domain $\Omega$ [S1, unofficial].

### Algorithm Steps

1.  **Initialization:** Start with an empty set of probes and an empty triangulation [S1, unofficial].
2.  **Probing Loop:** Continuously probe the domain $\Omega$ to find points. The probing strategy analyzes Voronoi cells of the current triangulation [S1, unofficial].
    *   If a probed segment intersects the surface (detected via sign change of $f$), add the intersection point to the triangulation [S1, unofficial].
    *   If a probed segment intersects the interior of a connected component but does not intersect the surface (indicating an isolated component not yet meshed), identify an "isolated point" [S1, unofficial].
3.  **Component Initialization:** When an isolated point is found:
    *   Initialize the component by adding the point to the triangulation [S1, unofficial].
    *   *Alternative Initialization:* Attempt to form a large triangle first, then refine and check if the component is meshed. If not, apply the standard isolated point initialization to prevent vertex clusters [S1, unofficial].
4.  **Refinement:** Apply the standard Delaunay refinement steps (e.g., inserting circumcenters of poor-quality simplices) to the triangulation [S1, unofficial]. This step meshes components that are close to each other or already partially initialized [S1, unofficial].
5.  **Termination Check:** The algorithm terminates when the domain is covered with probed Voronoi cells. Formally, every point in the domain is at distance at most $\varepsilon$ from the set of probes [S1, unofficial].
    *   *Optimization:* If parameter $r$ is known, the termination condition is relaxed: every point in a probed cell must only be at distance $r$ from the set of probes [S1, unofficial].

### Proof of Correctness (Proposition 4)
The algorithm guarantees that each connected component contains at least one point of the triangulation [S1, unofficial].
*   **Proof Sketch:** If the algorithm ends, the domain is covered with probed Voronoi cells (distance $\le \varepsilon$). Each connected component includes a ball of radius $\ge \varepsilon$. Thus, a probed segment intersects this ball (creating a probed point inside the volume). There also exists an infinite Voronoi edge providing a point outside the volume (clipped on the boundary of $\Omega$). Since the set of probes is connected and the surface is closed, a probed segment must intersect the surface. If this intersection point was not added to the triangulation, it would be at distance $< \varepsilon$ from a mesh face, and thus at most $2\varepsilon$ from a meshed component. Since components are at least $2\varepsilon$ apart, this confirms the component is meshed [S1, unofficial].

### Edge Cases and Pitfalls
*   **Accretion of Points:** Initializing components can cause accretion of points on spheres if points are not close enough to ensure the created facet is persistent, resulting in vertex clusters [S1, unofficial].
    *   *Mitigation:* A proposed remeshing step (delete or relocate vertices) can solve this, preserving topology and approximation criteria [S1, unofficial].
*   **Small/Spaced Components:** Very small and widely spaced components may not be intersected by Voronoi edges (no isolated points) and must be found by the probe procedure [S1, unofficial].
*   **Initialization Efficiency:** Initializing all isolated points at once is faster than one-by-one, as it initializes several components simultaneously, reducing refinement steps. However, it may initialize the same component twice if multiple isolated points lie on it [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probing_delaunay_refinement(implicit_surface, bounding_volume, epsilon, r=None):
    """
    Generate surface mesh using probing and Delaunay refinement.
    
    Args:
        implicit_surface: Function f(x) returning signed distance or value.
        bounding_volume: Bounding box or volume Omega.
        epsilon: Max distance from any point in domain to probes.
        r: Optional radius of ball contained in each component.
        
    Returns:
        Triangulation mesh.
    """
    probes = []
    triangulation = EmptyTriangulation()
    
    # Determine termination threshold
    threshold = r if r is not None else epsilon
    
    while not is_domain_covered(probes, bounding_volume, threshold):
        # Analyze Voronoi cells to find probing locations
        probe_points = analyze_voronoi_cells(triangulation, bounding_volume)
        
        for p in probe_points:
            # Check if probe intersects surface
            if implicit_surface.intersects(p):
                intersection = implicit_surface.find_intersection(p)
                if not triangulation.contains(intersection):
                    triangulation.insert(intersection)
                    probes.append(intersection)
            else:
                # Check for isolated component (probed inside volume but no surface intersection)
                if is_isolated_component(p, triangulation, implicit_surface):
                    # Initialize component
                    initialize_component(p, triangulation, implicit_surface)
                    probes.append(p)
                    
        # Refine mesh to improve quality and mesh close components
        refine_delaunay(triangulation)
        
    return triangulation

def is_domain_covered(probes, volume, threshold):
    """
    Check if every point in volume is within threshold distance of probes.
    Implemented by checking if all Voronoi cells in the domain are 'probed'.
    """
    # In practice, this involves checking if the union of balls of radius threshold
    # around probes covers the volume.
    # For implementation from text: check if max distance from any sample point
    # in volume to nearest probe is <= threshold.
    pass

def is_isolated_component(probe_point, triangulation, implicit_surface):
    """
    Detect if a probe point lies inside a connected component that has no
    surface points in the current triangulation.
    """
    # Find the connected component of the probe in the Voronoi diagram
    # or check if the probe is surrounded by existing probes but no surface face exists nearby.
    # Specifically, if a probed segment intersects the interior ball of a component
    # but no surface intersection was found along the path to an infinite Voronoi edge.
    pass

def initialize_component(point, triangulation, implicit_surface):
    """
    Add a point to initialize a new connected component.
    May attempt to form a large triangle first to avoid clusters.
    """
    # Option 1: Standard isolated point initialization
    triangulation.insert(point)
    
    # Option 2: Attempt large triangle first (proposed)
    # triangle = attempt_form_large_triangle(point, implicit_surface)
    # if triangle is valid and meshes component:
    #     triangulation.insert_triangle(triangle)
    # else:
    #     triangulation.insert(point)
    pass
```

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** The cost is proportional to the volume of the bounding volume $\Omega$. In the worst case for a cube of size $a$, the algorithm probes $(a/\varepsilon)^3$ cubes of size $\varepsilon$, each in constant time [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of vertices in the triangulation and the number of probes [S1, unofficial].

### Measured Performance
*   Tested on 10 random spheres and an implicit surface defined by $x^2 + y^2 + z^2 + \sin(4x) + \sin(4y) + \sin(4z) = 0$ [S1, unofficial].
*   At least 80% of components are meshed on average in the partial implementation (isolated points only). In the best case, all components are meshed [S1, unofficial].
*   Results are less unpredictable than previous methods using random initialization [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** No initial point set required; guarantees topological equivalence; handles isolated components; adaptable if prior knowledge (like $r$ or number of components) is available [S1, unofficial].
*   **Limitations:** Accretion of points on spheres during initialization can create clusters; partial implementation may leave smallest components unmeshed; performance depends heavily on the choice of bounding volume $\Omega$ [S1, unofficial].

### When to Use
*   Use when the surface geometry is unknown and no seed points are available.
*   Use when the surface consists of multiple disconnected components, some of which may be small or isolated.
*   Do not use if the bounding volume is excessively large relative to the surface, as this increases probing cost significantly [S1, unofficial].

## Background

### Origin and History
The method was proposed in the paper "Probing for Surface Mesh Generation through Delaunay Refinement" [S1, unofficial]. It extends the work of Boissonnat, Guibas, and Oudon on learning smooth shapes by probing [S1, unofficial]. The algorithm addresses the limitation of standard Delaunay refinement which requires an initial point set, potentially missing isolated components [S1, unofficial].

### Usage
Implemented as a partial version including only the treatment of isolated points. The mesh is initialized and refined while isolated points exist. The implementation allows initializing one or all isolated points at a given step [S1, unofficial].

## Sources

[S1] Probing for Surface Mesh Generation through Delaunay Refinement, https://inria.hal.science/hal-00747344v1, fetched 2026-10-26 (unofficial)
