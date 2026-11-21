# Uncoupled Parallel BTT LOD

Also called: Parallel BTT LOD, Parallel Bintree Tessellation, Uncoupled Parallel Mesh Refinement, Uncoupled Parallel Refinement, Uncoupled Parallel View Dependent Level of Detail Rendering, Uncoupled Parallel View Dependent Level of Detail Rendering of Bintree Triangulations

## Overview
Uncoupled Parallel BTT LOD is a parallel technique for view-dependent continuous level of detail (LOD) rendering of regular height field terrain using Bintree Triangulations (BTT) [S1, unofficial]. It solves the high CPU overhead of dynamic mesh generation by splitting the terrain mesh into two independent halves along the observer's line of sight, allowing each half to be tessellated simultaneously on separate processors [S1, unofficial]. Unlike concurrent vertex insertion algorithms that schedule access to shared resources, this method decouples the mesh into separate problems with independent vertex pools, avoiding race conditions while maintaining mesh continuity through boundary consistency enforcement [S1, unofficial].

## How it works
**Inputs:**
- A high-resolution height map terrain data [S1, unofficial].
- Observer position for each frame [S1, unofficial].

**Outputs:**
- A rendered terrain mesh with continuous LOD, split into two consistent sub-meshes [S1, unofficial].

**Data Structures:**
- **Bintree Triangle Structure (BTT):** Each patch contains two BTT structures at the coarsest level. Triangles are linked by pointers to their three neighbors [S1, unofficial].
- **Vertex Pool:** Two separate pools for vertex allocation, one for each half of the mesh, allowing independent refinement without sharing resources [S1, unofficial].
- **Height Map:** The only shared resource between the two tessellation threads [S1, unofficial].

