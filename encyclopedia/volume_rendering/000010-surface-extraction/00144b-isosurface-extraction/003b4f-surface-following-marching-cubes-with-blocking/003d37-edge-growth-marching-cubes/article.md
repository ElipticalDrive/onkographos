# Edge Growth Marching Cubes

Also called: Edge growth MC, Edge-growing MC algorithm, Marching Cube Algorithm Based on Edge Growth

## Overview
Edge Growth Marching Cubes is a surface extraction algorithm that resolves the topological ambiguities and disconnected holes inherent in standard Marching Cubes (MC) by treating the iso-surface as a connected manifold grown from seed triangles rather than processed voxel-by-voxel [S1]. Unlike standard MC, which evaluates each cube independently and may produce inconsistent topology at shared edges, this method propagates the surface through adjacent cubes via a queue of "growth edges," ensuring topological consistency across the entire reconstructed model [S1]. It is particularly suited for 3D reconstruction where only the main contour is required or for isolating specific connected components from scattered data by controlling the seed region [S1].

## How it works
The algorithm operates by initializing a set of seed triangles, typically derived from specific topological configurations, and then iteratively expanding the surface into neighboring voxels using a queue of edges that define the boundary of the currently reconstructed surface [S1].

### Inputs and Outputs
*   **Input:** A 3D scalar field (volume data) with associated voxel coordinates and gray values; an iso-surface threshold $Y$.
*   **Output:** A triangular mesh representing the iso-surface.

### Data Structures
The algorithm relies on three primary data structures [S1]:

1.  **Growth Queue:** A queue storing `GrowthEdge` objects. Each entry contains:
    *   `p1`, `p2`: 3D coordinates of the two endpoints of the edge.
    *   `g1`, `g2`: Gradient values (or scalar field values) at the two endpoints.
    *   `cube_coord`: The 3D integer coordinates of the cube to which this edge is to be grown.
2.  **Labeled Arrays:** Two boolean or integer arrays of size equal to the volume grid dimensions, used to track processing state [S1]:
    *   `processed_cube`: Records whether a cube has already been processed to generate triangles.
    *   `queued_cube`: Records whether a cube has already been added to the growth queue to prevent duplicate processing.

### Parameters
*   **Threshold ($Y$):** The scalar value defining the iso-surface [S1].
*   **Interpolation Limits:** The "three-segment interpolation" method requires four constants $q, m, n, p$ such that $0 < q < m < n < p < 1$ and $q + p = 1$ [S1]. These values determine how the interpolation ratio is clamped to handle numerical stability or specific topological constraints.
*   **Seed Selection Strategy:** Typically, all triangles generated from **Topology Configuration 1** in a single middle layer (where curvature changes are significant) are selected as seeds [S1]. Alternatively, seeds can be manually selected or restricted to a designated 3D area [S1].

### Algorithm Steps

1.  **Seed Selection:** Identify seed triangles. The source recommends using triangles from MC Topology Configuration 1 in a middle layer of the volume [S1]. For each seed triangle, extract its three edges. Insert these edges into the `Growth Queue` as initial `GrowthEdge` entries. Mark the corresponding cubes as `queued` and `processed` [S1].
2.  **Growth Loop:** While the `Growth Queue` is not empty:
    a.  Dequeue a `GrowthEdge` ($E$). Let the associated cube be $C$.
    b.  **Interpolation:** Calculate the intersection points of the iso-surface with the edges of cube $C$. The source specifies a "three-segment interpolation" method [S1].
    c.  **Mesh Generation:** Connect the calculated intersection points to form the triangular mesh for cube $C$, based on the topological configuration determined by the signs of the scalar values at the cube's vertices relative to $Y$ [S1].
    d.  **Identify New Edges:** Determine which edges of cube $C$ intersect the newly generated iso-surface. These are the "new growth edges" [S1].
    e.  **Queue Update:** For each new growth edge:
        i.  Check if the adjacent cube (the one sharing this edge) has been processed or queued.
        ii. If not, create a new `GrowthEdge` entry for this edge, pointing to the adjacent cube, and enqueue it.
        iii. Mark the adjacent cube as `queued` and `processed` [S1].
3.  **Termination:** The algorithm terminates when the `Growth Queue` is empty, implying the connected component defined by the seeds has been fully reconstructed [S1].

### Interpolation Formula
The source describes a "three-segment interpolation" method that modifies standard linear interpolation to handle edge cases or specific topological requirements [S1].

Given two vertices $p_1$ and $p_2$ with scalar values $hu_1$ and $hu_2$, and an iso-surface threshold $Y$:

