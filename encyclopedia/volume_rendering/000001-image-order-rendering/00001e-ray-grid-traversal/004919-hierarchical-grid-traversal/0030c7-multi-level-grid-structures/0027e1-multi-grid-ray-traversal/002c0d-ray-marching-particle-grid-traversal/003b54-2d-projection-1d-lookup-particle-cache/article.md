# 2D Projection 1D Lookup Particle Cache

Also called: Field Value Cache, Particle Lookup Cache

## Overview
The 2D Projection 1D Lookup Particle Cache is a spatial data structure designed to optimize field value calculations in volume rendering by iterating only over particles within a fixed cutoff radius $R_c$ from a given sample point [S1]. It solves the problem of efficiently retrieving influencing particles for isosurface extraction (e.g., Marching Slices) by combining a 2D projection with a 1D search along the z-axis [S1]. Unlike 3D lookup grid approaches (such as those by Wyvill et al., Müller et al., and Triquet) which return approximately 6.5 times more particles due to cubic bounding volumes requiring expensive floating-point distance culling, this method returns exactly the particles within the spherical radius $R_c$ using only integer comparisons [S1].

## How it works
**Inputs:**
- A set of particles, each defined by its center position $(x_p, y_p, z_p)$, radius of influence, and a pointer to its data.
- A cutoff radius $R_c$ defining the maximum distance for influence.
- A cube size $S$ used for discretization.
- Grid dimensions $N_x, N_y, N_z$.

**Outputs:**
- A projection slice data structure allowing $O(1)$ access to candidate particle lists for any grid corner $(x, y)$.
- A list of particles influencing a specific field corner $(x, y, z)$ that lie within distance $R_c$.

**Data Structures:**
1.  **Projection Slice:** A 2D array of dimensions $(N_x + 1) \times (N_y + 1)$. Each element is a pointer to a doubly linked list [S1].
2.  **List Element:** Each node in the linked list contains:
    -   `minz`: Integer, the minimum z-slice index where the particle has influence.
    -   `maxz`: Integer, the maximum z-slice index where the particle has influence.
    -   `midz`: Integer, the z-slice index containing the particle's center.
    -   `particle_ptr`: Pointer to the particle object [S1].
    -   *Size:* 16 bytes per element (assuming 32-bit integers and pointers) [S1].

**Parameters:**
-   $R_c$: Cutoff radius. Typical value: 3 [S1].
-   $S$: Cube size. Typical value: 0.8 [S1].

**Algorithm Steps:**

1.  **Build the Cache:**
    a.  Sort all particles in ascending order of their z-coordinate using Quick-sort [S1].
    b.  For each particle, determine the 2D region in the $(x, y)$ plane where it has influence. This region is a disk (2D projection of the spherical influence) [S1].
    c.  For each grid column $(x, y)$ intersected by this disk, insert a new list element into the linked list at `ProjectionSlice[x][y]` [S1].
    d.  **Boundary Handling:** Do not insert elements into columns on the block boundaries (i.e., $x=0, x=N_x, y=0, y=N_y$). Clamp `minz` and `maxz` to the range $[1, N_z]$ to enforce field values of 0 along block boundaries [S1].
    e.  **Small Particle Adjustment:** If a particle's field of influence is smaller than $R_c$, shrink the radius used for insertion to ensure an exact lookup (no false positives) [S1].

2.  **Lookup for Corner $(x, y, z)$:**
    a.  Access the linked list at `ProjectionSlice[x][y]` [S1].
    b.  **Find Start:** Search the list for the bottommost entry where `midz` $\ge \lfloor z - d \cdot R_c / S \rfloor$. Here, $d$ is a scaling factor related to the slab thickness or grid spacing, and $\lfloor \cdot \rfloor$ denotes the floor function (integer part) [S1]. *Note: The source formula uses `d*Rc/S*e`, interpreted as the integer threshold for the start of the search range.*
    c.  **Traverse:** Traverse the list upwards from the start entry to the topmost entry where `midz` $\le \lfloor z + d \cdot R_c / S \rfloor$. This traversal identifies all particles within a cylinder of radius $R_c$ and height $2 \cdot R_c$ centered at $z$ [S1].
    d.  **Cull:** For each element in the traversed range, perform integer comparisons:
        -   Check if `minz` $\le z$ AND `maxz` $\ge z$.
        -   If true, the particle influences the field at $z$ and is passed to the field calculation routine [S1].
    e.  **Optimization:** Update the pointer at `ProjectionSlice[x][y]` to point to the first valid element found in the current lookup. This speeds up subsequent lookups in the same column, as lookups are likely to occur near previous locations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structure Definition