**Parameters:**
- $t$: An adaptive tolerance parameter that depends on the distance of a given point to the observer [S1, unofficial].
- $L$: The observer line used to partition the mesh [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    a. Divide the terrain data into $N$ square patches [S1, unofficial].
    b. Initialize each terrain patch with two BTT structures at the coarsest level (every patch contains two triangles). The height coordinate of each triangle vertex is given by the corresponding value in the height map [S1, unofficial].
    c. Link all triangles of the BTT structures within each half of the mesh together, leaving an unconnected division running through the middle of the mesh where the observer line $L$ passes [S1, unofficial].
    d. Define two separate pools for vertex allocation, one for each half of the mesh [S1, unofficial].

2.  **Per-Frame Processing (While the observer traverses the terrain):**
    a. **Partitioning:** For the current observer position, partition the mesh along the observer line $L$. Identify patches at each side of the observer [S1, unofficial].
    b. **Parameter Calculation:** Calculate the common tolerance parameter $t$ along the line $L$ for both adjacent borders [S1, unofficial].
    c. **Assignment:** Assign each submesh and its associated parameters $t$ to a separate processor/thread for tessellation [S1, unofficial].
    d. **Parallel Tessellation:** Each thread adaptively calculates its submesh using BTT structures until the desired level of detail is achieved. The refinement condition for a triangle is: if the distance between the middle point of the triangle's base and the corresponding point in the Height Map is bigger than the calculated $t$, split the triangle [S1, unofficial].
    e. **Consistency Enforcement:** Before rendering, ensure the two halves are consistent. Force the tolerance factor $t$ at both adjacent borders to be exactly the same. This ensures the same level of detail is used on both halves and the stop refining condition is identical at the union [S1, unofficial]. This produces a mesh with duplicate vertices at the union [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_terrain(height_map):
    # Divide terrain into N square patches
    patches = divide_into_patches(height_map)
    
    # Initialize BTT structures
    btt_structures = []
    for patch in patches:
        # Each patch initialized with two BTT structures at coarsest level
        btt_structures.append(create_btt_structures(patch))
        
    # Link triangles within each half of the mesh
    # Assuming mesh is split into left_half and right_half based on observer line L
    # Note: L is dynamic, but initial linking prepares the structure for decoupling
    # The "unconnected division" is conceptual, defined by the split line L at runtime
    link_triangles_in_halves(btt_structures)
    
    # Define two separate pools for vertex allocation
    vertex_pool_left = create_vertex_pool()
    vertex_pool_right = create_vertex_pool()
    
    return btt_structures, vertex_pool_left, vertex_pool_right

def parallel_tessellation(btt_structures, vertex_pool_left, vertex_pool_right, height_map, observer_position):
    # Partition mesh along observer line L
    left_patches, right_patches, line_L = partition_mesh(btt_structures, observer_position)
    
    # Calculate common tolerance parameter t along L
    # t depends on distance from observer
    t_left = calculate_tolerance(line_L, observer_position, left_patches)
    t_right = calculate_tolerance(line_L, observer_position, right_patches)
    
    # Enforce consistency: force t to be exactly the same at both adjacent borders
    t_common = t_left # or average, but source says "force... to be exactly the same"
    # This implies t_left = t_right = t_common for the boundary triangles
    
    # Assign submeshes to threads
    # Thread 1 handles left_patches
    # Thread 2 handles right_patches
    
    # Parallel execution
    left_mesh = adaptively_refine(left_patches, vertex_pool_left, height_map, t_common)
    right_mesh = adaptively_refine(right_patches, vertex_pool_right, height_map, t_common)
    
    # Consistency check is implicit in using t_common for boundary refinement
    # Duplicate vertices at union are created
    
    return left_mesh, right_mesh

def adaptively_refine(patches, vertex_pool, height_map, t):
    # While there are triangles to refine and vertices in pool
    while patches_has_refinable_triangles(patches) and vertex_pool_has_space():
        for triangle in patches:
            # Compute t for the given position (distance to observer)
            # Note: t is passed as common boundary value, but internal refinement
            # might use local t. Source says "compute t for the given position".
            # However, for consistency at union, the boundary t is forced.
            
            # Calculate distance from middle point of triangle base to height map
            dist = distance_from_base_to_height(triangle, height_map)
            
            # Compare with tolerance
            if dist > t:
                # Split the triangle
                split_triangle(triangle, vertex_pool)
                
    return patches
```

**Edge Cases and Pitfalls:**
- **Cracks:** If consistency is not enforced, the two parts of the meshes might not fit together, leaving cracks at the union [S1, unofficial]. This is handled by decoupling the mesh along the observer line and forcing the same tolerance factor $t$ at the borders [S1, unofficial].
- **Over-refinement:** Forcing the same tolerance factor causes over-refinement of the neighboring mesh, resulting in extra triangles on the coarser side of the mesh [S1, unofficial].
- **Access Violations:** Occur if CPUs try to allocate the same vertex simultaneously. Avoided by using separate vertex allocation pools for each decoupled submesh [S1, unofficial].
- **Load Balance:** Not exact due to differences in terrain geography on each side of the mesh division [S1, unofficial].
- **Scalability:** Balancing load on more than two processors is difficult because the mesh partitioning strategy needs generalization, which is harder due to decreasing LOD towards the horizon and changing mesh complexity [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Tessellation is done in almost half the time due to parallelization, though other processes like computing texture coordinates remain sequential [S1, unofficial].
- **Memory Complexity:** The structure took about 1.2 MB at low vertex rates and increased to a maximum of 17.2 MB at 180,000 vertices per frame. No considerable change in memory usage between serial and parallel versions [S1, unofficial].
- **Measured Performance:** On a dual Turion64 processor (1.6 GHz) with 1 GB RAM and Nvidia GeForce 6100:
    - At 180,000 vertices per frame: 23.78 fps (parallel) vs 17.03 fps (serial), a 39.64% increase [S1, unofficial].
    - At 100,000 vertices per frame: 39.85 fps (parallel) vs 23.61 fps (serial), a 68.78% increase [S1, unofficial].
    - At 20,000 vertices per frame: 73.06 fps (parallel) vs 71.04 fps (serial), a 2.84% increase [S1, unofficial].
- **Strengths:** Significant performance increase in frames per second, especially at high vertex counts (100,000–160,000). Dynamic mesh generation provides better representation than pre-calculated meshes like GeoMipMaps [S1, unofficial].
- **Limitations:** Load balance is not exact. Scalability to more than two processors is difficult. Over-refinement occurs at the boundary, creating extra triangles. Sequential overhead (e.g., texture coordinates) limits maximum speedup [S1, unofficial].
- **When to use:** When rendering regular height field terrain with high vertex counts on multi-core systems where dynamic LOD is preferred over pre-calculated meshes.
- **When not to use:** When scalability to many processors is required, or when memory bandwidth is a bottleneck due to shared height map access (though access is safe, it is still shared) [S1, unofficial].

## Background
- **Origin:** The method is based on parallel decoupled terminal-edge bisection techniques for 3D mesh generation described by Rivara et al. in 2006 [S1, unofficial].
- **History:** Introduced by Bernd Biedermann and María Cecilia Rivara in 2007 in the paper 'UNCOUPLED PARALLEL VIEW DEPENDANT LEVEL OF DETAIL RENDERING OF BINTREE TRIANGULATIONS' presented at GM/R [S1, unofficial].
- **Usage:** Used for real-time terrain rendering, particularly in scenarios requiring high-quality continuous LOD where CPU overhead is a concern [S1, unofficial].

## Sources
[S1] Biedermann B. and Cecilia Rivara M. (2007). UNCOUPLED PARALLEL VIEW DEPENDANT LEVEL OF DETAIL RENDERING OF BINTREE TRIANGULATIONS. In Proceedings of the Second International Conference on Computer Graphics Theory and Applications - GM/R, pages 333-338. DOI: 10.5220/0002083203330338. https://www.scitepress.org/Papers/2007/20832/, fetched 2026-10-30 [S1, unofficial]. (unofficial)
