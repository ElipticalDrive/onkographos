# Dynamic Update Optimization


## Overview
Dynamic Update Optimization refers to a set of techniques for accelerating the voxelization and indirect illumination calculation of dynamic geometry within a hierarchical volume structure [S1, unofficial]. It solves the performance bottleneck of re-processing the entire scene every frame by isolating static and dynamic components, allowing the system to re-voxelize only dynamic objects and update only the affected nodes in the octree [S1, unofficial]. Unlike sibling methods that focus on memory layout or update scheduling, this approach specifically optimizes the computational pipeline for real-time dynamic scenes, including specialized atomic operations for normals and frustum culling for lighting injection [S1, unofficial].

## How it works
The method operates on a hybrid static/dynamic octree structure [S1, unofficial]. Static geometry is voxelized once at initialization and stored in the octree, marked with a bit flag to prevent overwriting [S1, unofficial]. Dynamic geometry is voxelized every frame and appended to the end of the octree node buffer, which can be cleared each frame [S1, unofficial].

### Inputs and Outputs
- **Inputs**: Scene geometry (static and dynamic), camera position, camera frustum, maximum cone tracing distance, voxel size [S1, unofficial].
- **Outputs**: Updated octree node buffer containing static nodes (unchanged) and dynamic voxel fragments; updated indirect illumination values for affected nodes [S1, unofficial].

### Data Structures
The octree node buffer stores nodes sequentially [S1, unofficial]. Each node contains:
- A bit flag indicating if the node is static or dynamic [S1, unofficial].
- Normal data (stored with a 1-byte counter for dynamic nodes to support averaging or max operations) [S1, unofficial].
- Diffuse color data, computed via `InterlockedMax()` for dynamic geometry [S1, unofficial].
- Neighbor offsets pointing to adjacent nodes, including those in the dynamic buffer [S1, unofficial].

### Steps
1. **Initialization (Static)**:
   - Voxelize all static geometry [S1, unofficial].
   - Perform the full indirect illumination calculation pipeline for all static voxels [S1, unofficial].
   - Store static nodes in the octree with the static bit flag set [S1, unofficial].
2. **Dynamic Voxelization (Per Frame)**:
   - Voxelize only dynamic geometry [S1, unofficial].
   - Use `InterlockedMax()` to compute diffuse and normal values for octree voxels, which is faster than atomic average [S1, unofficial].
   - **Pitfall**: Using `InterlockedMax()` for normals reduces lighting quality in small voxel volumes (e.g., 256×256×256), causing artifacts where light reflects incorrectly compared to average normals [S1, unofficial].
   - **Mitigation**: Use `InterlockedMax()` for normals only in dynamic geometry; keep static geometry using average normals [S1, unofficial]. This necessitates storing a 1-byte counter with the normal in dynamic nodes [S1, unofficial].
   - Append dynamic voxel fragments to the end of the octree node buffer [S1, unofficial].
3. **Neighbor Reset and Affected Node Identification**:
   - Reset static octree node neighbor offsets that point to dynamic nodes [S1, unofficial].
   - Identify static nodes affected by dynamic voxels from the previous and current frames [S1, unofficial].
   - Dispatch threads for all nodes to queue affected node indices into a buffer [S1, unofficial].
4. **Indirect Illumination Update (Per Frame)**:
   - Re-calculate all steps of the indirect illumination pipeline only for the queued affected nodes [S1, unofficial].
   - Only changed values are recalculated [S1, unofficial].
5. **Direct Lighting Injection with Frustum Culling**:
   - Perform view-frustum culling when injecting direct lighting into the octree [S1, unofficial].
   - Calculate an extended frustum by moving the camera backward and increasing the far plane using the same field of view [S1, unofficial].
   - The extended distance is calculated as:
     $$D_{ext} = D_{cone} + S_{voxel}$$
     where $D_{cone}$ is the maximum cone tracing distance and $S_{voxel}$ is the filtered voxel size [S1, unofficial].
   - Light far from this extended frustum is not filtered into the octree [S1, unofficial].