class ParticleCacheElement:
    minz: int
    maxz: int
    midz: int
    particle: Particle
    next: ParticleCacheElement
    prev: ParticleCacheElement

class ProjectionSlice:
    # 2D array of pointers to ParticleCacheElement
    # Dimensions: (Nx + 1) x (Ny + 1)
    columns: List[List[ParticleCacheElement]]

def build_cache(particles: List[Particle], Nx: int, Ny: int, Nz: int, Rc: float, S: float) -> ProjectionSlice:
    # 1. Sort particles by z
    sorted_particles = quick_sort(particles, key=lambda p: p.z)
    
    # 2. Initialize Projection Slice
    proj_slice = ProjectionSlice()
    proj_slice.columns = [[None for _ in range(Ny + 1)] for _ in range(Nx + 1)]
    
    # 3. Insert particles
    for particle in sorted_particles:
        # Determine 2D disk influence in x,y
        # This requires calculating the bounding box of the sphere projected on xy plane
        # and iterating over grid cells (x, y) intersected by the disk
        x_min = floor(particle.x - Rc)
        x_max = ceil(particle.x + Rc)
        y_min = floor(particle.y - Rc)
        y_max = ceil(particle.y + Rc)
        
        for x in range(max(1, x_min), min(Nx, x_max)): # Avoid boundary columns 0 and Nx
            for y in range(max(1, y_min), min(Ny, y_max)):
                # Check if (x,y) is within the disk of influence
                # Distance from particle center to grid corner (x,y) in xy plane
                dist_xy = sqrt((x - particle.x)**2 + (y - particle.y)**2)
                
                if dist_xy <= Rc:
                    # Calculate z-range of influence
                    # Using Pythagorean theorem: dz = sqrt(Rc^2 - dist_xy^2)
                    dz = sqrt(max(0, Rc**2 - dist_xy**2))
                    
                    min_z_influence = particle.z - dz
                    max_z_influence = particle.z + dz
                    
                    # Clamp to valid slice range [1, Nz]
                    minz = max(1, floor(min_z_influence))
                    maxz = min(Nz, ceil(max_z_influence))
                    
                    # If particle influence is smaller than Rc, shrink radius for exact lookup
                    # [G] Implementation detail: if particle.radius < Rc, use particle.radius instead of Rc
                    # for the dist_xy check and dz calculation above.
                    
                    midz = floor(particle.z)
                    
                    # Create new element
                    new_elem = ParticleCacheElement()
                    new_elem.minz = minz
                    new_elem.maxz = maxz
                    new_elem.midz = midz
                    new_elem.particle = particle
                    new_elem.next = None
                    new_elem.prev = None
                    
                    # Insert into linked list at (x, y)
                    # [G] Assuming insertion at head for simplicity, or append to maintain sort order
                    # The source says "inserting a list element", and lists are sorted by z due to input sort.
                    # To maintain sorted order, we insert in correct position.
                    insert_sorted(proj_slice.columns[x][y], new_elem)
    
    return proj_slice

