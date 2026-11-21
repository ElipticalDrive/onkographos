# Per-Frame SSBO Reconstruction for Voxel Buckets

Also called: New SSBO per bucket per frame, VXL-041

## Overview
Per-Frame SSBO Reconstruction is a GPU-driven voxel rendering architecture that abandons incremental instance management in favor of completely discarding and recreating Shader Storage Buffer Objects (SSBOs) every frame for each non-empty render bucket [S1]. It differs from its parent, the Instanced Voxel Renderer, by rejecting persistent buffers, double buffering, swap-and-pop optimizations, and partial uploads, opting instead for a "write-once, draw-many" model where instance data is rebuilt from scratch each frame [S1]. This approach simplifies synchronization and state management at the cost of higher memory bandwidth and allocation overhead [S1].

## How it works
The method operates within the `VoxelRenderingSystem` and relies on the `RenderSnapshot` architecture. It processes active scene components to generate immutable GPU buffers for each frame.

### Inputs and Outputs
*   **Input:** A `Registry` containing active `Model` and `Assembly` components, each with local transforms, parent references, and mesh/material bindings [S1].
*   **Output:** A `RenderSnapshot` containing `DrawVoxelBucketCommand` objects, each bound to a newly created, immutable `ShaderStorageBuffer::Handle` [S1].

### Data Structures
The core data structures are defined in the `spk::Voxel` namespace [S1]:

1.  **`GPUInstance`**: A struct containing the dynamic data for a single voxel instance. Fields include:
    *   World transform matrix [S1].
    *   Palette or material mapping [S1].
    *   Shader instance fields [S1].
    *   *Layout:* The exact byte layout is determined by the compiler packing of these fields, aligned for SSBO binding [S1].

2.  **`BucketKey`**: A struct used to group instances that can be drawn together. Fields include:
    *   Mesh identity [S1].
    *   Render pass identifier [S1].
    *   Compatible GPU bindings [S1].

3.  **`DrawVoxelBucketCommand`**: A render command inheriting from `spk::RenderCommand`. It stores:
    *   Immutable mesh and material bindings [S1].
    *   A `ShaderStorageBuffer::Handle` specific to the current snapshot [S1].
    *   The number of instances [S1].

### Steps
The reconstruction process occurs in `VoxelRenderingSystem::_buildRenderSnapshot(builder)` [S1]:

1.  **Collection:** Collect all active `Model` and `Assembly` components from the `Registry` [S1].
2.  **Transform Calculation:** Recalculate the world transform for every collected component. The world matrix is computed as `worldEntityMatrix * parentPartMatrices * local` [S1].
3.  **Bucketing:** Group the calculated occurrences into compact vectors (buckets) based on `BucketKey`. Instances sharing the same mesh identity, render pass, and compatible GPU bindings are grouped together [S1].
4.  **SSBO Creation:** For each non-empty bucket:
    a.  Create a new `spk::ShaderStorageBuffer` object [S1].
    b.  Resize the SSBO to `instances.size()` [S1].
    c.  Write the full dynamic data array using `std::span<const GPUInstance>` [S1].
    d.  Call `ssbo.validate()` to signal modification and prepare the buffer for GPU upload [S1].
5.  **Command Construction:** Capture the immutable `ShaderStorageBuffer::Handle` from the SSBO [S1]. Emplace a `DrawVoxelBucketCommand` into the `RenderSnapshot` builder, passing the mesh/material bindings, the handle, and the instance count [S1].
6.  **Execution:** During rendering, `DrawVoxelBucketCommand::execute(RenderContext&)` activates the SSBO handle [S1]. The `RenderContext` triggers GPU resource synchronization (uploading the entire buffer content) and performs the instanced draw call [S1].

