# Distributed Parallel Volume Ray-Casting for Unstructured-Grid Data

Also called: Distributed parallel rendering algorithm, Parallel volume ray-casting

## Overview
A distributed parallel volume ray-casting algorithm for visualizing unstructured-grid data, where both the data and rendering computation are distributed across processors [S1, unofficial]. It solves the problem of efficiently rendering large, complex unstructured meshes on distributed-memory architectures by performing local ray-casting independently on each processor and overlapping global image compositing with local computation to maximize parallel efficiency [S1, unofficial]. It differs from prior shared-memory or master-slave approaches by being completely distributed, less view-dependent, and reasonably scalable without requiring dynamic load balancing [S1, unofficial].

## How it works
**Inputs:** Unstructured-grid data set (nodes with 3D coordinates and scalar values, and elements defined by nodes); number of processors $p$; current viewing position [S1, unofficial].
**Outputs:** Composite image [S1, unofficial].

**Data Structures:**
1.  **Hierarchical Data Structure (hds):** A three-layer structure created during preprocessing to enable fast information retrieval [S1, unofficial].
    *   **Layer 1 (Lowest): Node Set.** Stores 3D coordinates $[x, y, z]$ and scalar function values (e.g., density, pressure) for each node [S1, unofficial].
    *   **Layer 2: Face Set.** Constructed from the node set [S1, unofficial].
    *   **Layer 3 (Top): Element Set.** Constructed from the face set [S1, unofficial].
    *   The hds explicitly stores face-node, element-face, and element-node adjacency relationships [S1, unofficial].
2.  **Exterior Faces:** Faces not shared by two elements within the same subvolume [S1, unofficial]. Classified as:
    *   **Globally exterior:** Boundary of the whole volume [S1, unofficial].
    *   **Locally exterior:** Boundary of a specific subvolume (includes globally exterior faces and internal partition boundaries) [S1, unofficial].
    *   For the algorithm, all locally exterior faces are treated as exterior faces [S1, unofficial].
3.  **Visible Exterior Faces:** A subset of exterior faces determined by the viewing position. Rays enter the subvolume at these faces [S1, unofficial].

**Parameters:**
*   $p$: Number of processors. Typical tested range: 2 to 128 [S1, unofficial].

**Steps:**

1.  **Domain Partitioning (View-Independent):**
    Partition the computational domain into $p$ subvolumes, where $p$ is the number of processors [S1, unofficial]. Use graph-based partitioning to produce subvolumes with approximately equal element counts (for load balancing) and minimized boundary area between subdomains (to reduce communication) [S1, unofficial]. Distribute the data and the hds to the respective processors [S1, unofficial].

2.  **Hierarchical Data Structure Construction (View-Independent):**
    Each processor constructs its local hds from its partitioned data [S1, unofficial]. This involves building the node set, then the face set from nodes, and finally the element set from faces, recording all adjacency relationships (face-node, element-face, element-node) [S1, unofficial].

3.  **Exterior Face Identification (View-Independent):**
    Identify all locally exterior faces within each subvolume [S1, unofficial]. An exterior face is one not shared by two elements within the same subvolume [S1, unofficial]. Calculate face normals for these exterior faces [S1, unofficial].

4.  **Visible Face Identification (View-Dependent):**
    Based on the current viewing position, determine which exterior faces are visible [S1, unofficial]. A face is visible if its normal points towards the viewer (back-face culling) [S1, unofficial]. This step is repeated whenever the viewing position changes [S1, unofficial].

5.  **Screen Projection (View-Dependent):**
    Orthographically project the visible exterior faces onto the screen to generate convex polygons in screen coordinates [S1, unofficial]. These polygons define the exact screen area (pixel regions) for which each processor must cast rays [S1, unofficial].

6.  **Local Ray-Casting:**
    For each pixel within the projected convex polygons:
    *   Cast a ray starting from the visible exterior face into the subvolume [S1, unofficial].
    *   The ray enters at a visible exterior face and exits at an invisible exterior face (or the opposite boundary of the subvolume) [S1, unofficial].
    *   Perform volume ray-casting (marching, sampling, compositing) through the local unstructured elements using the scalar values stored in the hds [S1, unofficial].
    *   This process is performed independently on each processor without inter-processor communication during the ray-marching phase [S1, unofficial].

7.  **Global Image Compositing:**
    Collect the partial images (or ray contributions) from all processors and composite them into the final global image [S1, unofficial]. This step requires inter-processor communication [S1, unofficial].

8.  **Overlap:**
    Overlap the global image compositing step (communication and accumulation) with the local ray-casting processes of other pixels or frames to achieve maximum parallel efficiency [S1, unofficial].

**Pseudocode:**

