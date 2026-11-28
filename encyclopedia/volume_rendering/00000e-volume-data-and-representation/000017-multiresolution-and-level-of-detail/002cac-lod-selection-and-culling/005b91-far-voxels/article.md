# Far Voxels


## Overview
Far Voxels is a hierarchical level-of-detail (HLOD) technique for massive model visualization that replaces detailed geometry with coarse voxel representations for distant viewing [S1]. It solves the performance bottleneck of rendering hundreds of millions of triangles by culling, occluding, and simplifying sub-models into out-of-core manageable voxel blocks [S1]. Unlike sibling methods that rely on spatial hierarchies like octrees or frequency-domain transforms, Far Voxels constructs its hierarchy via recursive subdivision based on a surface-area heuristic and uses ray-traced sampling to generate direction-aware shader models for each voxel [S1]. Its primary distinction is the use of environmental occlusion and multi-directional material interpolation to maintain visual quality at a distance, though it suffers from aliasing artifacts on thin objects [S1].

## How it works
The method operates in two distinct phases: Preprocessing (offline) and Visualization (runtime).

### Inputs and Outputs
*   **Input:** A high-resolution triangle mesh (CAD model) containing potentially hundreds of millions of triangles [S1].
*   **Output:** A data file containing:
    1.  HLOD hierarchy traversal information [S1].
    2.  Model representation data: lists of voxels for internal nodes and lists of triangles/triangle-strips for leaf nodes [S1].

### Data Structures

**1. HLOD Hierarchy Node**
Each node in the hierarchy represents a spatial region.
*   **Children:** Two child nodes created by axis-aligned subdivision [S1].
*   **Leaf Status:** Boolean flag indicating if the node contains fewer than a predefined threshold of triangles [S1].
*   **Representation:**
    *   If Internal: A list of voxels.
    *   If Leaf: A list of triangles or triangle-strips [S1].

**2. Voxel Structure**
Voxels are organized on a uniform grid within a node [S1]. Each voxel stores parameters for a shader model. There are two types:
*   **Type 1 (Flat/Almost Flat Surfaces):**
    *   `normal`: Single 3D vector [S1].
    *   `material_1`: Material properties for one viewing direction [S1].
    *   `material_2`: Material properties for the opposite viewing direction [S1].
*   **Type 2 (Complex Surfaces):**
    *   `normals`: Array of 6 3D vectors, corresponding to main viewing directions ($\pm x, \pm y, \pm z$) [S1].
    *   `materials`: Array of 6 material properties, associated with the same 6 directions [S1].