1.  Calculate the standard linear interpolation ratio $k_{linear}$:
    $$ k_{linear} = \frac{Y - hu_1}{hu_2 - hu_1} $$
2.  The actual interpolation ratio $k$ is determined by comparing $|k_{linear}|$ against the limits $m$ and $n$:
    *   If $|k_{linear}| < m$, then $k = q$ (if $k_{linear} > 0$) or $k = -q$ (if $k_{linear} < 0$). *Note: The source implies symmetry based on $q+p=1$ and the range constraints, but explicitly states "When the absolute value of k is less than m, the absolute value of k is q" [S1].*
    *   If $|k_{linear}| > n$, then $k = p$ (if $k_{linear} > 0$) or $k = -p$ (if $k_{linear} < 0$). *Note: The source text cuts off after "greater than...", but the condition $q+p=1$ and the structure imply clamping to $p$ for large ratios.*
    *   If $m \le |k_{linear}| \le n$, then $k = k_{linear}$ (standard linear interpolation).
3.  Calculate the intersection point $p$:
    $$ p = p_1 + k(p_2 - p_1) $$

*Note: The source text is incomplete regarding the exact behavior for $|k_{linear}| > n$ and the sign handling for negative $k_{linear}$, but the clamping logic is defined by the bounds $q, m, n, p$ [S1].*

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def edge_growth_marching_cubes(volume, threshold, limits):
    """
    volume: 3D array of scalar values
    threshold: float Y
    limits: tuple (q, m, n, p) where 0 < q < m < n < p < 1 and q + p = 1
    """
    # Initialize labeled arrays
    processed_cube = zeros_like(volume, dtype=bool)
    queued_cube = zeros_like(volume, dtype=bool)
    
    growth_queue = Queue()
    
    # Step 1: Seed Selection
    # Select middle layer where curvature changes are significant
    # For simplicity, assume we identify seed cubes based on Topology Config 1
    seed_cubes = find_seed_cubes(volume, threshold, limits)
    
    for cube in seed_cubes:
        # Generate triangles for this cube using standard MC logic but 
        # constrained to Config 1 or general MC lookup
        triangles = generate_triangles(cube, volume, threshold, limits)
        
        # Extract edges from these triangles
        edges = get_edges_from_triangles(triangles)
        
        for edge in edges:
            # Edge is defined by two vertices. We need the cube coordinates
            # to grow into. The edge belongs to the boundary of the current cube.
            # The "cube to which the edge is to be grown" is the neighbor 
            # sharing this edge.
            neighbors = get_neighbors_sharing_edge(cube, edge)
            
            for neighbor in neighbors:
                if not queued_cube[neighbor]:
                    growth_queue.enqueue(GrowthEdge(
                        p1=edge.p1, p2=edge.p2,
                        g1=volume[edge.p1], g2=volume[edge.p2],
                        cube_coord=neighbor
                    ))
                    queued_cube[neighbor] = True
                    processed_cube[neighbor] = True # Mark as processed to avoid re-adding? 
                    # Note: Source says "records how the cube's growth is put into the growth queue"
                    # and "records how each cube is processed". 
                    # Usually, we mark queued to prevent duplicate queue entries.
                    # We mark processed when we actually generate triangles.
    
    # Re-initialize processed to track actual mesh generation
    processed_cube = zeros_like(volume, dtype=bool)
    
    # Re-enqueue seeds properly? 
    # The source says: "put the growth edge formed by the seed triangles into the growth queue"
    # and "records how each cube is processed".
    # Let's refine the seed step to match the loop logic.
    
    growth_queue.clear()
    queued_cube = zeros_like(volume, dtype=bool)
    processed_cube = zeros_like(volume, dtype=bool)
    
    for cube in seed_cubes:
        triangles = generate_triangles(cube, volume, threshold, limits)
        edges = get_edges_from_triangles(triangles)
        for edge in edges:
            neighbors = get_neighbors_sharing_edge(cube, edge)
            for neighbor in neighbors:
                if not queued_cube[neighbor]:
                    growth_queue.enqueue(GrowthEdge(
                        p1=edge.p1, p2=edge.p2,
                        g1=volume[edge.p1], g2=volume[edge.p2],
                        cube_coord=neighbor
                    ))
                    queued_cube[neighbor] = True
                    # Do not mark processed yet, only when dequeued
    
    # Step 2-4: Growth Loop
    while not growth_queue.is_empty():
        edge = growth_queue.dequeue()
        cube = edge.cube_coord
        
        # Check if already processed (robustness)
        if processed_cube[cube]:
            continue
            
        # Step 2: Intersect and Mesh
        # Determine topology of cube based on vertex values
        topology = get_topology(cube, volume, threshold)
        
        # Calculate intersection points using three-segment interpolation
        intersection_points = []
        for cube_edge in cube.edges:
            v1, v2 = cube_edge.vertices
            val1, val2 = volume[v1], volume[v2]
            
            # Check if edge intersects iso-surface
            if (val1 - threshold) * (val2 - threshold) < 0:
                # Calculate interpolation ratio k
                k_linear = (threshold - val1) / (val2 - val1)
                k = clamp_interpolation(k_linear, limits)
                
                # Calculate point
                p = v1 + k * (v2 - v1)
                intersection_points.append(p)
        
        # Connect points to form triangles
        triangles = connect_intersection_points(intersection_points, topology)
        add_to_mesh(triangles)
        
        processed_cube[cube] = True
        
        # Step 3: Identify new growth edges
        # Find edges of this cube that intersect the surface
        new_edges = []
        for cube_edge in cube.edges:
            v1, v2 = cube_edge.vertices
            val1, val2 = volume[v1], volume[v2]
            if (val1 - threshold) * (val2 - threshold) < 0:
                new_edges.append(cube_edge)
        
        # Add to queue
        for cube_edge in new_edges:
            # Find the neighbor cube sharing this edge
            # The neighbor is the one NOT in the current cube's set of 8 vertices?
            # Or simply the adjacent voxel.
            neighbor = get_neighbor_sharing_edge(cube, cube_edge)
            
            if neighbor is not None and not queued_cube[neighbor]:
                growth_queue.enqueue(GrowthEdge(
                    p1=cube_edge.p1, p2=cube_edge.p2,
                    g1=volume[cube_edge.p1], g2=volume[cube_edge.p2],
                    cube_coord=neighbor
                ))
                queued_cube[neighbor] = True

