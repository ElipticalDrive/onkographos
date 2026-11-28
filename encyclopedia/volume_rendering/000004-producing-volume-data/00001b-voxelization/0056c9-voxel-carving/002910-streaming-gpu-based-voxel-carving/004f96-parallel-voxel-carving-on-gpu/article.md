# Parallel Voxel Carving on GPU


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
    *   `2`: "surface" (determined to be on the object boundary) [S1, unofficial].
*   **Surface Voxel Layout (8 bytes):**
    *   Bytes 0-3: Plane quantities $(A, B, C, D)$, each quantized to a signed 1-byte integer [S1, unofficial].
        *   $A, B, C$: Normal vector components.
        *   $D$: Normalized distance, defined as the ratio of the signed distance between the plane and the voxel center to the half-diagonal of the voxel cube [S1, unofficial].
        *   Range: $-127$ to $+127$.
        *   Value $-128$: Indicates no intersection; the voxel is removed [S1, unofficial].
    *   Bytes 4-7: Color $(R, G, B, A)$.
        *   $R, G, B$: Averaged color components.
        *   $A$: Alpha component, used to store the voxel mode (e.g., mapping "surface" to a specific alpha value) [S1, unofficial].

**Algorithm Steps**
1.  **Initialization:** Tag all voxels in the current volume kernel as "uncarved" [S1, unofficial].
2.  **Thread Assignment:** Launch $V^2$ threads (e.g., $256^2$ threads for a $256^3$ volume) [S1, unofficial]. Each thread is assigned a column of voxels along the y-axis [S1, unofficial]. This balances workload because urban scenes are typically flat on the x-z plane [S1, unofficial].
3.  **Carving Loop:** For each voxel in the assigned y-axis column:
    a.  If the voxel is already tagged "carved", skip to the next voxel [S1, unofficial].
    b.  Iterate through all input views $i = 0 \dots N-1$:
        i.  Project the voxel center onto the image plane of view $i$ to find the corresponding pixel coordinates [S1, unofficial].
        ii. Retrieve the depth value $d_{view}$ from the depth map at these coordinates [S1, unofficial].
        iii. Calculate the Euclidean distance $d_{voxel}$ from the voxel center to the camera position of view $i$ [S1, unofficial].
        iv. If $d_{voxel} < d_{view}$, mark the voxel as "carved" and break the view loop (the voxel is occluded or outside) [S1, unofficial].
        v.  If $d_{voxel} > d_{view}$, the voxel is preserved in this view; continue to the next view [S1, unofficial].
4.  **Surface Tagging:** After checking all views, if the voxel is still "uncarved":
    a.  Check if the voxel is within a specified threshold distance of the depth map in any view [S1, unofficial].
    b.  If yes, tag the voxel as "surface" [S1, unofficial].
5.  **Color and Plane Computation (for Surface Voxels):**
    a.  **Color:** For each view where the voxel is visible, bilinearly interpolate the color from the neighboring pixels in the color image [S1, unofficial]. Average these interpolated colors using weights based on the angle between the surface normal and the viewing direction (as per Debevec et al. [6]) [S1, unofficial].
    b.  **Plane:** If multiple planes intersect the voxel cube, compute the normal $(A, B, C)$ and distance $D$ as weighted averages of the intersecting planes, similar to signed distance and weight functions [3] (Note: Source [3] is not provided in the citation list, so the exact formula is not given) [S1, unofficial].
    c.  **Quantization:** Quantize $A, B, C, D$ to signed 1-byte integers in the range $[-127, 127]$ [S1, unofficial]. Set to $-128$ if no intersection [S1, unofficial].
    d.  Store the quantized plane and averaged color in the 8-byte structure [S1, unofficial].
