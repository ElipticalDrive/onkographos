# Coherent Grid Traversal

Also called: Bundle Traversal, Coherent Grid Traversal (CGT) for Volume Rendering, Frustum-based Grid Traversal, Packetized Grid Traversal

## Overview
Coherent Grid Traversal (CGT) is a ray-grid traversal method that accelerates rendering by processing bundles (frustums) of coherent rays through a uniform grid slice-by-slice along a dominant axis, rather than cell-by-cell [S2, S3]. It solves the problem of state invalidation in standard Digital Differential Analyzer (DDA) traversal when rays within a packet diverge and demand different next cells [S2, S3]. Unlike [3DDDA](3061-digital-differential-analyzer-dda-traversal.md), which breaks when rays split, CGT maintains packet coherence by incrementally computing the bounding frustum's overlap with entire grid slices, allowing SIMD-accelerated traversal and efficient early termination [S2, S3].

## How it works
**Inputs:** A packet of $N \times N$ coherent rays (origin $\mathbf{o}_i$, direction $\mathbf{d}_i$), a uniform grid of dimensions $N_x \times N_y \times N_z$, and the grid's bounding box.
**Outputs:** A list of grid cells (or bricks) intersected by the packet frustum, ordered along the traversal axis.

**Data Structures:**
1.  **Ray Packet:** Array of rays. For primary rays, the packet is defined by four corner rays forming a frustum.
2.  **Grid:** Uniform grid mapping to canonical space $[0, N_x) \times [0, N_y) \times [0, N_z)$.
3.  **Overlap Box $B(i)$:** A 2D axis-aligned bounding box (AABB) in the $U, V$ plane (minor axes) for slice $k$ (major axis). Represented as $[u_{min}, v_{min}, u_{max}, v_{max}]$.
4.  **Increment Vector $\Delta B$:** Constant vector $[du_{min}, du_{max}, dv_{min}, dv_{max}]$ representing the slope of the frustum planes in grid space.

**Algorithm Steps:**

1.  **Coordinate Transformation:** Transform all ray origins and directions into the canonical grid coordinate system. In this space, the grid occupies $[0, N_x) \times [0, N_y) \times [0, N_z)$, and cell indices are obtained by truncating coordinates [S2, S3].

2.  **Determine Dominant Axis:** Identify the dominant axis $K$ of the first ray's direction (e.g., $\pm X, \pm Y, \pm Z$). This becomes the major traversal axis. The remaining two axes are $U$ and $V$ [S2, S3].
    *   *Constraint:* All rays in the packet must have the same sign along $K$ (angle $< \pi/2$) to ensure front-to-back traversal and enable early termination [S2, S3].

3.  **Frustum Construction:** Compute the bounding frustum of the ray packet.
    *   For primary rays, use the four corner rays to define the frustum planes.
    *   For general packets, compute the minimum and maximum $U$ and $V$ slopes along $K$ across all rays [S2, S3].

4.  **Initial Overlap Box $B(0)$:** Intersect the frustum with the first slice $k=0$ (or the first slice intersecting the grid). Compute the entry and exit points of the frustum planes with this slice. The bounding box $B(0)$ of these points in the $U, V$ plane defines the initial overlap [S2, S3].
    *   $B(0) = [u_{min}^{(0)}, v_{min}^{(0)}, u_{max}^{(0)}, v_{max}^{(0)}]$

5.  **Compute Increment $\Delta B$:** Calculate the change in the overlap box per unit step along $K$.
    $$ \Delta B = (du_{min}, du_{max}, dv_{min}, dv_{max}) $$
    where $du_{min/max}$ and $dv_{min/max}$ are the slopes of the frustum's bounding planes in the grid coordinate space [S2, S3].