### Pseudocode
The following Python-like pseudocode represents the core logic of `_buildRenderSnapshot`. It assumes standard vector and matrix operations are available.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_render_snapshot(registry, builder):
    """
    Reconstructs voxel buckets and SSBOs for the current frame.
    """
    # 1. Collect active components
    active_components = registry.get_active_models_and_assemblies()
    
    # Dictionary mapping BucketKey to list of GPUInstance
    buckets = {}
    
    for component in active_components:
        # 2. Recalculate world transform
        # worldEntityMatrix * parentPartMatrices * local
        world_transform = compute_world_transform(component)
        
        # Create GPUInstance data
        gpu_instance = GPUInstance(
            world_transform=world_transform,
            palette=component.palette_mapping,
            shader_fields=component.shader_instance_data
        )
        
        # 3. Group into buckets
        key = BucketKey(
            mesh_id=component.mesh_identity,
            render_pass=component.render_pass,
            bindings=component.gpu_bindings
        )
        
        if key not in buckets:
            buckets[key] = []
        buckets[key].append(gpu_instance)
    
    # 4 & 5. Create SSBOs and Commands
    for key, instances in buckets.items():
        if not instances:
            continue
            
        # Create new SSBO
        ssbo = ShaderStorageBuffer(
            binding_point=key.bindings,
            size=0,
            element_size=sizeof(GPUInstance)
        )
        
        # Resize and write full data
        ssbo.resize(len(instances))
        ssbo.set_dynamic_data(std_span(instances))
        
        # Signal modification
        ssbo.validate()
        
        # Capture immutable handle
        handle = ssbo.handle()
        
        # Add command to snapshot
        builder.render_pass(key.render_pass).emplace(
            DrawVoxelBucketCommand(
                mesh_bindings=key.mesh_id,
                material_bindings=key.bindings,
                ssbo_handle=handle,
                instance_count=len(instances)
            )
        )

    # Note: SSBOs must not be modified after handle publication [S1]
    # Note: Instance removal is handled by simply not inserting into the bucket [S1]
```

### Parameters and Edge Cases
*   **BucketKey:** Must accurately reflect all GPU state differences (mesh, pass, bindings) to ensure correct grouping [S1].
*   **Instance Removal:** Handled by omitting the instance from the next frame's bucket. No holes or sentinels are left; buckets are always compact [S1].
*   **Immutability:** SSBO handles must be immutable after publication to the snapshot. Modifying the SSBO after creating the handle is forbidden [S1].
*   **Lifecycle:** Each `RenderSnapshot` must retain its SSBO handles and immutable resources until execution completes. A new snapshot cannot alter an old SSBO [S1].
*   **Double Buffering:** Explicit double buffering of SSBOs is not implemented. New SSBOs are created per snapshot instead of swapping [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per frame, where $N$ is the number of active instances. The cost includes collecting components, calculating transforms, grouping, and allocating/uploading buffers.
*   **Memory Complexity:** $O(N)$ for the SSBO storage per frame. Since buffers are recreated every frame, peak memory usage may be higher due to temporary allocations before garbage collection or pool reuse [S1].
*   **Performance:** The method incurs the cost of full GPU object creation and complete data upload every frame, regardless of whether instances changed [S1]. This is a deliberate trade-off for simplicity [S1].
*   **Strengths:** Simplified synchronization (no dirty ranges, no partial uploads), no need for complex double-buffering logic, and robust handling of dynamic scene changes [S1].
*   **Limitations:** High bandwidth usage and allocation overhead for static or slowly changing scenes [S1].
*   **When to Use:** Scenes with high instance turnover or when implementation simplicity is prioritized over bandwidth efficiency [S1].
*   **When Not to Use:** Scenes with mostly static instances where incremental updates or persistent buffers would offer significant performance gains [S1].

## Background
This architecture was introduced in version 0.1.4 of the Sparkle voxel engine design [S1]. It represents a shift away from previous architectures that used incremental updates, persistent SSBOs, and swap-and-pop optimizations [S1]. The design was validated as VXL-041 [S1].

## Sources
[S1] V0.1.4 — Voxel engine design and implementation specification, https://github.com/EreliaStudio/Sparkle/pull/37, fetched 2026-10-26