**3. Raytracing Configuration**
*   **Volume $V$:** The 3D region being sampled (the node's spatial extent) [S1].
*   **Surface $S$:** A boundary surface surrounding $V$.
*   **$d_{min}$:** The minimum distance from the viewer to $V$ required to use the voxel representation. $S$ is positioned at distance $d_{min}$ from $V$ [S1].
*   **Occluders:** All objects inside $S$ act as occluders for the raytracer, assuming the viewer is always outside $S$ [S1].

### Algorithm Steps

#### Preprocessing Phase

1.  **Hierarchy Construction:**
    *   Start with all triangles of the model assigned to the root node [S1].
    *   Recursively subdivide nodes using axis-aligned planes positioned according to the surface-area heuristic [S1].
    *   Split existing triangles between the two new child nodes [S1].
    *   Terminate subdivision when a node contains fewer than a predefined number of triangles; mark this node as a leaf [S1].

2.  **Voxel Generation (for Internal Nodes):**
    *   Define the sampling volume $V$ and surface $S$ with distance $d_{min}$ [S1].
    *   Launch a CPU-based raytracer from surface $S$ into volume $V$ [S1].
    *   **Environmental Occlusion:** Discard any voxels in $V$ that are never hit by sampling rays. This assumes the view is always from outside $S$, so hidden surfaces do not need representation [S1]. (Typical discard rate: 25–43%, up to 57% in tested models) [S1].
    *   For all hit voxels, compute radiance/colors from all intersection directions [S1].

3.  **Shader Model Selection:**
    *   Analyze the stored colors/radiance for each hit voxel to determine the appropriate shader model type (Type 1 or Type 2) [S1].
    *   Selection criterion: The model should project to approximately one pixel at the minimum viewing distance [S1].
    *   **Type 1 Selection:** If the surface is flat/almost flat, assign a single normal and two materials [S1].
    *   **Type 2 Selection:** If the surface is complex, assign 6 normals and 6 materials corresponding to $\pm x, \pm y, \pm z$ [S1].

4.  **Interpolation Preparation:**
    *   For voxels using Type 2 models, prepare for intermediate viewing directions by storing normals and materials for the 6 cardinal directions [S1].
    *   The runtime shader will interpolate these values based on direction cosines for non-cardinal views [S1].

5.  **Storage:**
    *   Write the HLOD hierarchy and voxel/triangle data to disk [S1].

#### Visualization Phase

1.  **Hierarchy Traversal:**
    *   Traverse the HLOD hierarchy starting from the root [S1].
    *   For each node, decide whether to render its voxel representation or traverse its children for higher precision [S1].
    *   **Decision Criterion:** Continue descending until the projected size of the voxels is smaller than a user-defined threshold or a leaf node is reached [S1].

2.  **Occlusion Culling:**
    *   Test branches of the hierarchy for occlusion using Hardware Occlusion Queries [S1].
    *   Skip drawing hidden parts of the model [S1].

3.  **Rendering:**
    *   For each voxel node, bind the appropriate vertex shader (Type 1 or Type 2) [S1].
    *   Pass additional parameters (normals, materials) via VertexAttributes [S1].
    *   Draw all voxels in the node with a single call to `glDrawArrays` [S1].
    *   For leaf nodes, render the stored triangles/triangle-strips [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing Phase

def build_hlod_hierarchy(triangles):
    """
    Recursively subdivide triangles into an HLOD tree.
    """
    root = Node(triangles=triangles)
    queue = [root]
    
    while queue:
        node = queue.pop(0)
        
        # Check leaf condition
        if len(node.triangles) < PREDEFINED_TRIANGLE_THRESHOLD:
            node.is_leaf = True
            continue
            
        # Subdivide using surface-area heuristic
        plane = compute_surface_area_heuristic_plane(node.bounding_box)
        left_triangles, right_triangles = split_triangles(node.triangles, plane)
        
        left_child = Node(triangles=left_triangles, parent=node)
        right_child = Node(triangles=right_triangles, parent=node)
        
        node.children = [left_child, right_child]
        node.is_leaf = False
        
        queue.append(left_child)
        queue.append(right_child)
        
    return root

def generate_voxel_representation(node):
    """
    Generate voxel data for an internal node using raytracing.
    """
    volume_V = node.bounding_box
    d_min = compute_minimum_viewing_distance(node)
    surface_S = create_boundary_surface(volume_V, d_min)
    
    # Initialize voxel grid for volume_V
    voxel_grid = create_uniform_grid(volume_V)
    
    # Raytracing from surface_S into volume_V
    # Objects inside S act as occluders
    rays = generate_rays_from_surface(surface_S, density=HIGH)
    
    hit_voxels = set()
    voxel_radiance_data = {}
    
    for ray in rays:
        intersection = raytrace(ray, scene_objects_inside_S)
        if intersection:
            voxel_id = get_voxel_id(intersection.point, voxel_grid)
            hit_voxels.add(voxel_id)
            if voxel_id not in voxel_radiance_data:
                voxel_radiance_data[voxel_id] = []
            voxel_radiance_data[voxel_id].append(intersection.color)
            
    # Environmental Occlusion: discard non-hit voxels
    for voxel_id in voxel_grid.keys():
        if voxel_id not in hit_voxels:
            voxel_grid.pop(voxel_id)
            
    # Shader Model Selection
    for voxel_id, colors in voxel_radiance_data.items():
        voxel = voxel_grid[voxel_id]
        
        # Analyze colors to determine surface complexity
        if is_flat_surface(colors):
            # Type 1: Flat/Almost Flat
            voxel.normal = compute_average_normal(colors)
            voxel.material_1 = compute_material(colors, direction_1)
            voxel.material_2 = compute_material(colors, direction_2)
            voxel.shader_type = 1
        else:
            # Type 2: Complex
            voxel.normals = [compute_normal_for_direction(colors, dir) for dir in CARDINAL_DIRECTIONS]
            voxel.materials = [compute_material_for_direction(colors, dir) for dir in CARDINAL_DIRECTIONS]
            voxel.shader_type = 2
            
    return voxel_grid

# Visualization Phase

def render_hlod(node, camera, screen_threshold):
    """
    Traverse HLOD and render.
    """
    # Occlusion Culling
    if not is_visible(node.bounding_box, camera):
        return
        
    # LOD Decision
    projected_size = compute_projected_size(node.bounding_box, camera)
    
    if node.is_leaf or projected_size < screen_threshold:
        if node.is_leaf:
            render_triangles(node.triangles, camera)
        else:
            render_voxels(node.voxels, camera)
    else:
        for child in node.children:
            render_hlod(child, camera, screen_threshold)

def render_voxels(voxels, camera):
    """
    Render voxels using appropriate shader.
    """
    # Group voxels by shader type for single draw calls
    type_1_voxels = [v for v in voxels if v.shader_type == 1]
    type_2_voxels = [v for v in voxels if v.shader_type == 2]
    
    if type_1_voxels:
        bind_shader(ShaderType1)
        set_vertex_attributes(type_1_voxels)
        glDrawArrays(GL_POINTS, 0, len(type_1_voxels))
        
    if type_2_voxels:
        bind_shader(ShaderType2)
        set_vertex_attributes(type_2_voxels)
        glDrawArrays(GL_POINTS, 0, len(type_2_voxels))
```

### Parameters and Typical Values
*   **Predefined Triangle Threshold:** Not given in the sources.
*   **$d_{min}$:** Not given in the sources.
*   **Screen Threshold:** User-defined parameter indicating the projected size of voxels [S1].
*   **Ray Density:** High density used for sampling [S1].
*   **Occlusion Discard Rate:** 25% to 43% typically, up to 57% in tested models [S1].

### Edge Cases and Pitfalls
*   **Aliasing on Thin Objects:** The original Far Voxels algorithm suffers from severe aliasing when representing small or thin objects. Voxels create representations larger than the original geometry, causing "popping" artifacts during LOD transitions [S1].
*   **Temporal Aliasing:** The distortion of thin objects is aggravated during navigation, causing them to appear to move between consecutive frames [S1].
*   **Hidden Surfaces:** If the assumption that the viewer is always outside surface $S$ is violated, hidden surfaces may incorrectly influence the voxel appearance or be incorrectly culled [S1].

## Subtopics

### [Flat blocking multi-resolution volume rendering](6428-flat-blocking-multi-resolution-volume-rendering.md)
A strategy dividing volume into sub-blocks of constant spatial size but varying resolution. It differs from Far Voxels by maintaining fixed block dimensions where lower resolution blocks contain fewer samples, achieving higher data reduction ratios (3.8:1) compared to hierarchical methods.

### [Hierarchical Chunk LOD System](17123-hierarchical-chunk-lod-system.md)
A recursive LOD system using octree-like chunk subdivision. It differs from Far Voxels by using a dynamic, recursive tree structure for consistent surface approximation, whereas Far Voxels uses a fixed hierarchy with ray-traced voxel generation.

## Operating characteristics
*   **Time Complexity:** Preprocessing is dominated by CPU-based raytracing and hierarchy construction via surface-area heuristic. Visualization traversal is logarithmic with respect to the hierarchy depth, with rendering cost proportional to the number of visible voxels/triangles [S1].
*   **Memory Complexity:** Depends on the number of voxels stored for internal nodes and triangles for leaf nodes. Environmental occlusion reduces voxel count by 25–57% [S1].
*   **Measured Performance:** Yields interactive rates for large datasets (hundreds of millions of triangles) by handling levels of detail, culling, and out-of-core storage [S1].
*   **Strengths:** Effective for massive CAD models; reduces data size via occlusion; supports out-of-core storage; single draw calls for voxels improve GPU efficiency [S1].
*   **Limitations:** Severe aliasing artifacts on thin/small objects during LOD transitions; requires CPU-based raytracing for preprocessing; assumes viewer is always outside the sampling volume [S1].
*   **When to Use:** Visualization of large-scale CAD models where interactive frame rates are required and objects are generally large/complex.
*   **When Not to Use:** Scenes with many thin, small, or delicate geometric features where aliasing artifacts are unacceptable [S1].

## Background
The Far Voxels algorithm was introduced in the context of visualizing massive Computer Aided Design (CAD) models, which can contain hundreds of millions of triangles [S1]. It was developed to address the problems of model size and visualization quality in engineering activities [S1]. The method builds on the concept of using voxels to create coarse representations of sub-models, leveraging hierarchical level-of-detail (HLOD) structures [S1]. The specific anti-aliasing improvements and detailed shader model selection described in the sources were presented in a paper by G.N. Wagner, A. Raposo, and M. Gattass [S1].

## Sources
[S1] An Anti-aliasing Technique for Voxel-Based Massive Model Visualization Strategies, https://doi.org/10.1007/978-3-540-76858-6_29, fetched 2026-10-27
