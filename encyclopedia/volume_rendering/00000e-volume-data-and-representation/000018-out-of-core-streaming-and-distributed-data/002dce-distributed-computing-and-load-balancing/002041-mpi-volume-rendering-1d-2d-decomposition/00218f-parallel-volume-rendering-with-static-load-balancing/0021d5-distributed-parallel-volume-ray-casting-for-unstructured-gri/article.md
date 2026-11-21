# Distributed Parallel Volume Ray-Casting for Unstructured-Grid Data

Also called: Distributed parallel rendering algorithm, Parallel volume ray-casting

## Overview
This method is a distributed parallel volume ray-casting algorithm designed to render unstructured-grid data by distributing both the data and the rendering computation across $p$ processors [S1, S2, unofficial]. It solves the challenge of visualizing large, complex geometries from finite volume or finite element simulations on distributed-memory architectures [S1, S2, unofficial]. Unlike prior shared-memory or replicated master-slave approaches, this algorithm is completely distributed, requires inter-processor communication only for the final image compositing step, and achieves reasonable scalability without dynamic load balancing [S1, S2, unofficial].

## How it works
**Inputs:** Unstructured-grid dataset (nodes, elements, scalar values), number of processors $p$, camera/viewing position [S1, S2, unofficial].
**Outputs:** Composite 2D image [S1, S2, unofficial].

**Data Structures:**
1.  **Hierarchical Data Structure (hds):** A three-layer structure constructed during preprocessing to enable fast retrieval of connectivity [S1, S2, unofficial].
    *   **Layer 1 (Lowest): Node Set.** Stores 3D coordinates $[x, y, z]$ and scalar function values (e.g., density, pressure) for each node [S1, S2, unofficial].
    *   **Layer 2: Face Set.** Constructed from the node set [S1, S2, unofficial].
    *   **Layer 3 (Top): Element Set.** Constructed from the face set [S1, S2, unofficial].
    *   The hds explicitly stores face-node, element-face, and element-node adjacency relationships [S1, S2, unofficial].
2.  **Exterior Faces:** Faces not shared by any element [S2, unofficial]. Classified as:
    *   **Globally Exterior:** Boundary of the entire volume [S2, unofficial].
    *   **Locally Exterior:** Boundary of a subvolume (includes globally exterior faces and globally interior faces that become exposed after partitioning) [S1, S2, unofficial].
    *   *Note:* The algorithm treats all locally exterior faces identically as "exterior faces" for ray entry/exit logic [S1, S2, unofficial].

**Numbered Steps:**
1.  **Domain Partitioning:** Partition the computational domain into $p$ subvolumes, where $p$ is the number of processors [S1, S2, unofficial]. Use a graph-based bisection/partitioning algorithm to produce subvolumes containing approximately the same number of elements and minimizing boundary area between subdomains [S1, S2, unofficial]. Distribute the data to the $p$ processors [S1, S2, unofficial].
2.  **View-Independent Preprocessing (per processor):** Each processor independently executes:
    *   Construct the hds from the local data [S1, S2, unofficial].
    *   Identify all exterior faces (faces not shared by elements in the local subvolume) [S1, S2, unofficial].
    *   Calculate face normals for all faces [S2, unofficial].
    *   This step is performed once [S2, unofficial].
3.  **View-Dependent Preprocessing (per processor, per frame):**
    *   Determine which exterior faces are visible from the current camera position [S1, S2, unofficial].
    *   Orthographically project the visible exterior faces onto the screen to generate a set of convex polygons in screen coordinates [S1, S2, unofficial].
    *   These polygons define the exact pixel regions (screen area) for casting rays [S1, S2, unofficial].
4.  **Local Ray-Casting (per processor):**
    *   Cast rays starting from the visible exterior faces into the subvolume [S1, S2, unofficial].
    *   Rays enter at a visible exterior face and exit at an invisible exterior face [S1, S2, unofficial].
    *   Perform local ray-casting of the subvolume's data independently of other processors [S1, S2, unofficial].
