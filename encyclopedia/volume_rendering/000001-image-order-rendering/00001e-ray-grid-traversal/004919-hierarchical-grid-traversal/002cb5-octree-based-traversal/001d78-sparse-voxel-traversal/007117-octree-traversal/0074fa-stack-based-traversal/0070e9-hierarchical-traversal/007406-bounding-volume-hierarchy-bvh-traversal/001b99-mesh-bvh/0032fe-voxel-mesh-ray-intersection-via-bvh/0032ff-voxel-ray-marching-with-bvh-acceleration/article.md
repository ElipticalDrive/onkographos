# Voxel Ray Marching with BVH Acceleration

Also called: VoxelMeshQuery, VoxelRayRender

## Overview
A compute shader-based ray tracing technique for voxel data that uses a Bounding Volume Hierarchy (BVH) to accelerate ray-mesh intersection tests for lighting and ambient occlusion calculations [S1, unofficial]. It differs from hardware-accelerated approaches like [Sparse Voxel Set (SVS)](9654-sparse-voxel-set-svs.md) or [GPU-based Ray Shooting for Volumetric Mapping](18280-gpu-based-ray-shooting-for-volumetric-mapping.md) by performing the traversal logic entirely within a custom compute shader rather than relying on dedicated DXR hardware pipelines [S1, unofficial]. This method is specifically designed for deferred shading pipelines where position and normal buffers are available, enabling efficient shadow and occlusion queries against sparse voxel meshes [S1, unofficial].

