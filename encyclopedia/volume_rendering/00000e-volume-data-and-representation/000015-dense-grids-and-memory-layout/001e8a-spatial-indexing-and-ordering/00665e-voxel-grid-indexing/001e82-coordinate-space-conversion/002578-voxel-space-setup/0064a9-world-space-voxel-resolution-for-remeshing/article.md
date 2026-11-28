# World-Space Voxel Resolution for Remeshing

Also called: Honor world-space voxel resolution independently of the active object's transform

## Overview
This method ensures that voxel resolution is defined in world units rather than the mesh's local coordinate frame, preventing resolution artifacts caused by object scale transforms [S1, unofficial]. It differs from standard implementations (e.g., Blender's native remesh modifier) by explicitly decoupling the voxel grid construction from the object's local coordinate system, ensuring that a specified world-space feature size results in consistent geometric fidelity regardless of the object's scale, rotation, or shear [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of source objects (operands) with potentially non-uniform scales, rotations, and translations; a target world-space voxel size (or feature/error target); and a chosen output object frame.
*   **Outputs:** A remeshed mesh in the output object frame, preserving geometric validity and fidelity.

**Data Structures:**
*   `JoinAttempt`: A record containing:
    *   `requested_world_size`: The target voxel dimension in world units.
    *   `execution_transform`: The transformation matrix of the working frame.
    *   `resolved_grid_size`: The dimensions of the resulting voxel grid.
    *   `world_bounds`: The bounding box of the geometry in the working frame.
    *   `native_runtime`: Performance metrics from the remeshing kernel.
    *   `realized_quality/cost`: Metrics for geometric fidelity and computational cost.

**Steps:**
1.  **Define Execution Frame:** Establish a rigid, unit-scale working frame for remeshing. This frame isolates the voxel grid construction from the arbitrary affine scale of the active object [S1, unofficial].
2.  **Transform Operands:** Transform all joined operands coherently into the execution frame. This preserves evaluated geometry and corrects reflected winding if the evaluated meshes cross reflected world transforms (e.g., negative scale components) [S1, unofficial].
3.  **Construct Evaluated Mesh:** Build an owned evaluated mesh in the working frame. Do not pass a scalar voxel size through the arbitrary affine scale of the source object [S1, unofficial].
4.  **Remesh:** Perform the voxel remeshing operation within the working frame. The voxel grid is constructed based on the world-space resolution target applied to the transformed geometry [S1, unofficial].
5.  **Map Result:** Transform the resulting mesh from the working frame into the chosen output object frame [S1, unofficial].
6.  **Validate and Commit:** Preserve source objects until the attempt is admitted and committed. Use geometric validity and fidelity checks (e.g., Guard mesh postprocessing) rather than connectivity scores alone to accept the result [S1, unofficial].
7.  **Record Provenance:** Record all parameters and results in the `JoinAttempt` structure. Explicitly preserve mode/fallback provenance; do not silently call an anisotropic result isotropic or overwrite source geometry during recovery from a failed remesh [S1, unofficial].

**Formulas:**
*   **Voxel Grid Construction:** The voxel grid is constructed from the mesh positions in the working frame and the `voxel_size` parameter, which is now interpreted as a world-unit distance [S1, unofficial].
*   **Scale Conversion:** For uniform scale $s$, the local voxel size $v_{local}$ can be converted to world voxel size $v_{world}$ as $v_{world} = v_{local} \times s$. However, for non-uniform scale or shear, no simple scalar conversion exists; the transform must be applied to the geometry, not the voxel size [S1, unofficial].

**Parameters:**
*   **Voxel Size:** The absolute floor for voxel size is a documented numerical/runtime limit or a scale-aware policy, not an accidental change in shape quality after unit conversion [S1, unofficial].
*   **Default Resolution:** In some implementations, the default voxel size is calculated as `max(world_bounds_extent) / 64` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Uniform Scale/Shear:** Dividing by one scale component or a mean is not a correct fix for non-uniform scale or shear. Unsupported or singular transforms must be rejected explicitly [S1, unofficial].
*   **Reflected Transforms:** When evaluated meshes cross reflected world transforms (e.g., negative scale), winding order must be corrected to preserve solid volume [S1, unofficial].
*   **Fidelity Acceptance:** A nonempty output alone is not sufficient for fidelity acceptance. Geometric validity and fidelity checks are required [S1, unofficial].
*   **Thin Features:** Known narrow gaps or thin features may be lost if the voxel size is too large relative to the feature size in world units [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def remesh_with_world_space_resolution(objects, target_world_voxel_size, output_object_frame):
    """
    Remesh a set of objects using world-space voxel resolution.
    
    Args:
        objects: List of source objects with transforms.
        target_world_voxel_size: Desired voxel size in world units.
        output_object_frame: The frame for the output mesh.
        
    Returns:
        Remeshed mesh in output_object_frame.
    """
    # 1. Define a rigid, unit-scale working frame
    # This frame isolates the voxel grid from object scales
    working_frame = IdentityMatrix()
    
    # 2. Transform operands into the working frame
    transformed_meshes = []
    for obj in objects:
        # Apply world transform to get geometry in working frame
        # Correct winding if transform includes reflection (negative determinant)
        mesh = obj.get_evaluated_mesh()
        transform = obj.get_world_matrix()
        if determinant(transform) < 0:
            mesh.flip_normals() # Correct reflected winding
        transformed_meshes.append(mesh.transform(transform))
        
    # 3. Join and build evaluated mesh in working frame
    combined_mesh = join_meshes(transformed_meshes)
    
    # 4. Remesh in working frame using world-space voxel size
    # The voxel kernel uses target_world_voxel_size directly on the transformed geometry
    remeshed_mesh = voxel_remesh(combined_mesh, target_world_voxel_size)
    
    # 5. Map result to output object frame
    output_transform = output_object_frame.get_world_matrix()
    final_mesh = remeshed_mesh.transform(inverse(output_transform))
    
    # 6. Validate and commit
    if not is_geometrically_valid(final_mesh):
        raise RemeshError("Geometric validity check failed")
        
    # 7. Record provenance
    attempt_record = JoinAttempt(
        requested_world_size=target_world_voxel_size,
        execution_transform=working_frame,
        resolved_grid_size=remeshed_mesh.grid_size,
        world_bounds=combined_mesh.bounds,
        native_runtime=get_runtime_stats(),
        realized_quality=compute_fidelity(combined_mesh, final_mesh)
    )
    
    return final_mesh
```

## Operating characteristics
*   **Time Complexity:** Depends on the voxel remeshing kernel, typically $O(N)$ where $N$ is the number of voxels. The transformation steps are $O(V)$ where $V$ is the number of vertices.
*   **Memory Complexity:** $O(N)$ for the voxel grid.
*   **Strengths:** Ensures consistent resolution across objects with different scales; prevents artifacts from non-uniform scaling; robust handling of reflected transforms.
*   **Limitations:** Requires explicit handling of complex transforms; may be slower than local-space remeshing due to coordinate transformations; rejection of unsupported transforms may limit applicability.
*   **When to Use:** When consistent world-space resolution is critical, such as in multi-object scenes with varying scales, or when importing geometry with non-uniform transforms.
*   **When Not to Use:** When local-space resolution is sufficient, or when performance is critical and scale variations are minimal.

## Background
This method addresses a limitation in standard voxel remeshing implementations (e.g., Blender's native remesh modifier) where the voxel size parameter is interpreted in the mesh's local coordinate frame, leading to inconsistent resolution when objects are scaled [S1, unofficial]. The approach was proposed to ensure that voxel resolution is defined in world units, independent of the active object's transform [S1, unofficial].

## Sources
[S1] Honor world-space voxel resolution independently of the active object's transform, https://github.com/doublemover/blendslop/issues/104, fetched 2026-10-28 (unofficial)