6.  **Traverse Slices:** Iterate $k$ from $k_{min}$ to $k_{max}$ (determined by intersecting the frustum with the grid's bounding box along $K$) [S2, S3]:
    a.  **Increment Box:** Update the overlap box for the current slice using SIMD addition:
        $$ B(k) = B(k-1) + \Delta B $$
        This requires four floating-point additions, performable in a single SIMD instruction [S2, S3].
    b.  **Compute Cell Indices:** Convert the floating-point coordinates of $B(k)$ to integer cell indices $[i_0, i_1] \times [j_0, j_1]$ by truncating the min/max $U$ and $V$ values. This is done with a single SIMD float-to-int conversion instruction [S2, S3].
    c.  **Process Cells:** For each cell $(i, j)$ in the range $[i_0, i_1] \times [j_0, j_1]$:
        i.  **Fetch Data:** Retrieve the primitives (triangles/bricks) stored in cell $(i, j)$.
        ii. **Intersect:** Intersect all rays in the packet with the primitives in the cell.
        iii. **Cull/Tag:** Apply frustum culling and mailboxing (see Pitfalls) to avoid redundant tests [S2, S3].
    d.  **Early Termination:** If all rays in the packet have intersected their targets, terminate traversal [S2, S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coherent_grid_traversal(rays, grid, grid_dims):
    # 1. Transform to canonical grid space
    # grid_dims: (Nx, Ny, Nz)
    # rays: list of Ray objects with origin (ox, oy, oz) and direction (dx, dy, dz)
    
    # 2. Determine dominant axis K
    # Assume K is X for this example; U=Y, V=Z
    K = 'x'
    U = 'y'
    V = 'z'
    
    # Check sign consistency along K
    signs = [ray.d[K] > 0 for ray in rays]
    if not all(signs) and not all(not s for s in signs):
        raise ValueError("Rays must have same sign along dominant axis K")
        
    step_K = 1 if signs[0] else -1
    
    # 3. Construct Frustum Planes
    # For primary rays, use 4 corner rays to find min/max slopes in U, V along K
    # Slope = delta_coord / delta_K
    # We need min/max of (u - k * slope_u) and (v - k * slope_v)
    # Actually, we need the bounds of the frustum at any k.
    # The frustum is defined by 4 planes.
    # Let's compute the bounding box of the packet at k=0 and the deltas.
    
    # Find min/max U and V coordinates at k=0 (or entry slice)
    # Entry slice k_min is determined by intersecting frustum with grid bbox
    # For simplicity, assume we start at k=0 or compute k_min/k_max first
    
    # Compute initial bounds B(0) at k=0
    # B(0) = [u_min, v_min, u_max, v_max]
    # u_min = min(ray.u - ray.d[U]/ray.d[K] * ray.ox/K_offset) ... 
    # Simplified: Transform ray origins to grid space, then compute bounds.
    
    # Let's assume rays are already in grid space [0..Nx) etc.
    # Compute the 4 corner rays of the packet in grid space
    corner_rays = get_corner_rays(rays) # Returns 4 rays defining the frustum
    
    # Compute slopes of the frustum planes in U, V with respect to K
    # Plane equation: U = slope_U * K + intercept_U
    # We need min and max of these slopes and intercepts
    
    slopes = []
    intercepts = []
    for r in corner_rays:
        if r.d[K] == 0:
            continue # Handle parallel case separately or ignore if coherent
        slope = r.d[U] / r.d[K]
        slopes.append(slope)
        # Intercept at K=0
        intercept = r.o[U] - slope * r.o[K]
        intercepts.append(intercept)
        
    # We need min/max of (slope * K + intercept) for K in [k_min, k_max]
    # Since K increases monotonically, the min/max will be at k_min or k_max
    # However, the standard CGT approach computes B(k) incrementally.
    # B(k) = B(k-1) + Delta_B
    # Delta_B is the change in the bounding box per unit K.
    # The bounding box at K is defined by the min/max of the 4 planes.
    # Let P_i(K) = m_i * K + c_i.
    # The min U at K is min_i(P_i(K)).
    # The change in min U from K to K+1 is min_i(m_i) if the min plane doesn't change?
    # Actually, Wald 2006 states: Delta B = (du_min, du_max, dv_min, dv_max)
    # where du_min/max are the slopes of the bounding frustum planes.
    # This implies the "active" plane for the min/max doesn't change within a slice step?
    # Or rather, the bounding box of the frustum at K+1 is B(K) + Delta.
    # This is an approximation or exact for convex frustums?
    # Wald 2006: "B(i+1) = B(i) + Delta B ... requires four floating point additions"
    # This suggests Delta B is constant.
    
    # Let's compute Delta B as the slopes of the extreme planes.
    # du_min = min(slopes)
    # du_max = max(slopes)
    # dv_min = min(slopes_v) # Similar for V
    # dv_max = max(slopes_v)
    
    du_min = min(slopes)
    du_max = max(slopes)
    
    # Repeat for V
    slopes_v = []
    intercepts_v = []
    for r in corner_rays:
        slope = r.d[V] / r.d[K]
        slopes_v.append(slope)
        intercept = r.o[V] - slope * r.o[K]
        intercepts_v.append(intercept)
        
    dv_min = min(slopes_v)
    dv_max = max(slopes_v)
    
    Delta_B = (du_min, du_max, dv_min, dv_max)
    
    # Compute initial B(0) at k_min
    # k_min is the first slice intersecting the grid
    # Intersect frustum with grid bbox [0, Nx) x [0, Ny) x [0, Nz)
    # For simplicity, assume k_min = 0 and k_max = Nz
    # Compute B(0) by evaluating the 4 planes at K=0
    
    u_vals = [intercepts[i] for i in range(4)]
    v_vals = [intercepts_v[i] for i in range(4)]
    
    B = (min(u_vals), min(v_vals), max(u_vals), max(v_vals))
    
    k_min = 0
    k_max = grid_dims[0] # Assuming K is X
    
    cells_visited = []
    
    for k in range(k_min, k_max):
        # Increment B
        B = (B[0] + Delta_B[0], B[1] + Delta_B[1], B[2] + Delta_B[2], B[3] + Delta_B[3])
        
        # Truncate to integer cell indices
        # i0 = floor(B[0]), i1 = floor(B[2])
        # j0 = floor(B[1]), j1 = floor(B[3])
        # Note: Grid space is [0, Nx). Truncation is floor for positive, ceil for negative?
        # In canonical space, coordinates are positive.
        i0 = int(B[0]) # Truncate
        i1 = int(B[2])
        j0 = int(B[1])
        j1 = int(B[3])
        
        # Clamp to grid bounds
        i0 = max(0, min(i0, grid_dims[1]-1))
        i1 = max(0, min(i1, grid_dims[1]-1))
        j0 = max(0, min(j0, grid_dims[2]-1))
        j1 = max(0, min(j1, grid_dims[2]-1))
        
        # Process cells
        for i in range(i0, i1 + 1):
            for j in range(j0, j1 + 1):
                # Fetch primitives from cell (i, j, k)
                primitives = grid.get_primitives(i, j, k)
                
                # Intersect rays with primitives
                for ray in rays:
                    for prim in primitives:
                        if mailbox_check(ray.packet_id, prim.id):
                            continue
                        if frustum_cull(ray, prim, B): # Check if prim is outside frustum
                            continue
                        if intersect(ray, prim):
                            mailbox_tag(ray.packet_id, prim.id)
                            # Handle hit (e.g., store closest)
                            pass
```

**Edge Cases and Pitfalls:**
*   **Ray Divergence:** If rays in a packet span an angle $> \pi/2$ along $K$, they have different signs, violating the front-to-back assumption [S2, S3].
*   **Silhouette Hit Points:** If primary rays hit object silhouettes, the resulting shadow frustum can be very wide, causing the method to intersect large parts of the scene. Mitigation: Split the packet into subpackets if hit points are too far apart [S2, S3].
*   **Roll Angle:** When the bundle's roll angle is close to $\pi/4$ relative to the volume, block fetches can double [S1, S5, unofficial].
*   **Redundant Tests:** Grids do not conform tightly to geometry, and triangles may appear in multiple cells. Mailboxing and frustum culling are required to avoid excessive intersection tests [S2, S3].

## Subtopics

### [Sphere-Center Method](5200-sphere-center-method.md)
An optimization for glyph-based particle visualization within CGT. It stores each sphere in exactly one cell (the one containing its center) and traverses an enlarged frustum, eliminating indirection overhead.

### [Macrocell Hierarchical CGT](5464-macrocell-hierarchical-cgt.md)
Extends CGT with a recursive hierarchy of macrocells ($2^3$ children per parent) to handle large volumes. It uses an array stack for iterative traversal and skips empty macrocell slices.

### [Slice Parallelisation](6292-slice-parallelisation.md)
A parallelization scheme where processing elements are assigned to specific volume slices. It requires explicit data structures to map ray bundles to slices but reduces redundant block fetches.

### [Bundle Parallelisation](6293-bundle-parallelisation.md)
Distributes ray bundle processing across processing elements. It exploits spatial coherence but may result in redundant block fetches from neighboring bundles.

## Operating characteristics
**Time Complexity:** The traversal cost per step is independent of the number of rays in the packet, as the frustum overlap is computed incrementally with constant-time SIMD operations [S2].
**Memory Complexity:** Linear with respect to the grid size and the number of primitives stored in cells. Bricked volumes can compress empty space [S1, S5, unofficial].
**Performance:**
*   On a dual 3.2 GHz Xeon, CGT with 8x8 packets achieved 3.2–20.0 fps for static scenes at 1024x1024, compared to 0.30–1.59 fps for single-ray 3DDDA [S2, S3].
*   Frustum traversal visits 10–20 times fewer cells for 4x4 packets and over 50 times fewer for 8x8 packets compared to single-ray traversal [S2, S3].
*   Combined with mailboxing and frustum culling, ray-triangle intersection tests are reduced by a factor of 8.5 to 14, matching kd-tree performance [S2, S3].
*   For animated scenes, grid rebuild times are low (e.g., 1–14 ms for moderate scenes), allowing interactive frame rates (e.g., 15–35 fps) [S2, S3].
**Strengths:**
*   High performance for coherent ray packets.
*   Simple grid rebuild makes it suitable for animated/dynamic scenes [S2, S3].
*   SIMD-friendly incremental updates [S2, S3].
**Limitations:**
*   Uniform grid is ill-suited for highly complex scenes with high variation in geometry size/density (e.g., Boeing 777) [S2, S3].
*   Performance degrades if the bundle roll angle is near $\pi/4$ [S1, S5, unofficial].
*   Requires rays to be coherent and have the same sign along the dominant axis [S2, S3].

## Background
Coherent Grid Traversal was introduced in 2006 by Wald et al. in "Ray Tracing Animated Scenes using Coherent Grid Traversal" published in ACM Transactions on Graphics (SIGGRAPH) [S2, S3]. It was developed to address the performance gap between packet-based kd-tree tracers and grid-based tracers, particularly for animated scenes where kd-tree rebuilds are expensive. The method leverages SIMD instructions and frustum-based slice traversal to accelerate grid traversal. It has been adapted for volume rendering [S1, S5, unofficial] and particle visualization [S4].

## Sources
[S1] PPT - A Coherent Grid Traversal Algorithm for Volume Rendering PowerPoint Presentation - ID:991792, https://slideserve.com/kineta/a-coherent-grid-traversal-algorithm-for-volume-rendering-powerpoint-ppt-presentation, fetched 2026-10-24 (unofficial)
[S2] PDF Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~thiago/papers/grid.pdf, fetched 2026-10-25
[S3] PDF Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~wald/Publications/2006/CGT/grid.pdf, fetched 2026-10-25
[S4] A Coherent Grid Traversal Approach to Visualizing Particle ..., https://www.sci.utah.edu/~thiago/papers/pcgt_revised.pdf, fetched 2026-10-25
[S5] A Coherent Grid Traversal Algorithm for Volume Rendering, https://www.slideserve.com/iliana-house/a-coherent-grid-traversal-algorithm-for-volume-rendering, fetched 2026-10-26 (unofficial)
[S6] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
