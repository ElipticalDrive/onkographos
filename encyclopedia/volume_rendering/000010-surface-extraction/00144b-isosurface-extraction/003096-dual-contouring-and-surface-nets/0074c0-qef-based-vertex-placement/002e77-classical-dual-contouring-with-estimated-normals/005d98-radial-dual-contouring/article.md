# Radial Dual Contouring

Also called: RadialDC

## Overview
Radial Dual Contouring (RadialDC) is a method for large-scale world mesh generation that extends Classical Dual Contouring with Dynamic Voxel Sizing and Dynamic Voxel Density Evaluation [S1, unofficial]. It solves the performance bottleneck of generating massive, high-detail terrains by adapting voxel resolution and density function complexity based on the distance from a central player position [S1, unofficial]. Unlike standard Dual Contouring, which uses a uniform grid, RadialDC ensures a crack-free mesh by generating voxels radially outward from a high-detail center, allowing for seamless regeneration as the player moves [S1, unofficial].

## How it works
RadialDC builds upon the vertex placement and quad-mesh generation of Classical Dual Contouring [S1, unofficial]. It introduces a radial hierarchy of voxels and a multi-threaded execution pipeline to achieve real-time generation of large volumes (e.g., a 4-mile cube in 1-2 seconds) [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Player position ($P$) [S1, unofficial].
*   World center ($C$) for the current mesh chunk [S1, unofficial].
*   Configuration parameters: $V_{min}$ (smallest voxel size), $V_{max}$ (largest voxel size limit), $R_{regen}$ (regeneration threshold), and tier specifications [S1, unofficial].
*   Density function definitions (e.g., Simplex noise octaves, Recipe Blocks) [S1, unofficial].

**Outputs:**
*   A `Mesh` object containing position, normal, UV, and triangle arrays [S1, unofficial].
*   A boolean flag indicating completion status [S1, unofficial].

### Data Structures
The world is divided into two primary voxel categories [S1, unofficial]:
1.  **Core Voxels:** Voxels making up several tiers of the highest detail [S1, unofficial].
2.  **Outer Voxels:** Voxels that expand in size as distance from the center increases, up to $V_{max}$ [S1, unofficial].

Each voxel $V_i$ is defined by its spatial bounds and a detail level $L_i$ [S1, unofficial]. The detail level determines the complexity of the density function evaluation [S1, unofficial].

### Algorithm Steps

#### 1. Setup and Initialization
Initialize the system with $V_{min}$, tier sizes, and threading control flags [S1, unofficial].

#### 2. Regeneration Trigger Check
Determine if a new mesh center is required [S1, unofficial]. Let $D$ be the distance from the player $P$ to the current center $C$. Let $R_{max}$ be the radius of the high-detail area. If $D > \frac{1}{2} R_{max}$, a regeneration is triggered [S1, unofficial].

#### 3. Recentering
If regeneration is triggered, find a new center $C'$ [S1, unofficial]. Define a set of candidate recenter points (e.g., a $3 \times 3 \times 3$ grid of voxel centers surrounding the current region) [G]. Calculate the distance from $P$ to each candidate [G]. Select the candidate $C'$ with the minimum distance to $P$ [S1, unofficial].

#### 4. Voxel Generation (Step 1)
Generate the voxel grid radially from $C'$ [S1, unofficial].
*   **Core Region:** Create voxels with size $V_{min}$ for the first $T_{core}$ tiers [S1, unofficial].
*   **Outer Region:** For tiers beyond $T_{core}$, increase voxel size with distance from $C'$ until $V_{max}$ is reached [S1, unofficial].
*   **Crack-Free Rule:** Ensure adjacent voxels share boundary vertices; the radial generation order and strict rules prevent gaps and duplication [S1, unofficial].

#### 5. Density Evaluation (Step 2)
For each voxel $V_i$, evaluate the density function $f(x)$ to find the isosurface intersection and normal [S1, unofficial].
*   **Dynamic Density Evaluation:** The complexity of $f(x)$ depends on the voxel's detail level $L_i$ [S1, unofficial].
    *   High $L_i$ (Core): Use multiple octaves of Simplex noise or complex Recipe Blocks [S1, unofficial].
    *   Low $L_i$ (Outer): Use simpler functions or fewer noise octaves [S1, unofficial].
*   **Parallel Execution:** Each voxel's evaluation is independent and executed on a separate thread [S1, unofficial].

#### 6. Mesh Data Generation (Step 3)
Using the vertices and normals computed in Step 2, generate the triangle arrays [S1, unofficial]. This step aggregates the per-voxel data into a single mesh structure [S1, unofficial].

#### 7. Mesh Assembly
Create the final `Mesh` object from the position, normal, UV, and triangle arrays. Return the mesh pointer and a completion boolean [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import threading

def radial_dc(player_pos, current_center, config):
    """
    Main entry point for Radial Dual Contouring.
    Returns (mesh, is_complete).
    """
    dist_to_center = math.dist(player_pos, current_center)
    threshold = config.regen_radius / 2.0
    
    new_center = current_center
    regenerate = False
    
    if dist_to_center > threshold:
        new_center = find_closest_recenter_point(player_pos, config.candidate_points)
        regenerate = True
        
    if regenerate:
        current_center = new_center
        return generate_mesh_pipeline(current_center, player_pos, config)
    else:
        return config.existing_mesh, True

def find_closest_recenter_point(player_pos, candidate_points):
    min_dist = float('inf')
    best_point = candidate_points[0]
    for point in candidate_points:
        d = math.dist(player_pos, point)
        if d < min_dist:
            min_dist = d
            best_point = point
    return best_point

def generate_mesh_pipeline(center, player_pos, config):
    # Stage 1: Voxel Creation (Single Core)
    voxels = create_radial_voxels(center, config.min_voxel_size, config.max_voxel_size, 
                                  config.core_tiers, config.max_layers, config.expansion_rate)
    
    # Stage 2: Density Evaluation (Multi-threaded)
    voxel_results = []
    def eval_thread(v):
        voxel_results.append(evaluate_voxel(v, config.density_functions))
    threads = [threading.Thread(target=eval_thread, args=(v,)) for v in voxels]
    for t in threads: t.start()
    for t in threads: t.join()
    
    # Stage 3: Mesh Data Generation (Single Core)
    mesh_data = assemble_mesh_data(voxel_results)
    
    # Final Assembly
    mesh = create_mesh_object(mesh_data.positions, mesh_data.normals, mesh_data.uvs, mesh_data.triangles)
    return mesh, True

def evaluate_voxel(voxel, density_functions):
    detail_level = voxel.detail_level
    if detail_level > config.core_tiers:
        func = density_functions.simple
    else:
        func = density_functions.complex
    # Perform Dual Contouring steps: sample density, compute normals, solve QEF, generate quad mesh
    return voxel

def create_radial_voxels(center, min_size, max_size, core_tiers, max_layers, expansion_rate):
    voxels = []
    for layer in range(max_layers):
        if layer <= core_tiers:
            size = min_size
        else:
            size = min_size + (layer - core_tiers) * expansion_rate
            if size > max_size:
                size = max_size
        layer_voxels = generate_layer_voxels(center, size, layer)
        voxels.extend(layer_voxels)
    return voxels

def generate_layer_voxels(center, size, layer):
    # Standard grid iteration to create voxels at given size and layer
    return []

def assemble_mesh_data(voxel_results):
    positions, normals, uvs, triangles = [], [], [], []
    for v in voxel_results:
        positions.extend(v.positions)
        normals.extend(v.normals)
        uvs.extend(v.uvs)
        triangles.extend(v.triangles)
    return type('MeshData', (), {'positions': positions, 'normals': normals, 'uvs': uvs, 'triangles': triangles})()

def create_mesh_object(positions, normals, uvs, triangles):
    return type('Mesh', (), {'positions': positions, 'normals': normals, 'uvs': uvs, 'triangles': triangles})()
```

### Edge Cases and Pitfalls
*   **Regeneration Timing:** Regeneration must occur off the main thread and before the player leaves the high-detail area to avoid loading screens or visual popping [S1, unofficial].
*   **Crack Prevention:** The radial generation order and strict rules for voxel sizing are critical. If outer voxels are not sized correctly relative to their neighbors, cracks will appear at boundaries [S1, unofficial].
*   **Unity Main Thread:** The implementation uses a "flow gate" style because Unity requires the main thread to run continuously. The main thread checks and advances the process once per frame [S1, unofficial].

## Operating characteristics
*   **Performance:** Capable of generating a roughly 4-mile cube in 1-2 seconds [S1, unofficial].
*   **Complexity:**
    *   Time: $O(N \cdot C)$, where $N$ is the number of voxels and $C$ is the complexity of the density function for that voxel. Parallelization reduces wall-clock time significantly for Step 2 [S1, unofficial].
    *   Memory: Proportional to the number of voxels and the size of the mesh data arrays [S1, unofficial].
*   **Strengths:**
    *   Handles extremely large worlds efficiently [S1, unofficial].
    *   Seamless regeneration as the player moves [S1, unofficial].
    *   Adaptive detail reduces computational load in distant areas [S1, unofficial].
*   **Limitations:**
    *   Requires careful tuning of voxel sizing rules to prevent cracks [S1, unofficial].
    *   Implementation complexity is high due to multi-threading and radial logic [S1, unofficial].
    *   Dependent on the specific "Recipe Block" abstraction for density definition [S1, unofficial].

## Background
Radial Dual Contouring was introduced by James Pardue as an extension of the Dual Contouring algorithm [S1, unofficial]. It was developed to address the limitations of standard Dual Contouring in large-scale world generation, specifically the inability to efficiently handle varying levels of detail across a massive grid. The method is implemented in the `RadialDualContouring` repository, often in the context of the Unity engine [S1, unofficial].

## Sources
[S1] JamesPardue/RadialDualContouring, https://github.com/JamesPardue/RadialDualContouring, fetched 2026-10-26 (unofficial)