6. **Cone Tracing Pass Optimization**:
   - Perform the cone tracing pass at half screen resolution [S1, unofficial].
   - **Artifact**: Visible artifacts appear when up-scaling to full resolution, especially at geometry edges [S1, unofficial].
   - **Mitigation**: Blur the pixels with artifacts by averaging with neighbor pixels. This is faster than re-computing cone tracing at full resolution for edge pixels but yields lower quality [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
def init_static_geometry(scene):
    octree = []
    for obj in scene.static_geometry:
        for voxel in voxelize(obj):
            node = {'is_static': True, 'normal': [0.0, 0.0, 0.0], 'diffuse': [0.0, 0.0, 0.0], 'counter': 0}
            node['normal'], node['diffuse'] = compute_average_normal_and_diffuse(node, voxel)
            node['indirect_illumination'] = calculate_indirect_illumination(node)
            octree.append(node)
    return octree

# Per Frame Update
def update_dynamic_geometry(octree, scene, camera, max_cone_distance, voxel_size):
    dynamic_buffer = []
    
    # 1. Voxelize Dynamic Geometry
    for obj in scene.dynamic_geometry:
        for voxel in voxelize(obj):
            # Use InterlockedMax for speed, accepting quality trade-off for normals
            voxel.normal = interlocked_max(voxel.normal, compute_normal(voxel))
            voxel.diffuse = interlocked_max(voxel.diffuse, compute_diffuse(voxel))
            voxel.is_static = False
            voxel.counter = 1
            dynamic_buffer.append(voxel)
    
    # Append to octree buffer
    octree.extend(dynamic_buffer)
    
    # 2. Reset Neighbors and Find Affected Nodes
    affected_nodes = []
    for node in octree:
        if node['is_static'] and node.has_dynamic_neighbor():
            affected_nodes.append(node)
            node.reset_dynamic_neighbor_offsets()
            
    # 3. Update Indirect Illumination for Affected Nodes
    for node in affected_nodes:
        node['indirect_illumination'] = calculate_indirect_illumination(node)
        
    # 4. Inject Direct Lighting with Frustum Culling
    extended_far = calculate_extended_far_plane(camera, max_cone_distance)
    extended_frustum = create_frustum(camera, extended_far)
    
    for node in octree:
        if is_inside_frustum(node.position, extended_frustum):
            node.direct_light = inject_direct_light(node, scene.lights)
            
    # 5. Cone Tracing Pass (Half Resolution)
    half_res_buffer = render_cone_tracing_at_half_res(octree, camera)
    full_res_buffer = upsample_and_blur_artifacts(half_res_buffer)
    
    return full_res_buffer

# Helper definitions
def voxelize(geometry): return rasterize_to_voxels(geometry) # Standard GPU rasterization
def compute_average_normal_and_diffuse(node, voxel): return atomic_average(node.normal, voxel.normal), atomic_average(node.diffuse, voxel.diffuse) # Standard atomic reduction
def calculate_indirect_illumination(node): return run_5_step_pipeline(node) # Standard indirect lighting calculation
def interlocked_max(a, b): return max(a, b) # GPU atomic maximum operation
def compute_normal(voxel): return normalize(cross(voxel.edge1, voxel.edge2)) # Standard surface normal
def compute_diffuse(voxel): return voxel.color * voxel.roughness # Standard diffuse reflectance
def has_dynamic_neighbor(node): return any(offset > 0 for offset in node.neighbor_offsets) # Check offset flags
def reset_dynamic_neighbor_offsets(node): node.neighbor_offsets = [0] * 8 # Clear dynamic pointers
def calculate_extended_far_plane(camera, max_cone_distance): return camera.far_plane + max_cone_distance # Extend frustum
def create_frustum(camera, far_plane): return generate_frustum_planes(camera, far_plane) # Standard frustum generation
def is_inside_frustum(position, frustum): return all(dot(position, plane.normal) <= plane.d for plane in frustum.planes) # Standard point-in-frustum test
def inject_direct_light(node, lights): return sum(light.intensity * attenuation(node, light) for light in lights) # Standard direct lighting sum
def render_cone_tracing_at_half_res(octree, camera): return downsample_and_trace(octree, camera, scale=0.5) # Half-resolution cone tracing
def upsample_and_blur_artifacts(buffer): return bilinear_upsample(buffer) + average_neighbor_blur(buffer) # Bilinear upsample with neighbor averaging
```

## Subtopics

### [Priority Queue Based Field Update](7430-priority-queue-based-field-update.md)
A scheduling mechanism for hierarchical volume data structures that ensures field modifications are applied from coarser levels to finer levels. It solves the problem of visual inconsistency in interactive volume sculpting, where a depth-first recursive update would modify fine details before their coarser parents are adjusted. Unlike sibling methods that handle memory management or texture uploads, this method specifically manages the temporal and spatial order of geometric updates using a priority queue of cell queues.

### [Sparse Voxel Octree Memory Management and Texture Upload](11147-sparse-voxel-octree-memory-management-and-texture-upload.md)
This method manages the memory layout of a sparse voxel octree in a flat integer array and optimizes GPU texture uploads by tracking changes per 256×256 slice. It solves the problem of inefficient full-texture updates for dynamic or sparse data by only transferring modified slices to the GPU. It differs from sibling methods like [Priority Queue Based Field Update](7430-priority-queue-based-field-update.md) by focusing on memory compaction and incremental texture synchronization rather than field update ordering.

### [Coarse-to-Fine Octree Update Strategy](15715-coarse-to-fine-octree-update-strategy.md)
This strategy accelerates sparse voxel octree (SVO) updating and reduces noise sensitivity by decoupling structural changes from density optimization. It initially updates the octree topology using low-resolution (down-sampled) projections to mitigate high-frequency noise, then fixes the tree structure to optimize only the density grids using original high-resolution projections. It differs from [Priority Queue Based Field Update](7430-priority-queue-based-field-update.md) by using resolution scaling rather than priority metrics to guide updates, and from [Wavemap Recursive Update](18498-wavemap-recursive-update.md) by explicitly separating topology convergence from density refinement.

### [Wavemap Recursive Update](18498-wavemap-recursive-update.md)
The Wavemap Recursive Update is an algorithm for updating a volumetric map posterior given a current measurement, utilizing an octree structure with adaptive resolution and early termination based on approximation error. It distinguishes itself from sibling methods by combining wavelet-based compression with a recursive traversal that prunes branches based on sensor likelihood and approximation bounds, rather than relying solely on priority queues or static memory management.

## Operating characteristics
- **Time Complexity**: Reduces per-frame cost by re-voxelizing only dynamic objects and recalculating only affected nodes, avoiding full scene re-processing [S1, unofficial].
- **Memory Complexity**: Requires additional buffer space for dynamic voxel fragments appended to the octree node buffer and a 1-byte counter per dynamic normal [S1, unofficial].
- **Measured Performance**: In the source demo, dynamic updates allow real-time frame rates by avoiding full scene re-voxelization [S1, unofficial]. The half-resolution cone tracing with blur mitigation provides a significant speedup over full-resolution edge-aware re-computation, though at the cost of visual quality at geometry edges [S1, unofficial].
- **Strengths**: Enables real-time indirect illumination for dynamic scenes; efficient handling of sparse dynamic changes; reduced noise in high-noise volumetric data via coarse-to-fine strategies [S1, unofficial][S2].
- **Limitations**: `InterlockedMax()` for normals causes lighting artifacts in small voxel volumes [S1, unofficial]; half-resolution cone tracing introduces up-scaling artifacts at geometry edges [S1, unofficial]; mixed-integer programming for SD-based updates may be computationally expensive for large octrees [S2].
- **When to Use**: Real-time rendering with dynamic geometry; cryo-ET reconstruction with high noise; open-world games with local modifications [S1, unofficial][S2, S3].
- **When Not to Use**: Scenes with fully static geometry (use static-only optimization); scenes requiring high-fidelity normal-based lighting in small voxels (avoid `InterlockedMax()` for normals) [S1, unofficial].

## Background
The concept of dynamic update optimization in voxel cone tracing was introduced in a blog series on implementing voxel cone tracing [S1, unofficial]. The coarse-to-fine and standard deviation-based update strategies were developed for cryo-electron tomography reconstruction by researchers publishing in *Bioinformatics Advances* [S2]. The hybrid dense array and SVDAG modification was proposed in the Aokana framework for GPU-driven voxel rendering in open-world games [S3].

## Sources
[S1] Implementing Voxel Cone Tracing, http://simonstechblog.blogspot.com/2013/01/implementing-voxel-cone-tracing.html, fetched 2026-10-27 (unofficial)
[S2] Adaptive differentiable grids for cryo-electron tomography reconstruction and denoising, https://doi.org/10.1093/bioadv/vbad131, fetched 2026-10-27
[S3] Aokana: A GPU-Driven Voxel Rendering Framework for Open World Games, https://doi.org/10.1145/3728299, fetched 2026-10-26