6.  **Post-Processing (Optional):** For point-based rendering (e.g., QSplat), shift the center of each surface voxel to the nearest point on its weighted average plane [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_voxel_carving(V, views, threshold):
    # V: volume dimension (e.g., 256)
    # views: list of view dicts with 'camera_pos', 'depth_map', 'color_map', 'proj_matrix'
    # threshold: max distance to depth map for surface tagging
    
    # Initialize voxel tags: 0=uncarved, 1=carved, 2=surface
    tags = [[[0 for _ in range(V)] for _ in range(V)] for _ in range(V)]
    surface_voxels = []

    def thread_work(x, z):
        for y in range(V):
            if tags[x][y][z] == 1:
                continue
            
            is_carved = False
            for view in views:
                # Project voxel center to image plane (standard perspective projection, formula not given in sources)
                u, v = perspective_project(view['proj_matrix'], view['camera_pos'], (x, y, z))
                
                if not (0 <= u < view['width'] and 0 <= v < view['height']):
                    continue
                
                depth_val = view['depth_map'][v][u]
                dist_to_cam = ((view['camera_pos'][0]-x)**2 + (view['camera_pos'][1]-y)**2 + (view['camera_pos'][2]-z)**2)**0.5
                
                if dist_to_cam < depth_val:
                    tags[x][y][z] = 1
                    is_carved = True
                    break
            
            if not is_carved and tags[x][y][z] == 0:
                is_surface = False
                for view in views:
                    u, v = perspective_project(view['proj_matrix'], view['camera_pos'], (x, y, z))
                    if 0 <= u < view['width'] and 0 <= v < view['height']:
                        depth_val = view['depth_map'][v][u]
                        dist_to_cam = ((view['camera_pos'][0]-x)**2 + (view['camera_pos'][1]-y)**2 + (view['camera_pos'][2]-z)**2)**0.5
                        if abs(dist_to_cam - depth_val) <= threshold:
                            is_surface = True
                            break
                
                if is_surface:
                    tags[x][y][z] = 2
                    
                    avg_color = compute_color((x, y, z), views)
                    plane = compute_plane((x, y, z), views)
                    
                    A, B, C, D = quantize(plane)
                    
                    surface_voxels.append({
                        'pos': (x, y, z),
                        'A': A, 'B': B, 'C': C, 'D': D,
                        'R': avg_color[0], 'G': avg_color[1],
                        'B': avg_color[2], 'A': 2
                    })

    # Conceptual thread launch
    for x in range(V):
        for z in range(V):
            thread_work(x, z)
            
    return surface_voxels

def perspective_project(proj_matrix, cam_pos, voxel_center):
    # Standard homogeneous coordinate transformation to image plane
    # Exact formula not given in sources
    return (0.0, 0.0)

def compute_color(voxel_pos, views):
    colors = []
    for view in views:
        u, v = perspective_project(view['proj_matrix'], view['camera_pos'], voxel_pos)
        if 0 <= u < view['width'] and 0 <= v < view['height']:
            # Bilinear interpolation from color map (standard implementation)
            color = bilinear_interpolate(view['color_map'], u, v)
            # Weight by angle between surface normal and viewing direction (Debevec et al. [6])
            # Exact weighting formula not given in sources
            colors.append(color)
    if not colors: return (0, 0, 0, 0)
    return tuple(sum(c[i] for c in colors) / len(colors) for i in range(3)) + (255,)

def bilinear_interpolate(tex, u, v):
    # Standard bilinear interpolation
    # Implementation details not given in sources
    return (0, 0, 0)

def compute_plane(voxel_pos, views):
    # Weighted average of normals and distances from intersecting planes
    # Exact formula not given in sources
    return (0.0, 0.0, 0.0, 0.0)

def quantize(plane):
    # Quantize to signed 1-byte, range [-127, 127]
    # -128 indicates no intersection
    return tuple(max(-127, min(127, int(v * 127))) if v is not None else -128 for v in plane)
```

**Parameters**
*   **Volume Size per Kernel:** Approximately $256^3$ voxels, limited by GPU memory [S1, unofficial].
*   **Thread Count:** $256^2$ threads for a $256^3$ volume, where each thread carves 256 voxels independently [S1, unofficial].
*   **Distance D Definition:** Ratio of the signed distance between the plane and the voxel center to the half-diagonal of the voxel cube [S1, unofficial].
*   **Threshold:** Distance threshold for tagging a voxel as "surface" (Not given in the sources).

**Edge Cases and Pitfalls**
*   **Precision:** Quantization of plane parameters to 1-byte integers may introduce precision errors. Values of $-128$ indicate no intersection and must be handled to avoid rendering artifacts [S1, unofficial].
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
Parallel Voxel Carving on GPU was introduced in the paper "GPU-based Scalable Volumetric Reconstruction for Multi-view Stereo" [S1, unofficial]. It builds on the concept of voxel carving, transforming it into a parallel GPU algorithm to handle large-scale urban scenes [S1, unofficial]. The method leverages the GPU's parallel architecture to process voxel columns independently, optimizing for the flat nature of urban environments [S1, unofficial].

## Sources
[S1] GPU-based Scalable Volumetric Reconstruction for Multi-view Stereo, https://digital.library.unt.edu/ark:/67531/metadc831231/?q=gpu volume rendering, fetched 2026-10-28 (unofficial)
