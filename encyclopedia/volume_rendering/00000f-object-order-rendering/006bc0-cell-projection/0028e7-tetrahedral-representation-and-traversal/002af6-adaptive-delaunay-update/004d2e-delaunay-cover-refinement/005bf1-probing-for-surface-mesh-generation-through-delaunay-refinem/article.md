# Probing for Surface Mesh Generation through Delaunay Refinement

Also called: Boissonnat and Oudot method, Chew's algorithm, Delaunay Refinement Meshing Algorithm, Delaunay Refinement Surface Meshing, Delaunay Refinement for Surface Meshing, Probing Delaunay Refinement

## Overview
This method generates a surface mesh from an implicit surface without requiring an initial point set, interleaving shape probing (intersection with Voronoi edges) with Delaunay refinement [S1, unofficial]. It guarantees topological equivalence to the input surface and approximation properties defined by the refinement algorithm [S1, unofficial]. Unlike standard Delaunay refinement which requires seed points, this approach autonomously discovers isolated components through spatial probing of the domain [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Implicit surface oracle $f(\mathbf{x}) = 0$ defining the geometry [S1, unofficial].
*   Convex compact bounding volume $\Omega$ enclosing the surface [S1, unofficial].
*   Approximation parameter $\varepsilon$: strictly less than $0.091 \times \text{reach}(S)$, defining the maximum distance from any point in the domain to the set of probes [S1, unofficial].
*   Size criterion: strictly less than $0.019\varepsilon$ [S1, unofficial].
*   Optional parameter $r$: a positive number such that any connected component contains at least one ball of radius $r$ [S1, unofficial].

**Outputs:**
*   A 3D triangulation (surface mesh) that is a 2-manifold without boundary, isotopic to $S$, with bounded Hausdorff distance and well-shaped triangles [S1, unofficial].

### Data Structures
*   **Priority Queue $Q$:** Stores unprobed Voronoi cells sorted by decreasing size [S1, unofficial].
*   **Delaunay Triangulation $T$:** Maintains the current set of probes and their dual Voronoi diagram [S1, unofficial].
*   **Probes:** Set of sampled points in $\Omega$ [S1, unofficial].

### Algorithm Steps
1.  **Initialization:** Create empty priority queue $Q$. Probe $\Omega$ to find an initial intersection point $P_0$. Initialize the component of $P_0$ by adding two nearby points to form a persistent facet [S1, unofficial].
2.  **Main Loop:** While $Q$ is not empty, check if an isolated point $P$ exists in $T$. If yes, call `initialize_component(P)` [S1, unofficial]. If no, pop cell $C$ from $Q$, probe $C$ until an intersection is found or $Q$ is empty [S1, unofficial].
3.  **Probing (`probe(C)`):** Include cell $C$ in a minimal bounding cuboid. Probe along the lengthiest direction of the cuboid, clipped on faces. Order probes to maximize intersection probability by starting at the central axis [S1, unofficial]. Add additional probes to connect segments to the Voronoi diagram [S1, unofficial]. Ignore intersections closer than $\varepsilon$ to existing mesh [S1, unofficial].
4.  **Update:** If an intersection is found, insert the point into $T$, add its dual cell to $Q$, and update adjacent cells in $Q$ by decreasing their size [S1, unofficial]. The refinement step must also update $Q$ to keep it current [S1, unofficial].
5.  **Termination:** The algorithm terminates when the domain is covered with probed Voronoi cells, meaning every point in the domain is at distance at most $\varepsilon$ of the set of probes [S1, unofficial]. If $r$ is known, the termination threshold relaxes to $r$ [S1, unofficial].

### Formulas
*   Hausdorff distance: $\le 4.5 \cdot \text{diam}(S) \cdot \varepsilon^2 / \text{reach}(S)$ [S1, unofficial].
*   Area and normals approximation error: $O(\varepsilon)$ [S1, unofficial].
*   Persistent facet condition: surface Delaunay ball radius $r < 0.0455$ (half of the size criterion $0.091$) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Accretion of Points:** Initializing components can cause accretion of points on spheres if points are not close enough to ensure the created facet is persistent, resulting in vertex clusters [S1, unofficial]. A proposed remeshing step (delete or relocate vertices) can solve this while preserving topology and approximation criteria [S1, unofficial].
*   **Small/Spaced Components:** Very small and widely spaced components may not be intersected by Voronoi edges (no isolated points) and must be found by the probe procedure [S1, unofficial].
*   **Initialization Efficiency:** Initializing all isolated points at once is faster than one-by-one, as it initializes several components simultaneously, reducing refinement steps [S1, unofficial]. However, it may initialize the same component twice if multiple isolated points lie on it [S1, unofficial].
*   **Alternative Initialization:** Attempt to form a large triangle first, then refine and check if the component is meshed [S1, unofficial]. If not, apply the standard isolated point initialization to prevent vertex clusters [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probing_delaunay_refinement(implicit_surface, bounding_volume, epsilon, size_criterion, r=None):
    """
    Generate surface mesh using probing and Delaunay refinement.
    """
    triangulation = []  # List of vertices and facets
    voronoi_cells = []  # List of Voronoi cells
    Q = PriorityQueue() # Min-heap sorted by cell size (largest first)
    probes = []
    threshold = r if r is not None else epsilon

    # Step 1: Initialization
    P0 = probe_domain(implicit_surface, bounding_volume)
    triangulation.append(P0)
    probes.append(P0)
    initialize_component(P0, triangulation, implicit_surface, epsilon)

    # Step 2: Main Loop
    while not is_domain_covered(probes, bounding_volume, threshold):
        isolated_points = find_isolated_points(triangulation, implicit_surface, epsilon)
        if isolated_points:
            for P in isolated_points:
                initialize_component(P, triangulation, implicit_surface, epsilon)
                probes.append(P)
        else:
            while not Q.empty():
                C = Q.pop()
                intersections = probe_cell(C, implicit_surface, epsilon)
                for pt in intersections:
                    if not in_triangulation(pt, triangulation):
                        triangulation.insert(pt)
                        probes.append(pt)
                        update_queue(Q, C, pt)
                if intersections:
                    break
        # Step 4: Refine
        refine_delaunay(triangulation, size_criterion, Q)

    return triangulation

def probe_domain(implicit_surface, bounding_volume):
    """Find first intersection point in the domain."""
    # Probe along central axis of bounding_volume
    start = bounding_volume.center
    direction = bounding_volume.lengthiest_axis
    t = 0.0
    step = 0.1 * epsilon
    while t < bounding_volume.max_extent:
        p = start + t * direction
        if implicit_surface.sign_change(p, p + step * direction):
            return implicit_surface.find_intersection(p, p + step * direction)
        t += step
    return start  # Fallback if surface not found initially

def probe_cell(C, implicit_surface, epsilon):
    """Probe a Voronoi cell C for surface intersections."""
    cuboid = minimal_bounding_cuboid(C)
    intersections = []
    direction = cuboid.lengthiest_axis
    start = cuboid.center
    step = 0.1 * epsilon
    t = 0.0
    while t < cuboid.max_extent:
        p = start + t * direction
        if implicit_surface.sign_change(p, p + step * direction):
            pt = implicit_surface.find_intersection(p, p + step * direction)
            if distance_to_mesh(pt) >= epsilon:
                intersections.append(pt)
        t += step
    # Connect segments to Voronoi diagram
    connect_probes_to_voronoi(intersections, C)
    return intersections

def initialize_component(P, triangulation, implicit_surface, epsilon):
    """Initialize an isolated component by probing a small cube and medial square."""
    triangulation.append(P)
    # Find second point by probing edges of small cube (size < epsilon)
    cube_size = 0.9 * epsilon
    cube = centered_cube(P, cube_size)
    for edge in cube.edges:
        if implicit_surface.sign_change(edge.start, edge.end):
            P2 = implicit_surface.find_intersection(edge.start, edge.end)
            triangulation.append(P2)
            break
    # Find third point by probing edges of square in medial plane
    if len(triangulation) >= 2:
        mid = midpoint(P, P2)
        square = centered_square(mid, cube_size, medial_plane(P, P2))
        for edge in square.edges:
            if implicit_surface.sign_change(edge.start, edge.end):
                P3 = implicit_surface.find_intersection(edge.start, edge.end)
                triangulation.append(P3)
                break

def find_isolated_points(triangulation, implicit_surface, epsilon):
    """Identify components with <= 2 points that are isolated."""
    isolated = []
    for comp in connected_components(triangulation):
        if len(comp.vertices) <= 2:
            diameter = max_distance(comp.vertices)
            volume = convex_hull_volume(comp.vertices)
            if volume_contains_no_other_component(volume, triangulation) and \
               min_distance_to_other_components(comp, triangulation) >= diameter:
                isolated.append(comp.vertices[0])
    return isolated

def is_domain_covered(probes, bounding_volume, threshold):
    """Check if every point in bounding_volume is within threshold of probes."""
    # Approximate by sampling a grid or checking max distance in unprobed cells
    max_dist = 0.0
    for pt in bounding_volume.sample_points(1000):
        d = min_distance_to_set(pt, probes)
        if d > max_dist:
            max_dist = d
    return max_dist <= threshold

def refine_delaunay(triangulation, size_criterion, Q):
    """Insert intersections of surface with Voronoi edges dual to bad facets."""
    bad_facets = find_bad_facets(triangulation, size_criterion)
    for facet in bad_facets:
        voronoi_edge = dual_voronoi_edge(facet)
        if voronoi_edge.intersects_surface():
            pt = voronoi_edge.intersection_point()
            if not in_triangulation(pt, triangulation):
                triangulation.insert(pt)
                update_queue(Q, facet.cell, pt)

def update_queue(Q, old_cell, new_point):
    """Decrease size of adjacent cells in Q."""
    for adj_cell in Q.cells_adjacent_to(old_cell):
        adj_cell.size = volume(adj_cell) - volume_cone(adj_cell, new_point)
        Q.decrease_key(adj_cell)
```

## Subtopics

### Automatic Seeding for Delaunay Refinement
An automatic procedure to seed a surface mesh generator for compact smooth closed 2-manifold surfaces with arbitrary genus and number of connected components, given an intersection oracle, a convex compact domain $\Omega$, and a lower bound $\varepsilon$ of the reach [S1, unofficial]. The procedure interleaves refinement, seeding, and initialization steps, using the Voronoi diagram and additional probes without storing the latter, requiring only a subset of Voronoi cells (or triangulation vertices) in a priority queue [S1, unofficial]. Unlike previous methods restricted to non-nested components and probing from outside, this method allows nested components and probing from any direction, without requiring an initial point set or prior knowledge of the number of connected components [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** The cost lies in the analysis of Voronoi cells and is at most proportional to the volume of the bounding volume $\Omega$ [S1, unofficial]. For a cube of size $a$, the worst case involves $(a/\varepsilon)^3$ cubes of size $\varepsilon$, all probed in constant time [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of vertices in the triangulation and the number of probes stored in the priority queue [S1, unofficial].

### Measured Performance
*   Tested on 10 random spheres and an implicit surface defined by $x^2 + y^2 + z^2 + \sin(4x) + \sin(4y) + \sin(4z) = 0$ [S1, unofficial].
*   At least 80% of components are meshed on average in the partial implementation (isolated points only) [S1, unofficial]. In the best case, all components are meshed [S1, unofficial].
*   Results are less unpredictable than previous methods using random initialization [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** No initial point set required; guarantees topological equivalence; handles isolated components; adaptable if prior knowledge (like $r$ or number of components) is available [S1, unofficial].
*   **Limitations:** Accretion of points on spheres during initialization can create clusters; partial implementation may leave smallest components unmeshed; performance depends heavily on the choice of bounding volume $\Omega$ [S1, unofficial].

### When to Use
*   Use when the surface geometry is unknown and no seed points are available [S1, unofficial].
*   Use when the surface consists of multiple disconnected components, some of which may be small or isolated [S1, unofficial].

## Background

### Origin and History
The method was proposed by Hugo Férée and Pierre Alliez in 2009 [S1, unofficial]. It extends the work of Boissonnat, Guibas, and Oudon on learning smooth shapes by probing [S1, unofficial]. The algorithm addresses the limitation of standard Delaunay refinement which requires an initial point set, potentially missing isolated components [S1, unofficial].

### Usage
Implemented as a partial version including only the treatment of isolated points [S1, unofficial]. The mesh is initialized and refined while isolated points exist [S1, unofficial]. The implementation allows initializing one or all isolated points at a given step [S1, unofficial].

## Sources

[S1] Probing for Surface Mesh Generation through Delaunay Refinement, https://inria.hal.science/hal-00747344v1, fetched 2026-10-26 (unofficial)