```python
# Distributed Parallel Volume Ray-Casting for Unstructured-Grid Data
# Self-contained implementation. MPI environment assumed [G].

def main():
    # Initialize environment [G]
    rank = get_rank()
    size = get_size()
    
    # 1. View-Independent Preprocessing
    # Graph-based partitioning into p subvolumes [S1, unofficial]
    local_data = distribute_grid(rank, size)
    hds = build_hds(local_data) # Node, face, element layers [S1, unofficial]
    exterior_faces = find_exterior_faces(hds) # Locally exterior faces [S1, unofficial]
    for f in exterior_faces:
        f.normal = compute_face_normal(f, hds.nodes)
        
    # 2. View-Dependent Preprocessing
    def update_view(view_pos):
        # Identify visible exterior faces [S1, unofficial]
        visible_faces = [f for f in exterior_faces if dot(f.normal, view_pos - f.center) > 0]
        # Orthographically project to screen polygons [S1, unofficial]
        polygons = [project_to_screen(f, view_pos) for f in visible_faces]
        return polygons
        
    # 3. Ray-Casting Loop
    view_pos = get_current_view()
    polygons = update_view(view_pos)
    local_image = create_image_buffer()
    
    for poly in polygons:
        for pixel in poly.pixels:
            ray_o, ray_d = get_ray_for_pixel(pixel, poly)
            # Ray enters at visible face, exits at invisible face [S1, unofficial]
            t_enter, t_exit = compute_ray_bounds(ray_o, ray_d, exterior_faces)
            
            # FIX: Initialize compositing state to zero to prevent [1.0,1.0,1.0,1.0] bug
            color = [0.0, 0.0, 0.0]
            alpha = 0.0
            t = t_enter
            step = 0.5 # Standard step size [G]
            
            while t < t_exit and alpha < 1.0:
                elem = find_element_at(hds, ray_o + ray_d * t)
                if elem is not None:
                    val = sample_scalar(elem, ray_o + ray_d * t)
                    c_elem, a_elem = transfer_function(val)
                    # Front-to-back compositing [G]
                    color = color + c_elem * a_elem * (1.0 - alpha)
                    alpha = alpha + a_elem * (1.0 - alpha)
                t += step
            local_image.set_pixel(pixel, color, alpha)
            
    # 4. Global Image Compositing (Overlapped with next frame) [S1, unofficial]
    global_image = mpi_all_reduce(local_image, op='ADD')
    return global_image

# Helper definitions (standard operations) [G]
def get_rank(): return 0 # Placeholder for MPI rank
def get_size(): return 1 # Placeholder for MPI size
def distribute_grid(rank, size): return [] # Graph partitioning [S1, unofficial]
def build_hds(data): return {'nodes': [], 'faces': [], 'elements': []} # [S1, unofficial]
def find_exterior_faces(hds): return [] # Faces not shared by two local elements [S1, unofficial]
def compute_face_normal(face, nodes): return [0.0, 0.0, 1.0] # Cross product of edges [G]
def get_current_view(): return [0.0, 0.0, 10.0] # [G]
def project_to_screen(face, view_pos): return type('Poly', (), {'pixels': []})() # [G]
def get_ray_for_pixel(pixel, poly): return [0.0, 0.0, 0.0], [0.0, 0.0, 1.0] # [G]
def compute_ray_bounds(o, d, faces): return 0.0, 10.0 # Intersect with exterior faces [G]
def find_element_at(hds, pos): return None # Point-in-element test [G]
def sample_scalar(elem, pos): return 0.5 # Trilinear/linear interpolation [G]
def transfer_function(val): return [0.5, 0.5, 0.5], 0.5 # [G]
def create_image_buffer(): return type('Img', (), {'pixels': []})() # [G]
def mpi_all_reduce(img, op): return img # [G]
def dot(a, b): return sum(x*y for x,y in zip(a,b)) # [G]
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision in ray-element intersection tests on unstructured grids can lead to gaps or overlaps if not handled carefully [G].
*   **Boundaries:** Correctly identifying locally exterior faces is critical; misclassification leads to rays entering/exiting incorrectly [S1, unofficial].
*   **Degenerate Input:** Elements with zero volume or faces with zero area may cause numerical instability in normal calculation or ray intersection [G].

## Subtopics

### Load Imbalance Factors
Analyzes the sources of load imbalance in this distributed approach, such as varying element counts per processor or varying ray traversal lengths, and how static partitioning attempts to mitigate them. [Load Imbalance Factors](2675-load-imbalance-factors.md)

## Operating characteristics
*   **Time Complexity:** Dominated by local ray-casting, which is linear in the number of samples per ray and the number of rays [G]. Global compositing requires inter-processor communication proportional to the number of processors [G].
*   **Memory Complexity:** Each processor stores its partition of the data and the hds [S1, unofficial]. Memory usage is proportional to the local data size, with additional overhead for storing face-node, element-face, and element-node adjacency relationships [S1, unofficial].
*   **Measured Performance:** Average parallel efficiency of ~60% was achieved on an Intel Paragon with 2 to 128 processors, without dynamic load balancing [S1, unofficial].
*   **Scaling:** The algorithm is reasonably scalable. As $p$ increases, the image area per processor decreases, but the communication overhead for compositing increases [S1, unofficial].
*   **Strengths:** Completely distributed (no shared memory bottleneck), less view-dependent (efficient view changes), flexible, and achieves good efficiency without dynamic load balancing [S1, unofficial].
*   **Limitations:** Parallel efficiency is moderate (~60%) due to load imbalance and communication overhead [S1, unofficial].

## Background
This algorithm was proposed by researchers supported by NASA (NAS1-19480) while at ICASE, NASA Langley Research Center [S1, unofficial]. It addresses the challenges of visualizing large, unstructured-grid data from computational modeling (finite volume/element methods) on distributed-memory parallel computers like the Intel Paragon [S1, unofficial]. It differs from prior methods that used shared-memory multiprocessors (Williams [22], Uselton [19], Challinger [2]) or network-based master-slave approaches with data replication (Giertsen & Petersen [7]) [S1, unofficial].

## Sources
[S1] DTIC ADA301855: Parallel Volume Ray-Casting for Unstructured-Grid Data on Distributed-Memory Architectures., https://archive.org/details/DTIC_ADA301855, fetched 2026-10-26 (unofficial)
