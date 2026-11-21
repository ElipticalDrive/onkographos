# Hybrid Partitioning Parallel Ray Casting

Also called: High Performance Parallel Ray Casting, Hybrid Partitioning Parallel RC

## Overview
Hybrid Partitioning Parallel Ray Casting is a parallel volume rendering algorithm for shared-memory MIMD architectures that combines object-based partitioning for boundary cell projection with image-based partitioning for ray traversal to achieve load balancing [S2, unofficial]. It sets itself apart from its parent [Static Contiguous Image-Based Partitioning](11259-static-contiguous-image-based-partitioning.md) by decoupling the partitioning strategy: it uses object-based partitioning to handle the uneven distribution of boundary cells and image-based partitioning to handle the uneven distribution of ray traversal work, thereby avoiding the replicated ray casting costs associated with adaptive image sampling at processor boundaries [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, camera parameters (view direction, projection matrix), number of processors $P$, and the set of ray casting optimizations (early ray termination, adaptive depth sampling, adaptive image sampling).
*   **Output:** A 2D image buffer containing the final rendered pixel values.

**Data Structures:**
*   **Volume Data:** Regular 3D grid of voxels.
*   **Boundary Cell List:** List of voxels on the surface of the volume (front and back faces relative to view).
*   **Projection Buffers ($Z_n, Z_r$):** Two arrays of size $W \times H$ (image width and height).
    *   $Z_n[u, v]$: The nearest (minimum) depth intersection along the ray for pixel $(u, v)$. Initialized to $\infty$.
    *   $Z_r[u, v]$: The farthest (maximum) depth intersection along the ray for pixel $(u, v)$. Initialized to $-\infty$.
    *   Note: These buffers define the valid integration intervals for each ray.

**Numbered Steps:**

1.  **Preprocessing (Boundary Detection):** Identify all boundary cells in the volume. A boundary cell is a voxel that has at least one neighbor outside the volume or with different opacity properties, effectively defining the visible surface from the current view.

2.  **Object-Based Partitioning (Projection Phase):**
    *   Divide the volume into $P$ subsets such that each subset contains an approximately equal number of boundary cells. This ensures load balancing during the projection phase [S2, unofficial].
    *   Each processor $p$ independently scans its assigned volume subset.
    *   For each boundary cell detected in its subset, the processor projects the cell onto the image plane to determine the pixel coordinates $(u, v)$ and the depth $z$.
    *   The processor updates its local partial projection buffers $Z_n^p$ and $Z_r^p$:
        $$ Z_n^p[u, v] = \min(Z_n^p[u, v], z) $$
        $$ Z_r^p[u, v] = \max(Z_r^p[u, v], z) $$
    *   After all processors finish, the complete global projection buffers $Z_n$ and $Z_r$ are formed by combining the partial buffers (e.g., via a parallel reduction or atomic operations on shared memory):
        $$ Z_n[u, v] = \min_{p} (Z_n^p[u, v]) $$
        $$ Z_r[u, v] = \max_{p} (Z_r^p[u, v]) $$

3.  **Image-Based Partitioning (Traversal Phase):**
    *   Divide the image plane into $P$ large blocks of consecutive scanlines, one block per processor. The division is static and contiguous [S2, unofficial].
    *   Each processor $p$ is assigned a range of scanlines $[u_{start}, u_{end}]$.
    *   For each pixel $(u, v)$ in its assigned block:
        *   Check if the pixel is "viable" (i.e., $Z_n[u, v] < \infty$ and $Z_r[u, v] > -\infty$). If not, skip the pixel.
        *   Cast a ray from the camera through $(u, v)$.
        *   Perform ray integration (marching) only within the bounded interval $[Z_n[u, v], Z_r[u, v]]$.
        *   Apply ray-independent optimizations (early ray termination, adaptive depth sampling) directly within this interval [S2, unofficial].
        *   Apply ray-dependent optimizations (adaptive image sampling) with care to avoid replicated casting at shared pixels. The method uses static contiguous partitioning to completely avoid the cost of replicated ray casting from pixels shared by image sections assigned to different processors, without extra synchronization overhead [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_partitioning_parallel_ray_casting(volume, camera, P):
    # Step 1: Preprocessing
    boundary_cells = find_boundary_cells(volume)
    
    # Step 2: Object-Based Partitioning
    # Divide volume into P subsets with equal boundary cell counts
    volume_subsets = divide_volume_by_boundary_cells(boundary_cells, P)
    
    # Initialize global projection buffers
    Z_n = array(W, H, init=INF)
    Z_r = array(W, H, init=-INF)
    
    # Parallel Projection
    def processor_projection(subset_id, subset_voxels):
        local_Z_n = array(W, H, init=INF)
        local_Z_r = array(W, H, init=-INF)
        
        for voxel in subset_voxels:
            if is_boundary(voxel):
                u, v, z = project(voxel, camera)
                # Update local buffers
                if z < local_Z_n[u, v]:
                    local_Z_n[u, v] = z
                if z > local_Z_r[u, v]:
                    local_Z_r[u, v] = z
        
        # Combine with global buffers (atomic operations or reduction)
        for u in range(W):
            for v in range(H):
                if local_Z_n[u, v] < Z_n[u, v]:
                    Z_n[u, v] = local_Z_n[u, v]
                if local_Z_r[u, v] > Z_r[u, v]:
                    Z_r[u, v] = local_Z_r[u, v]

    # Launch P processors
    parallel_for p in range(P):
        processor_projection(p, volume_subsets[p])
        
    # Step 3: Image-Based Partitioning
    # Divide image into P contiguous scanline blocks
    scanline_blocks = divide_image_into_scanline_blocks(W, H, P)
    
    def processor_traversal(p, u_start, u_end):
        for v in range(H):
            for u in range(u_start, u_end):
                # Check viability
                if Z_n[u, v] == INF or Z_r[u, v] == -INF:
                    continue
                
                # Cast ray and integrate within bounds
                ray = cast_ray(camera, u, v)
                color = 0.0
                alpha = 0.0
                
                # March from Z_n to Z_r
                t = Z_n[u, v]
                while t < Z_r[u, v]:
                    # Adaptive depth sampling [S1, unofficial]
                    step_size = adaptive_step(ray, t)
                    sample = trilinear_interpolate(volume, ray.origin + ray.direction * t)
                    
                    # Composite
                    color, alpha = composite_step(sample, color, alpha)
                    
                    # Early ray termination [S1, unofficial]
                    if alpha > 1.0 - epsilon:
                        break
                    
                    t += step_size
                
                # Adaptive image sampling [S2, unofficial]
                # Handled carefully to avoid replication at boundaries
                image[u, v] = color

    # Launch P processors
    parallel_for p in range(P):
        u_start, u_end = scanline_blocks[p]
        processor_traversal(p, u_start, u_end)
        
    return image
```

**Parameters:**
*   $P$: Number of processors. Typical values depend on hardware (e.g., SGI Power Challenge had multiple CPUs).
*   $\epsilon$: Threshold for early ray termination. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Replicated Ray Casting:** Ray-dependent optimizations like adaptive image sampling can cause replicated work if pixels shared by different processor blocks are evaluated independently. The static contiguous image-based partitioning avoids this by ensuring that the decision to subdivide a pixel is made consistently or by avoiding the sharing altogether through the projection buffer bounds [S2, unofficial].
*   **Precision:** Depth values in $Z_n$ and $Z_r$ must be precise enough to avoid gaps or overlaps in the integration intervals.

## Operating characteristics
*   **Time Complexity:** The projection phase is $O(N_b)$ where $N_b$ is the number of boundary cells, distributed over $P$ processors. The traversal phase is $O(N_r)$ where $N_r$ is the number of viable pixels, distributed over $P$ processors. Overall, it is linear in the size of the volume and image, with a constant factor reduced by parallelism.
*   **Memory Complexity:** $O(V + I)$ where $V$ is the volume size and $I$ is the image size. The projection buffers require $O(I)$ space.
*   **Measured Performance:** The method was tested on an SGI Power Challenge (shared-memory MIMD) [S2, unofficial]. It achieves good load balancing by combining the strengths of both partitioning strategies.
*   **Strengths:** Avoids the dilemma of choosing between object-based and image-based partitioning. Leverages optimizations from the serial algorithm. No extra synchronization overhead for adaptive image sampling.
*   **Limitations:** Requires a shared-memory architecture. The static nature of the partitioning may lead to load imbalance if the distribution of boundary cells or viable pixels is highly skewed, though the equal boundary cell division mitigates this for the projection phase.

## Background
The method was developed for time-critical volume rendering on shared-memory MIMD architectures, specifically the SGI Power Challenge [S2, unofficial]. It builds on a serial ray casting algorithm that included optimizations like adaptive image sampling, early ray termination, and adaptive depth sampling [S1, unofficial]. The hybrid approach was proposed to overcome the limitations of existing parallel algorithms that had to choose between object-based or image-based partitioning, giving up the advantages of the other [S2, unofficial].

## Sources
[S1] Time-Critical Volume Rendering, https://ntrs.nasa.gov/api/citations/19990040407/downloads/19990040407.pdf, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19990040407: Time-Critical Volume Rendering, https://archive.org/details/NASA_NTRS_Archive_19990040407, fetched 2026-10-26 (unofficial)