def lookup(particles: List[Particle], proj_slice: ProjectionSlice, x: int, y: int, z: float, Rc: float, S: float, d: float) -> List[Particle]:
    # 1. Access list
    col_list = proj_slice.columns[x][y]
    if col_list is None:
        return []
    
    # 2. Find start point
    # Threshold for midz
    start_threshold = floor(z - d * Rc / S)
    
    # Search for bottommost entry with midz >= start_threshold
    # [G] Since list is sorted by midz, we can traverse from current pointer or head
    current = col_list
    start_node = None
    
    # Optimization: Use cached pointer if available (not explicitly stored in simple list, 
    # but source says "update the pointer... with the first element found")
    # For this implementation, we assume we traverse to find it.
    
    while current is not None:
        if current.midz >= start_threshold:
            start_node = current
            break
        current = current.next
        
    if start_node is None:
        return []
    
    # 3. Traverse upwards
    end_threshold = floor(z + d * Rc / S)
    candidates = []
    
    node = start_node
    while node is not None and node.midz <= end_threshold:
        # 4. Cull with integer comparisons
        if node.minz <= z and node.maxz >= z:
            candidates.append(node.particle)
        
        # Update cache pointer for next lookup in this column
        # [G] In a real implementation, proj_slice.columns[x][y] would be updated to node
        # to speed up future lookups.
        proj_slice.columns[x][y] = node 
        
        node = node.next
        
    return candidates

def insert_sorted(head: ParticleCacheElement, new_elem: ParticleCacheElement):
    # Insert new_elem into sorted doubly linked list at head
    # Maintains order by midz
    if head is None:
        return new_elem
    
    if new_elem.midz < head.midz:
        new_elem.next = head
        head.prev = new_elem
        return new_elem
    
    current = head
    while current.next is not None and current.next.midz < new_elem.midz:
        current = current.next
        
    new_elem.next = current.next
    if current.next is not None:
        current.next.prev = new_elem
    current.next = new_elem
    new_elem.prev = current
    
    return head
```

## Operating characteristics
-   **Time Complexity:** Building the cache requires sorting particles $O(N \log N)$ and inserting them into 2D columns. Lookup is $O(1)$ for column access plus $O(K)$ for list traversal, where $K$ is the number of particles in the cylindrical volume. The integer culling step is fast.
-   **Memory Complexity:** Approximately $\pi R_c^2 / S^2$ elements are inserted per particle [S1]. With $R_c=3$ and $S=0.8$, this is ~45 elements per particle. At 16 bytes per element, this is 720 bytes per particle [S1]. A cache of 1,000 particles consumes 720KB [S1].
-   **Performance:** On a 3.2GHz Intel Core 2 Duo, this approach is faster than surface-following marching cubes due to improved memory locality (2D accesses vs 3D) and avoids hash map overhead [S1]. It uses significantly less memory than hash-based approaches that store all corner values, as it only stores values for active slices [S1].
-   **Strengths:** Exact particle retrieval within $R_c$ without floating-point distance checks. Reduced memory usage compared to 3D lookup grids. Fast lookup due to integer arithmetic and sorted lists.
-   **Limitations:** Linked list insertion can be expensive for very large particle counts. Memory usage grows linearly with particle count. Not suitable for unbounded rendering without additional strategies (e.g., keeping only necessary z-ranges) [S1].
-   **When to use:** Real-time particle isosurface extraction (e.g., Marching Slices) where exact particle culling is required and memory is constrained.
-   **When not to use:** When particle counts are extremely high and memory bandwidth is a bottleneck, or when approximate results are acceptable and 3D grid lookups are preferred for simplicity.

## Background
The 2D Projection 1D Lookup Particle Cache was introduced in the context of real-time particle isosurface extraction using Marching Slices with block subdivision [S1]. It was developed to address the inefficiencies of previous 3D lookup grid approaches used by Wyvill et al. [WMW86b, WMW86a], Müller et al. [MCG03, MST04a, MST*04b], and Triquet [TMC01]. These earlier methods used a 3D grid with cube size $R_c$, resulting in a search volume of $(3R_c)^3$ which returned 6.5 times more particles than the spherical volume $(4/3)\pi R_c^3$, necessitating expensive floating-point distance culling [S1]. The 2D projection method reduces insertion and lookup costs by leveraging the 2D nature of the slice-based rendering and the sorted order of particles along the z-axis [S1].

## Sources
[S1] Real-time particle isosurface extraction, https://doi.org/10.1145/1342250.1342256, fetched 2026-11-04