def clamp_interpolation(k, limits):
    q, m, n, p = limits
    abs_k = abs(k)
    sign = 1 if k >= 0 else -1
    
    if abs_k < m:
        return sign * q
    elif abs_k > n:
        return sign * p
    else:
        return k

def find_seed_cubes(volume, threshold, limits):
    # Simplified: Return cubes in the middle layer with specific topology
    # In practice, this scans for Topology Config 1 triangles
    pass

def get_neighbors_sharing_edge(cube, edge):
    # Returns list of adjacent cubes sharing the edge
    pass
```

### Edge Cases and Pitfalls
*   **Seed Selection Balance:** Selecting too few seed triangles may fail to reconstruct the model due to noise or disconnected components, while selecting too many wastes computational time on seed selection [S1].
*   **Ambiguity Resolution:** By growing from seeds and propagating connectivity, the algorithm avoids the standard MC ambiguity problem where independent cube processing leads to holes or inconsistent topology at shared edges [S1].
*   **Incomplete Source Details:** The exact behavior of the three-segment interpolation for negative interpolation ratios and the precise definition of the "17 topological configurations" mentioned in the source are not fully detailed in the provided text [S1].

## Operating characteristics
*   **Time Complexity:** The algorithm processes only cubes connected to the seed region. In the worst case (full volume reconstruction), it approaches $O(N)$ where $N$ is the number of voxels, but with a lower constant factor than standard MC due to skipping empty or non-intersecting cubes outside the surface. The queue operations add overhead proportional to the surface area.
*   **Memory Complexity:** Requires storage for the volume data, two labeled arrays ($O(N)$), and the growth queue ($O(S)$ where $S$ is the surface area).
*   **Strengths:** Ensures topological consistency; handles disconnected components by controlling seeds; efficient for extracting main contours or specific parts [S1].
*   **Limitations:** Requires careful seed selection; performance depends on the connectivity of the iso-surface; not suitable for extracting all iso-surfaces in a volume simultaneously without multiple seed sets [S1].

## Background
The Edge Growth Marching Cubes algorithm was proposed to address the limitations of standard Marching Cubes, specifically the topological ambiguities and holes caused by independent cube processing [S1]. It was introduced in a 2021 paper by authors publishing in *Virtual Reality & Intelligent Hardware* [S1]. The method leverages the continuity of iso-surfaces, treating the reconstruction as a growth process from seed triangles, a concept distinct from the voxel-centric approach of traditional MC [S1].

## Sources
[S1] A marching cube algorithm based on edge growth, https://doi.org/10.1016/j.vrih.2021.08.006, fetched 2026-10-26
