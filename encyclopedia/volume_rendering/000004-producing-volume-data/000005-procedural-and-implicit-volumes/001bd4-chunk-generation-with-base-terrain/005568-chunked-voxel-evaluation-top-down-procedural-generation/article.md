# Chunked Voxel Evaluation (Top-Down Procedural Generation)

Also called: Chunked Evaluation, Top-Down Procedural Generation

## Overview
This method generates voxel data on-demand by evaluating procedural object definitions within GPU shaders, driven by high-level parameters computed on the CPU. It solves the problem of rendering large, continuous procedural structures (such as trees or buildings) that span multiple discrete chunks without requiring explicit meshing or pre-computed voxel storage for the entire world. It sets itself apart from [Chunk Generation with Base Terrain](1110-chunk-generation-with-base-terrain.md) by decoupling the structural definition (CPU) from the voxel evaluation (GPU), allowing for seamless normal generation via redundant chunk boundaries [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A deterministic seed, camera position, and a list of high-level object parameters (bounds, control points, thicknesses) computed in C++.
*   **Output:** A voxel grid for the current chunk, generated in the shader.

**Data Structures:**
*   **Object Definition (C++):** Structs containing start/end coordinates, control points, start/end thicknesses, and a computed bounding box.
*   **Shader Input:** A list of active objects and their parameters, passed per chunk.
*   **Chunk Context:** Start and end world-space coordinates defining the chunk's bounding box.

**Steps:**
1.  **High-Level Generation (CPU):** Generate world structures (e.g., world map, cities, roads) deterministically based on a seed. For each object (e.g., tree branch), compute its bounding box in world space.
2.  **Chunk Selection:** Identify which chunks intersect with the bounding boxes of active objects.
3.  **Parameter Passing (CPU to GPU):** For each chunk to be rendered, compile a list of objects whose bounding boxes intersect the chunk. Pass these object parameters to the shader.
4.  **Bounding Box Derivation (Shader):** For each chunk, define the start and end world coordinates (the chunk's bounding box).
5.  **Coordinate Transformation (Shader):** For each voxel in the chunk, derive its relative (object) coordinates by subtracting the chunk's start coordinates from the voxel's world coordinates.
6.  **Voxel Evaluation (Shader):** Evaluate each passed object against the voxel's relative coordinates. Determine if the voxel lies within the object's geometric definition (e.g., within a cylinder defined by control points and thickness).
7.  **Redundant Generation:** Generate voxels in a domain larger than the renderable chunk area (e.g., 1.25x the dimensions) to ensure seamless normal calculation at boundaries [S1, unofficial].

**Formulas:**
*   **Relative Coordinate:** $v_{rel} = v_{world} - c_{start}$
    *   $v_{world}$: World-space coordinate of the voxel.
    *   $c_{start}$: Start coordinate of the chunk's bounding box.
*   **Object Containment:** Determined by evaluating the voxel against the object's geometric primitive (e.g., distance to line segment for branches) using $v_{rel}$.

**Parameters:**
*   **Redundancy Factor:** 1.25 (current value, subject to change) [S1, unofficial].
*   **Generation Order:** Top-down (world -> cities -> neighborhoods -> objects) [S1, unofficial].
*   **Determinism:** All generation is deterministic based on a seed [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Seams:** Without redundant generation, normal vectors will be discontinuous at chunk boundaries.
*   **Performance:** Evaluating all objects for every chunk can be expensive; culling by bounding box is essential.
*   **Precision:** Floating-point precision in shader calculations may affect boundary accuracy for very large worlds.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CPU Side
def generate_world(seed):
    world_map = generate_map(seed)
    cities = place_cities(world_map, seed)
    for city in cities:
        neighborhoods = generate_neighborhoods(city, seed)
        for neighborhood in neighborhoods:
            streets = generate_streets(neighborhood, seed)
            for street in streets:
                buildings = generate_buildings(street, seed)
                for building in buildings:
                    building.compute_bounding_box()
                    active_objects.append(building)

def prepare_chunk(chunk_bounds, camera_pos):
    # Determine which objects are relevant to this chunk
    relevant_objects = []
    for obj in active_objects:
        if bbox_intersects(obj.bbox, chunk_bounds):
            relevant_objects.append(obj.params)
    return relevant_objects

# GPU Side (Shader)
struct ObjectParams {
    vec3 start_point;
    vec3 end_point;
    vec3 control_point;
    float start_thickness;
    float end_thickness;
    // ... other parameters
};

void main() {
    vec3 voxel_world_pos = get_voxel_world_position();
    vec3 chunk_start = get_chunk_start();
    
    // Derive relative coordinates
    vec3 rel_pos = voxel_world_pos - chunk_start;
    
    bool is_voxel = false;
    for (ObjectParams obj : u_objects) {
        // Evaluate if rel_pos is inside the object's geometry
        if (is_inside_object(rel_pos, obj)) {
            is_voxel = true;
            break;
        }
    }
    
    if (is_voxel) {
        // Generate voxel data, including normals
        // Note: Redundant generation ensures neighbors are available for normal calc
        set_voxel_data();
    }
}
```

## Subtopics

### [Chunk Redundancy for Seamless Normals](15485-chunk-redundancy-for-seamless-normals.md)
Solves normal discontinuities at chunk boundaries by generating voxels in a domain larger than the renderable area. Choose this when visual continuity of surface normals is critical and global data structures are undesirable.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of voxels in the chunk and $M$ is the number of active objects per chunk. Evaluation is parallelizable per voxel.
*   **Memory Complexity:** $O(N)$ for the current chunk's voxel data. No global voxel storage required.
*   **Performance:** Highly dependent on the number of active objects per chunk. Bounding box culling is critical to reduce $M$.
*   **Strengths:** Seamless normals, deterministic regeneration, no pre-computed storage, easy parallelization.
*   **Limitations:** High GPU load if many objects overlap a chunk; requires careful culling.
*   **When to use:** Procedural worlds with large, continuous structures where memory bandwidth is a constraint.
*   **When not to use:** Scenes with static, pre-computed geometry where storage is cheap.

## Background
Originating from the VoxelQuest project (2014), this method was developed to handle procedural generation of large voxel worlds without pre-computing the entire world data. It emphasizes a top-down approach where high-level structures are defined in C++ and evaluated at the voxel level in GLSL shaders based on camera locality [S1, unofficial].

## Sources
[S1] How Does It Work?, https://www.voxelquest.com/news/how-does-it-work, fetched 2026-10-29 (unofficial)
