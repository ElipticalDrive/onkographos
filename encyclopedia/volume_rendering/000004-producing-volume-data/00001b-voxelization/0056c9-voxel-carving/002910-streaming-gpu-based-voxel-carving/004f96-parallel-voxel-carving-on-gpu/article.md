# Parallel Voxel Carving on GPU

Also called: Parallel GPU Voxel Carving

## Overview
Parallel Voxel Carving on GPU is a method for reconstructing explicit 3D volumes from multi-view depth maps by executing voxel-by-voxel carving operations in parallel on a graphics processing unit [S1, unofficial]. It solves the problem of efficiently generating surface-aligned voxel data for large-scale scenes (e.g., urban environments) while minimizing GPU memory usage through a compact 8-byte data structure [S1, unofficial]. It differs from its parent method, Streaming GPU-based Voxel Carving, by explicitly detailing the parallel thread assignment strategy (y-axis columns) and the specific quantization of plane parameters, whereas the parent focuses on the broader streaming architecture and load balancing [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of $N$ input views, each consisting of a depth map and a color image. The volume is partitioned into kernels of size $V \times V \times V$ (typically $256^3$ voxels) to fit within GPU memory [S1, unofficial].
*   **Outputs:** A list of surface voxels. Each surface voxel stores 8 bytes: 4 bytes for plane parameters (quantized normal and distance) and 4 bytes for averaged color (RGBA) [S1, unofficial].

**Data Structures**
*   **Voxel Tag:** A single byte or integer per voxel indicating visibility mode:
    *   `0`: "uncarved" (initial state)
    *   `1`: "carved" (determined to be outside the object)
    *   `2`: "surface" (determined to be on the object boundary)
*   **Surface Voxel Layout (8 bytes):**
    *   Bytes 0-3: Plane quantities $(A, B, C, D)$, each quantized to a signed 1-byte integer.
        *   $A, B, C$: Normal vector components.
        *   $D$: Normalized distance, defined as the ratio of the signed distance between the plane and the voxel center to the half-diagonal of the voxel cube.
        *   Range: $-127$ to $+127$.
        *   Value $-128$: Indicates no intersection; the voxel is removed [S1, unofficial].
    *   Bytes 4-7: Color $(R, G, B, A)$.
        *   $R, G, B$: Averaged color components.
        *   $A$: Alpha component, used to store the voxel mode (e.g., mapping "surface" to a specific alpha value) [S1, unofficial].

**Algorithm Steps**
1.  **Initialization:** Tag all voxels in the current volume kernel as "uncarved" [S1, unofficial].
2.  **Thread Assignment:** Launch $V^2$ threads (e.g., $256^2$ threads for a $256^3$ volume). Each thread is assigned a column of voxels along the y-axis. This balances workload because urban scenes are typically flat on the x-z plane [S1, unofficial].
3.  **Carving Loop:** For each voxel in the assigned y-axis column:
    a.  If the voxel is already tagged "carved", skip to the next voxel [S1, unofficial].
    b.  Iterate through all input views $i = 0 \dots N-1$:
        i.  Project the voxel center onto the image plane of view $i$ to find the corresponding pixel coordinates.
        ii. Retrieve the depth value $d_{view}$ from the depth map at these coordinates.
        iii. Calculate the Euclidean distance $d_{voxel}$ from the voxel center to the camera position of view $i$.
        iv. If $d_{voxel} < d_{view}$, mark the voxel as "carved" and break the view loop (the voxel is occluded or outside) [S1, unofficial].
        v.  If $d_{voxel} > d_{view}$, the voxel is preserved in this view; continue to the next view [S1, unofficial].
4.  **Surface Tagging:** After checking all views, if the voxel is still "uncarved":
    a.  Check if the voxel is within a specified threshold distance of the depth map in any view.
    b.  If yes, tag the voxel as "surface" [S1, unofficial].
5.  **Color and Plane Computation (for Surface Voxels):**
    a.  **Color:** For each view where the voxel is visible, bilinearly interpolate the color from the neighboring pixels in the color image. Average these interpolated colors using weights based on the angle between the surface normal and the viewing direction (as per Debevec et al. [6]) [S1, unofficial].
    b.  **Plane:** If multiple planes intersect the voxel cube, compute the normal $(A, B, C)$ and distance $D$ as weighted averages of the intersecting planes, similar to signed distance and weight functions [3] (Note: Source [3] is not provided in the citation list, so the exact formula is not given) [S1, unofficial].
    c.  **Quantization:** Quantize $A, B, C, D$ to signed 1-byte integers in the range $[-127, 127]$. Set to $-128$ if no intersection [S1, unofficial].
    d.  Store the quantized plane and averaged color in the 8-byte structure [S1, unofficial].
6.  **Post-Processing (Optional):** For point-based rendering (e.g., QSplat), shift the center of each surface voxel to the nearest point on its weighted average plane [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_voxel_carving(volume_size, views, threshold):
    # volume_size: V (e.g., 256)
    # views: list of view objects with camera_pos, depth_map, color_map, project()
    # threshold: distance threshold for surface tagging
    
    V = volume_size
    num_threads = V * V
    # Each thread handles a y-column. 
    # Thread index t maps to (x, z) coordinates.
    # y is iterated within the thread.
    
    # Initialize voxel tags
    # voxel_tags[x][y][z] = 0 (uncarved)
    voxel_tags = [[[0 for _ in range(V)] for _ in range(V)] for _ in range(V)]
    
    # Surface voxel storage
    surface_voxels = []

    def thread_work(x, z):
        # Iterate over y-axis
        for y in range(V):
            voxel_pos = (x, y, z)
            
            # Skip if already carved
            if voxel_tags[x][y][z] == 1:
                continue
            
            is_carved = False
            
            # Loop over all views
            for view in views:
                # Project voxel to image plane
                u, v = view.project(voxel_pos)
                
                # Check bounds
                if not view.in_bounds(u, v):
                    continue
                
                # Get depth from depth map
                depth_val = view.depth_map[u][v]
                
                # Get distance from camera
                dist_to_cam = view.camera_pos.distance(voxel_pos)
                
                # Carving logic
                if dist_to_cam < depth_val:
                    voxel_tags[x][y][z] = 1 # Carved
                    is_carved = True
                    break # No need to check other views
                # If dist_to_cam > depth_val, preserve and check next view
            
            # If not carved, check for surface
            if not is_carved and voxel_tags[x][y][z] == 0:
                # Check threshold proximity to depth map in any view
                is_surface = False
                for view in views:
                    u, v = view.project(voxel_pos)
                    if view.in_bounds(u, v):
                        depth_val = view.depth_map[u][v]
                        dist_to_cam = view.camera_pos.distance(voxel_pos)
                        # Note: Source says "close to a depth map within a threshold"
                        # This implies checking if the voxel is near the surface defined by depth
                        # A common interpretation is checking if dist_to_cam is close to depth_val
                        if abs(dist_to_cam - depth_val) <= threshold:
                            is_surface = True
                            break
                
                if is_surface:
                    voxel_tags[x][y][z] = 2 # Surface
                    
                    # Compute color and plane
                    avg_color = compute_averaged_color(voxel_pos, views)
                    plane_params = compute_weighted_plane(voxel_pos, views)
                    
                    # Quantize plane params
                    A, B, C, D = quantize_plane(plane_params)
                    
                    # Store 8 bytes
                    surface_voxels.append({
                        'pos': voxel_pos,
                        'A': A, 'B': B, 'C': C, 'D': D,
                        'R': avg_color[0], 'G': avg_color[1], 
                        'B': avg_color[2], 'A': 2 # Mode indicator
                    })

    # Launch threads (conceptual)
    for x in range(V):
        for z in range(V):
            thread_work(x, z)
            
    return surface_voxels

def compute_averaged_color(voxel_pos, views):
    # Bilinear interpolation from each view
    colors = []
    for view in views:
        u, v = view.project(voxel_pos)
        if view.in_bounds(u, v):
            color = view.color_map.bilinear_interpolate(u, v)
            # Weight by angle between surface normal and viewing direction
            # Note: Surface normal is not known yet, so this is an iterative or 
            # simplified approximation in practice, or uses view direction as proxy
            # Source cites Debevec et al. [6]
            colors.append(color)
    
    if not colors:
        return (0, 0, 0, 0)
    
    # Average colors
    avg_r = sum(c[0] for c in colors) / len(colors)
    avg_g = sum(c[1] for c in colors) / len(colors)
    avg_b = sum(c[2] for c in colors) / len(colors)
    return (avg_r, avg_g, avg_b, 255)

def compute_weighted_plane(voxel_pos, views):
    # Weighted average of normals and distances
    # Source references [3] for signed distance and weight functions
    # Not given in sources: exact formula
    return (0, 0, 0, 0) # Placeholder as per rules

def quantize_plane(plane):
    A, B, C, D = plane
    # Quantize to signed 1-byte, range -127 to 127
    # If no intersection, return -128
    # Note: Source does not specify the scaling factor for quantization
    # assuming plane values are already normalized or scaled appropriately
    def q(val):
        if val is None: return -128
        # Example quantization logic (not specified in source)
        # Source says: "ratio of the signed distance... to the half-diagonal"
        # and "quantized into signed 1-byte values"
        # We assume the input plane values are in the range [-1, 1] or similar
        # and need to be scaled to [-127, 127]
        scaled = int(val * 127)
        return max(-127, min(127, scaled))
    
    return (q(A), q(B), q(C), q(D))
```

**Parameters**
*   **Volume Size per Kernel:** Approximately $256^3$ voxels, limited by GPU memory [S1, unofficial].
*   **Thread Count:** $256^2$ threads for a $256^3$ volume, where each thread carves 256 voxels independently [S1, unofficial].
*   **Distance D Definition:** Ratio of the signed distance between the plane and the voxel center to the half-diagonal of the voxel cube [S1, unofficial].
*   **Threshold:** Distance threshold for tagging a voxel as "surface" (Not given in the sources).

**Edge Cases and Pitfalls**
*   **Precision:** Quantization of plane parameters to 1-byte integers may introduce precision errors. Values of $-128$ indicate no intersection and must be handled to avoid rendering artifacts [S1, unofficial].
*   **Boundaries:** Projection of voxels outside the image plane must be checked to avoid out-of-bounds memory access [S1, unofficial].
*   **Degenerate Input:** If a voxel is not close to any depth map within the threshold, it remains "uncarved" and is effectively discarded (not stored as surface) [S1, unofficial].

## Subtopics

### [Scalable Reconstruction Scheme](14018-scalable-reconstruction-scheme.md)
A method for reconstructing large-scale urban scenes by partitioning the global volume into fixed-size sub-volumes that fit within GPU memory, processing them sequentially via streaming, and aggregating surface voxels on the CPU. It differs from monolithic GPU voxel carving by overcoming VRAM limits through a stream-processing architecture, enabling the reconstruction of volumes significantly larger than the available video memory.

## Operating characteristics
*   **Time Complexity:** $O(V^3 \cdot N)$, where $V$ is the volume dimension and $N$ is the number of views. Each voxel is processed once per view in the worst case [S1, unofficial].
*   **Memory Complexity:** $O(V^3)$ for voxel tags during carving, and $O(S)$ for output surface voxels, where $S$ is the number of surface voxels. Each surface voxel uses 8 bytes [S1, unofficial].
*   **Measured Performance:** A single reconstruction kernel handles approximately $256^3$ voxels. Performance drops rapidly as resolution increases beyond this limit due to memory constraints [S1, unofficial].
*   **Strengths:** High parallelism suitable for GPU; compact 8-byte data structure reduces memory bandwidth; effective for flat urban scenes due to y-axis column assignment [S1, unofficial].
*   **Limitations:** Limited by GPU memory size ($256^3$ voxels per kernel); requires streaming for larger scenes; precision loss due to 1-byte quantization [S1, unofficial].
*   **When to Use:** Large-scale urban scene reconstruction where memory is a constraint and scenes are relatively flat [S1, unofficial].
*   **When Not to Use:** Scenes with high vertical complexity where y-axis column assignment does not balance workload; scenes requiring high-precision plane parameters [S1, unofficial].

## Background
Parallel Voxel Carving on GPU was introduced in the paper "GPU-based Scalable Volumetric Reconstruction for Multi-view Stereo" [S1, unofficial]. It builds on the concept of voxel carving, transforming it into a parallel GPU algorithm to handle large-scale urban scenes. The method leverages the GPU's parallel architecture to process voxel columns independently, optimizing for the flat nature of urban environments [S1, unofficial].

## Sources
[S1] GPU-based Scalable Volumetric Reconstruction for Multi-view Stereo, https://digital.library.unt.edu/ark:/67531/metadc831231/?q=gpu volume rendering, fetched 2026-10-28 (unofficial)