## How it works
The method operates on a per-pixel basis in a compute shader, utilizing position and normal textures as input buffers to generate lighting and ambient occlusion (AO) textures [S1, unofficial]. It delegates the heavy lifting of ray-voxel intersection to a software-implemented BVH traversal function (`voxelMeshQuery`) [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   `position_texture`: `texture_2d<f32>` storing world position at screen coordinates [S1, unofficial].
*   `normal_texture`: `texture_2d<f32>` storing surface normals [S1, unofficial].
*   `scene_props`: Contains `sunPosition` [S1, unofficial].
*   `camera`: Contains `right`, `up`, `forward` vectors and `tanHalfFov` [S1, unofficial].
*   `voxel_bvh`: `array<AABB>` representing the hierarchy nodes [S1, unofficial].
*   `voxel_indice`: `array<vec2u>` mapping BVH nodes to mesh indices [S1, unofficial].
*   `voxel_meshes`: `array<VoxelMesh>` containing mesh data [S1, unofficial].
*   `vertices`: `array<VoxelMeshVertex>` [S1, unofficial].
*   `indices`: `array<u32>` [S1, unofficial].

**Outputs:**
*   `light_texture`: `texture_storage_2d<rgba32float, write>` for sun lighting [S1, unofficial].
*   `ao_texture`: `texture_storage_2d<rgba16float, write>` for ambient occlusion [S1, unofficial].

### Data Structures
**VoxelMeshVertex** [S1, unofficial]:
*   `position`: `vec3f`
*   `normal`: `vec3f`
*   `textureIndex`: `vec3f`
*   `uv`: `vec2f`
*   `colors`: `vec3f`
*   `voxelData`: `f32`
*   `_padding`: `vec4f`

**AABB** (implied from BVH usage):
*   Minimum and maximum bounds defining the volume [S1, unofficial].

### Parameters
*   Sun ray max distance: 300.0 [S1, unofficial].
*   AO ray max distance: 0.99 [S1, unofficial].
*   Epsilon offset for shadow/occlusion ray origin: 0.01 [S1, unofficial].
*   AO sample count: 4 [S1, unofficial].

### Steps
1.  **Initialization**: Calculate screen coordinates from the global invocation ID multiplied by 2 [S1, unofficial].
2.  **Position Check**: Load the world position from `position_texture` using the screen coordinates [S1, unofficial]. If the position's alpha component is $\le 0$, store $1.0$ in the light and AO textures and return [S1, unofficial].
3.  **Normal Extraction**: Extract the world position and normal from the textures [S1, unofficial]. Convert the normal from the $[0, 1]$ range to the $[-1, 1]$ range using:
    $$ \mathbf{n} = 2 \cdot \mathbf{c}_{normal} - 1 $$
    where $\mathbf{c}_{normal}$ is the color value from `normal_texture` [S1, unofficial]. Set `hitPosition` to the position's RGB components [S1, unofficial].
4.  **Sun Lighting Calculation**:
    *   Offset `hitPosition` along the normal by epsilon ($\epsilon = 0.01$) to get `newHitPosition` [S1, unofficial].
    *   Calculate the sun direction vector from `newHitPosition` to `scene_props.sunPosition` [S1, unofficial].
    *   Construct an orthonormal basis for the sun direction [S1, unofficial].
    *   Call `voxelMeshQuery` with `newHitPosition`, the sun direction, and a max distance of 300.0 [S1, unofficial].
    *   If an intersection is found, set the light value to dark gray $(0.25, 0.25, 0.25)$ [S1, unofficial]. If an error occurs, set the light value to red $(1, 0, 0)$ [S1, unofficial].
5.  **Ambient Occlusion Calculation**:
    *   Build an orthonormal basis from the surface normal using `buildOrthonormalBasis` [S1, unofficial].
    *   Initialize a `hit` counter to 0 [S1, unofficial].
    *   Loop 4 times (for the sample count):
        *   Generate random numbers using `getTwoRandomNumbers` with a seed based on fragment coordinates and time [S1, unofficial].
        *   Sample a hemisphere direction using `cosineWeightedHemisphereSample` [S1, unofficial].
        *   Transform the local sample to world space using `hemisphereToWorld` [S1, unofficial].
        *   Construct an orthonormal basis for the sample direction [S1, unofficial].
        *   Call `voxelMeshQuery` with `hitPosition`, the sample direction, and a max distance of 0.99 [S1, unofficial].
        *   If an intersection is found, increment the `hit` counter [S1, unofficial].
    *   Calculate the occlusion factor:
        $$ f_{occlusion} = 1.0 - \frac{\text{hit}}{4} $$
    *   Apply this factor to the AO color channels [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper functions defined in source [S1, unofficial]

def build_orthonormal_basis(N: vec3) -> Basis:
    """Constructs a basis (T, B, N) avoiding singularity when N is parallel to up."""
    up = vec3(0.0, 1.0, 0.0) if abs(N.z) < 0.999 else vec3(0.0, 0.0, 1.0)
    T = normalize(cross(up, N))
    B = normalize(cross(N, T))
    return Basis(T, B, N)

def cosine_weighted_hemisphere_sample(u: f32, v: f32) -> vec3:
    """Generates a direction on the upper hemisphere weighted by cosine."""
    phi = 2.0 * 3.1415926535 * u
    r = sqrt(v)
    x = r * cos(phi)
    y = r * sin(phi)
    z = sqrt(1.0 - v)
    return vec3(x, y, z)

def voxel_ray_marching_with_bvh_acceleration(global_invocation_id: vec3, 
                                            position_texture: Texture2D, 
                                            normal_texture: Texture2D, 
                                            light_texture: StorageTexture2D, 
                                            ao_texture: StorageTexture2D,
                                            scene_props: SceneProps,
                                            camera: Camera,
                                            voxel_bvh: Array[AABB],
                                            voxel_indice: Array[vec2u],
                                            voxel_meshes: Array[VoxelMesh],
                                            vertices: Array[VoxelMeshVertex],
                                            indices: Array<u32>):
    
    # 1. Calculate screen coordinates
    screen_coord = global_invocation_id.xy * 2
    
    # 2. Load position
    pos_color = position_texture.sample(screen_coord)
    if pos_color.a <= 0.0:
        light_texture.store(screen_coord, vec4(1.0, 1.0, 1.0, 1.0))
        ao_texture.store(screen_coord, vec4(1.0, 1.0, 1.0, 1.0))
        return

    # 3. Extract world position and normal
    hit_position = pos_color.rgb
    normal_color = normal_texture.sample(screen_coord)
    normal = (2.0 * normal_color) - 1.0
    
    # 4. Sun Lighting
    new_hit_position = hit_position + normal * 0.01 # Epsilon offset
    
    sun_direction = normalize(scene_props.sun_position - new_hit_position)
    sun_basis = build_orthonormal_basis(sun_direction)
    
    # Call BVH query for sun
    # Note: The specific implementation of voxelMeshQuery is not provided in the source,
    # but it takes origin, direction, and max distance.
    sun_hit = voxel_mesh_query(new_hit_position, sun_direction, 300.0, 
                               voxel_bvh, voxel_indice, voxel_meshes, vertices, indices)
    
    if sun_hit:
        light_color = vec4(0.25, 0.25, 0.25, 1.0)
    else:
        # Error case handling as per source
        light_color = vec4(1.0, 0.0, 0.0, 1.0)
        
    light_texture.store(screen_coord, light_color)
    
    # 5. Ambient Occlusion
    surf_basis = build_orthonormal_basis(normal)
    hit_count = 0
    
    for i in range(4):
        # Generate random numbers based on seed
        u, v = get_two_random_numbers(screen_coord, time)
        
        # Sample hemisphere
        local_sample = cosine_weighted_hemisphere_sample(u, v)
        
        # Transform to world space
        world_sample = hemisphere_to_world(local_sample, surf_basis)
        
        # Construct basis for sample direction (for BVH traversal if needed)
        sample_basis = build_orthonormal_basis(world_sample)
        
        # Call BVH query for AO
        ao_hit = voxel_mesh_query(hit_position, world_sample, 0.99,
                                  voxel_bvh, voxel_indice, voxel_meshes, vertices, indices)
        
        if ao_hit:
            hit_count += 1
            
    occlusion_factor = 1.0 - (hit_count / 4.0)
    ao_color = vec4(occlusion_factor, occlusion_factor, occlusion_factor, 1.0)
    ao_texture.store(screen_coord, ao_color)
```

### Edge Cases and Pitfalls
*   **Shadow Acne/Bleeding**: The ray origin for sun and AO queries is offset from the surface position along the normal by a small epsilon ($0.01$) to prevent self-intersection [S1, unofficial].
*   **Basis Construction Singularity**: The `buildOrthonormalBasis` function selects the 'up' vector based on the Z component of the normal to avoid cross product degeneracy when the normal is parallel to the default up vector [S1, unofficial].
*   **Empty Pixels**: If the position texture alpha is $\le 0$, the shader returns immediately with full brightness to avoid processing empty space [S1, unofficial].

## Subtopics

### [Sparse Voxel Set (SVS)](9654-sparse-voxel-set-svs.md)
Uses BVH but offloads traversal to DXR hardware. Choose when hardware ray tracing is available and custom shader overhead is undesirable.

### [GPU-based Ray Shooting for Volumetric Mapping](18280-gpu-based-ray-shooting-for-volumetric-mapping.md)
Uses BVH for occupancy queries in PVM via DXR. Choose for volumetric mapping tasks requiring free/occupied voxel determination.

## Operating characteristics
*   **Time Complexity**: Depends on the BVH traversal depth and the number of samples (4 for AO). The BVH traversal is generally $O(\log N)$ for sparse data, but worst-case can be $O(N)$ if the BVH is unbalanced or the scene is dense [S1, unofficial].
*   **Memory Complexity**: Requires storage for the BVH nodes (`voxel_bvh`), indices (`voxel_indice`), meshes (`voxel_meshes`), and vertices (`vertices`) [S1, unofficial].
*   **Performance**: Offloading traversal to a custom compute shader avoids the overhead of setting up DXR pipelines but requires manual implementation of the BVH traversal logic [S1, unofficial].
*   **Strengths**: Flexible, works on hardware without DXR support, integrates well with deferred shading pipelines [S1, unofficial].
*   **Limitations**: Requires manual implementation of BVH traversal, potentially slower than hardware-accelerated methods on supported GPUs [S1, unofficial].
*   **When to use**: When DXR is not available or when custom lighting/occlusion logic is required that doesn't fit standard ray tracing pipelines [S1, unofficial].
*   **When not to use**: When hardware ray tracing is available and standard ray tracing features are sufficient [S1, unofficial].

## Background
This method is described in the Divine Voxel Engine repository [S1, unofficial]. It is designed for real-time voxel rendering, specifically for calculating lighting and ambient occlusion in a deferred rendering context [S1, unofficial].

## Sources
[S1] Divine-Star-Software/DivineVoxelEngine, https://github.com/Divine-Star-Software/DivineVoxelEngine, fetched 2026-10-26 (unofficial)