5.  **Global Image Compositing:**
    *   Overlap the global image compositing process (which requires inter-processor communication) with the local ray-casting processes to maximize parallel efficiency [S1, S2, unofficial].
    *   Combine the partial images from all processors into the final image [S1, S2, unofficial].

**Formulas:**
*   Ray position at parameter $r$: $P(r) = O + r \cdot D$, where $O$ is the ray origin and $D$ is the normalized direction vector.
*   Front-to-back compositing for each step: 
    $C_{out} = C_{in} + C_{elem}(1 - \alpha_{in})$
    $\alpha_{out} = \alpha_{in} + \alpha_{elem}(1 - \alpha_{in})$
    where $C$ is color, $\alpha$ is opacity, subscript $in$ is accumulated, and $elem$ is the current element sample [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distributed_parallel_volume_ray_casting(dataset, p, camera):
    # Step 1: Domain Partitioning
    subvolumes = graph_based_partition(dataset, p)
    
    # Step 2 & 3: Preprocessing (View-Independent and View-Dependent)
    local_hds = build_hierarchical_data_structure(subvolumes.local_data)
    exterior_faces = identify_exterior_faces(local_hds)
    calculate_face_normals(exterior_faces)
    visible_exterior_faces = get_visible_faces(exterior_faces, camera)
    screen_polygons = orthographic_project(visible_exterior_faces, camera)
    
    # Step 4: Local Ray-Casting
    local_image = render_local_subvolume(screen_polygons, local_hds, camera)
    
    # Step 5: Global Image Compositing
    composite_image = mpi_all_reduce_composite([local_image], p)
    return composite_image

def graph_based_partition(dataset, p):
    """Partitions domain into p subvolumes with equal element counts and minimized boundary."""
    # [G] Standard graph-partitioning (e.g., METIS) on element adjacency graph
    return partition_by_graph(dataset.elements, p)

def build_hierarchical_data_structure(data):
    """Constructs the 3-layer hds: Node Set -> Face Set -> Element Set."""
    node_set = [{'pos': n.coords, 'value': n.scalar} for n in data.nodes]
    face_set = []
    for f in data.faces:
        verts = [node_set[i]['pos'] for i in f.node_indices]
        normal = compute_normal(verts)
        face_set.append({'nodes': f.node_indices, 'normal': normal, 'centroid': mean(verts)})
    element_set = [{'faces': e.face_indices, 'nodes': e.node_indices} for e in data.elements]
    return {'nodes': node_set, 'faces': face_set, 'elements': element_set}

def identify_exterior_faces(hds):
    """Identifies faces not shared by any element in the local subvolume."""
    exterior_faces = []
    for face in hds['faces']:
        if not is_shared(face, hds['elements']):
            exterior_faces.append(face)
    return exterior_faces

def calculate_face_normals(faces):
    """Computes outward normals for all faces."""
    for face in faces:
        face['normal'] = normalize(face['normal'])

def get_visible_faces(exterior_faces, camera):
    """Determines which exterior faces are visible from the current camera."""
    visible = []
    for face in exterior_faces:
        view_dir = normalize(camera['pos'] - face['centroid'])
        if dot(face['normal'], view_dir) > 0:
            visible.append(face)
    return visible

def orthographic_project(faces, camera):
    """Projects visible exterior faces onto the screen to get convex polygons."""
    polygons = []
    for face in faces:
        poly = []
        for idx in face['nodes']:
            x, y = project_to_screen(camera, idx)
            poly.append((x, y))
        polygons.append(poly)
    return polygons

def render_local_subvolume(screen_polygons, hds, camera):
    """Casts rays from screen polygons into the subvolume."""
    width, height = camera['resolution']
    image = [[(0.0, 0.0, 0.0, 0.0) for _ in range(width)] for _ in range(height)]
    step_size = 0.5
    for poly in screen_polygons:
        for y in range(height):
            for x in range(width):
                if point_in_polygon((x, y), poly):
                    ray_o = screen_to_world(camera, x, y)
                    ray_d = camera['direction']
                    color, alpha = cast_ray(ray_o, ray_d, step_size, hds, camera)
                    image[y][x] = composite_pixel(image[y][x], (color, alpha))
    return image

def cast_ray(origin, direction, step_size, hds, camera):
    """Performs volume ray-casting through the local unstructured grid."""
    r = 0.0
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_alpha = 0.0
    max_dist = 1000.0
    while r < max_dist:
        point = origin + direction * r
        elem = find_containing_element(point, hds)
        if elem is not None:
            val = sample_scalar(point, elem, hds)
            c, a = transfer_function(val)
            accumulated_color = [accumulated_color[i] + c[i] * (1.0 - accumulated_alpha) for i in range(3)]
            accumulated_alpha = accumulated_alpha + a * (1.0 - accumulated_alpha)
            if accumulated_alpha >= 1.0:
                break
        r += step_size
    return accumulated_color, accumulated_alpha

def mpi_all_reduce_composite(images, p):
    """Combines partial images from all processors."""
    # [G] Standard MPI_Allreduce with additive blending
    result = [[(0.0, 0.0, 0.0, 0.0) for _ in range(width)] for _ in range(height)]
    for img in images:
        for y in range(height):
            for x in range(width):
                result[y][x] = composite_pixel(result[y][x], img[y][x])
    return result
```

**Parameters:**
*   $p$: Number of processors used for domain partitioning and parallel execution. Typical tested range: 2 to 128 [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Globally Interior Faces Becoming Locally Exterior:** After domain partitioning, faces that were interior to the global volume may become boundaries of a subvolume. The algorithm handles this by treating all locally exterior faces identically, ignoring the global/local distinction to simplify ray entry/exit logic [S2, unofficial].
*   **Precision:** Not given in the sources.
*   **Boundaries:** Ray entry and exit are strictly defined by visible and invisible exterior faces, respectively [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** The hierarchical data structure (hds) incurs additional memory space for storing face-node, element-face, and element-node relationships [S1, S2, unofficial].
*   **Measured Performance:** Average 60% parallel efficiency achieved on an Intel Paragon system using 2 to 128 processors, without dynamic load balancing [S1, S2, unofficial].
*   **Scaling:** The algorithm is reasonably scalable; as more processing nodes are used, the image area each processor handles decreases, offsetting the increased number of compositing layers [S1, unofficial].
*   **Strengths:** Completely distributed (data and computation), less view-dependent, flexible, and requires inter-processor communication only for the final image compositing step [S1, S2, unofficial].
*   **Limitations:** Without dynamic load balancing, efficiency is limited to ~60% [S1, S2, unofficial]. The prototype implementation handled only tetrahedral cells, though the algorithm can be generalized [S2, unofficial].
*   **When to use:** Large-scale parallel simulations where data is too large to move for postprocessing, requiring in-situ or postprocess visualization [S2, unofficial].

## Background
*   **Origin:** Proposed by researchers at NASA ICASE [S1, S2, unofficial].
*   **Date:** 1996 (based on NTRS record date) [S2, unofficial].
*   **History:** Developed to address the challenge of visualizing large, unstructured-grid data from finite volume/element methods on distributed-memory parallel computers like the Intel Paragon [S1, S2, unofficial]. It differs from prior methods by being completely distributed and less view-dependent [S1, S2, unofficial].
*   **Usage:** Designed primarily as a postprocess for large-scale parallel simulations, but flexible enough to support runtime monitoring of in-situ simulation data [S2, unofficial]. Prototype implementation handled only tetrahedral cells [S2, unofficial].

## Sources
[S1] DTIC ADA301855: Parallel Volume Ray-Casting for Unstructured-Grid Data on Distributed-Memory Architectures., https://archive.org/details/DTIC_ADA301855, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19960003512: Parallel volume ray-casting for unstructured-grid data on distributed-memory architectures, https://archive.org/details/NASA_NTRS_Archive_19960003512, fetched 2026-10-26 (unofficial)
