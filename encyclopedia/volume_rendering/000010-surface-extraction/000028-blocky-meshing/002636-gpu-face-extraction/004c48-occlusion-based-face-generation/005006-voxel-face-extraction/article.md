# Voxel Face Extraction

Also called: Exposed Face Extraction, Solid & ~Shift Face Extraction

## Overview
Voxel Face Extraction is a method for generating surface geometry from a 3D voxel grid by identifying "exposed" faces—voxel sides adjacent to empty space or different material types. It operates on a per-chunk basis, iterating through defined face normals to compute a boolean mask of exposed voxels, which are then assembled into a face buffer. It is a specific implementation of the broader Occlusion-Based Face Generation strategy, distinguished by its explicit handling of axis-aligned shifts and boundary conditions to determine visibility without ray casting.

## How it works
### Inputs and Outputs
- **Input**: A 3D voxel grid `solid` of shape `(depth, height, width)` with dtype `uint8` or `bool`, where non-zero values indicate solid voxels. A list of face normals defined in `geometry.FACE_DATA`. Configuration flags `is_flat_asset` and `settings` (containing `CHUNK_SIZE`, `MAP_DEPTH`, etc.).
- **Output**: A `VoxelAsset` object containing a stacked face buffer array of shape `(N, 19)` with dtype `float32`, and chunk dimensions. If no faces are found, the buffer is empty `(0, 19)`.

### Data Structures
- **`solid`**: 3D array representing the voxel chunk.
- **`exposed`**: 3D boolean array of the same shape as `solid`, marking voxels with exposed faces for the current normal.
- **`assembled_list`**: List of face buffer arrays, each corresponding to a batch of exposed voxels for a specific normal.
- **`VoxelAsset`**: Container for the final concatenated face buffer and metadata.

### Steps
1. **Initialize**: Create an empty list `assembled_list`.
2. **Iterate Normals**: For each face normal `(nx, ny, nz)` in `geometry.FACE_DATA`:
   - **Skip Conditions**:
     - If `nz == -1`, skip this normal [S1, unofficial].
     - If `is_flat_asset` is true and `nz != 1`, skip this normal [S1, unofficial].
   - **Compute Exposed Mask**:
     - **Case `nz == 1` (Top Face)**:
       - If `depth == 1`, set `exposed = solid.copy()` [S1, unofficial].
       - Otherwise, compute `z_shift` by rolling `solid` by `-1` along axis 0 (depth) [S1, unofficial].
       - Set `exposed = solid & ~z_shift` [S1, unofficial].
       - Force the last slice to be exposed if solid: `exposed[-1, :, :] = solid[-1, :, :]` [S1, unofficial].
     - **Case `nz != 1` (Side Faces)**:
       - Initialize `exposed` as zeros (boolean) of shape `(depth, height, width)` [S1, unofficial].
       - Compute `y_shift`: If `ny != 0`, roll `solid` by `-ny` along axis 1 (height); else `y_shift = solid` [S1, unofficial].
       - Compute `x_shift`: If `nx != 0`, roll `solid` by `-nx` along axis 2 (width); else `x_shift = solid` [S1, unofficial].
       - Update `exposed`:
         - If `nx != 0`, `exposed = exposed & ~x_shift` [S1, unofficial].
         - If `ny != 0`, `exposed = exposed & ~y_shift` [S1, unofficial].
       - **Boundary Conditions**:
         - If `nx == 1`, set `exposed[:, :, -1] = solid[:, :, -1]` [S1, unofficial].
         - If `nx == -1`, set `exposed[:, :, 0] = solid[:, :, 0]` [S1, unofficial].
         - If `ny == 1`, set `exposed[:, -1, :] = solid[:, -1, :]` [S1, unofficial].
         - If `ny == -1`, set `exposed[:, 0, :] = solid[:, 0, :]` [S1, unofficial].
   - **Assemble Faces**:
     - Get indices of exposed voxels: `indices = np.where(exposed)` [S1, unofficial].
     - If `indices` is empty, continue to next normal [S1, unofficial].
     - Call `geometry.assemble_face_buffer` with:
       - Chunk indices (local coordinates within the chunk).
       - Full indices (global coordinates, if applicable).
       - `solid` voxel values.
       - Current normal `(nx, ny, nz)`.
       - Vertex offsets `v_offsets` (defined in `geometry`).
       - Registry (material ID mapping).
     - Append result to `assembled_list` [S1, unofficial].
3. **Finalize**:
   - If `assembled_list` is empty, return `VoxelAsset` with empty array `(0, 19)` dtype `float32` and dimensions [S1, unofficial].
   - Otherwise, concatenate all arrays in `assembled_list` into `stacked_faces` [S1, unofficial].
   - Return `VoxelAsset` with `stacked_faces` and dimensions [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_face_extraction(solid: NDArray[np.uint8], is_flat_asset: bool) -> VoxelAsset:
    assembled_list = []
    depth, height, width = solid.shape
    
    for normal in geometry.FACE_DATA:
        nx, ny, nz = normal
        
        # Skip conditions
        if nz == -1:
            continue
        if is_flat_asset and nz != 1:
            continue
            
        exposed = np.zeros((depth, height, width), dtype=bool)
        
        if nz == 1:
            if depth == 1:
                exposed = solid > 0
            else:
                z_shift = np.roll(solid, -1, axis=0)
                exposed = (solid > 0) & (z_shift == 0)
                exposed[-1, :, :] = solid[-1, :, :] > 0
        else:
            # Side faces
            y_shift = solid if ny == 0 else np.roll(solid, -ny, axis=1)
            x_shift = solid if nx == 0 else np.roll(solid, -nx, axis=2)
            
            # Initialize exposed based on neighbors
            # For side faces, exposed is true if the neighbor in the normal direction is empty
            # The logic in [S1, unofficial] uses ANDing with ~shifts. 
            # If nx != 0, we check x_shift. If ny != 0, we check y_shift.
            # Note: The initial 'exposed' is zeros. 
            # The logic implies: exposed = (~x_shift) & (~y_shift) for the relevant axes?
            # Actually, [S1, unofficial] says: "Update exposed by ANDing with ~x_shift if nx != 0..."
            # This implies exposed starts as all True? No, it starts as zeros.
            # Let's re-read carefully: "initialize exposed as zeros. ... Update exposed by ANDing..."
            # This seems to imply that for side faces, the mask is built up.
            # However, standard face extraction for side faces usually checks the neighbor.
            # If nx=1, we look at x+1. If x+1 is empty, current is exposed.
            # The shift logic: z_shift = solid rolled by -1. 
            # If solid[x] is 1 and solid[x+1] is 0, then exposed[x] should be 1.
            # z_shift[x] = solid[x+1]. So ~z_shift[x] is 1 if solid[x+1] is 0.
            # So exposed = solid & ~shift.
            
            # Re-evaluating [S1, unofficial] for non-(0,0,1):
            # "initialize exposed as zeros"
            # "Update exposed by ANDing with ~x_shift if nx != 0"
            # "Update exposed by ANDing with ~y_shift if ny != 0"
            # This logic is flawed if exposed starts as zeros. It would remain zeros.
            # It is likely that exposed should be initialized to `solid > 0` or the logic implies
            # that the ANDing is applied to a mask of `solid`.
            # Let's assume the intent is: exposed = solid & (~x_shift if nx!=0 else True) & (~y_shift if ny!=0 else True)
            # But [S1, unofficial] explicitly says "initialize exposed as zeros".
            # Let's look at the boundary conditions. They set exposed to solid.
            # This suggests that for side faces, the "exposed" mask is only partially computed by shifts?
            # No, the boundary conditions are for the edges of the chunk.
            
            # Correction based on standard logic and [S1, unofficial] structure:
            # The "initialize as zeros" might be a mistake in the note or implies a different accumulation.
            # However, to be faithful to [S1, unofficial]:
            # If nx != 0: exposed = exposed & (~x_shift)
            # If ny != 0: exposed = exposed & (~y_shift)
            # This results in all zeros if exposed starts as zeros.
            # Let's assume the note meant "initialize exposed to solid > 0" or the ANDing is ORing?
            # No, "ANDing with ~shift" is correct for "neighbor is empty".
            # Let's assume the note has a typo and exposed should be initialized to `solid > 0` for side faces too,
            # OR the logic is: exposed = (~x_shift) & (~y_shift) & solid.
            
            # Let's stick to the most logical interpretation of "Exposed Face Extraction":
            # A voxel is exposed if it is solid AND its neighbor in the normal direction is empty.
            # For nz=1: neighbor is z+1. z_shift = roll(solid, -1, axis=0). z_shift[i] = solid[i+1].
            # exposed = solid & (z_shift == 0).
            # For nx=1: neighbor is x+1. x_shift = roll(solid, -1, axis=2).
            # exposed = solid & (x_shift == 0).
            
            # The note [S1, unofficial] says: "initialize exposed as zeros. ... Update exposed by ANDing with ~x_shift..."
            # This is contradictory. I will implement the logical version which matches the nz=1 case structure.
            
            exposed = solid > 0
            if nx != 0:
                x_shift = np.roll(solid, -nx, axis=2)
                exposed = exposed & (x_shift == 0)
            if ny != 0:
                y_shift = np.roll(solid, -ny, axis=1)
                exposed = exposed & (y_shift == 0)
                
            # Boundary conditions from [S1, unofficial]
            if nx == 1:
                exposed[:, :, -1] = solid[:, :, -1] > 0
            if nx == -1:
                exposed[:, :, 0] = solid[:, :, 0] > 0
            if ny == 1:
                exposed[:, -1, :] = solid[:, -1, :] > 0
            if ny == -1:
                exposed[:, 0, :] = solid[:, 0, :] > 0

        indices = np.where(exposed)
        if len(indices[0]) == 0:
            continue
            
        batch_faces = geometry.assemble_face_buffer(
            chunk_indices=indices,
            full_indices=indices, # Assuming local indices are used for buffer assembly in this context
            voxels=solid[exposed],
            normal=normal,
            v_offsets=geometry.v_offsets,
            registry=registry
        )
        assembled_list.append(batch_faces)
        
    if not assembled_list:
        return VoxelAsset(np.empty((0, 19), dtype=np.float32), dimensions=(depth, height, width))
        
    stacked_faces = np.concatenate(assembled_list, axis=0)
    return VoxelAsset(stacked_faces, dimensions=(depth, height, width))
```

### Parameters
- `is_flat_asset`: Boolean flag. If true, only top faces (`nz=1`) are generated [S1, unofficial].
- `geometry.FACE_DATA`: List of normals `(nx, ny, nz)` to process [S1, unofficial].
- `geometry.v_offsets`: Vertex offsets for face assembly [S1, unofficial].
- `registry`: Material ID registry [S1, unofficial].

### Edge Cases and Pitfalls
- **Depth 1**: If `depth == 1`, the `nz=1` case copies `solid` directly to `exposed` [S1, unofficial].
- **Boundary Conditions**: For side faces, the shift logic does not account for chunk boundaries. Explicit boundary conditions are required to ensure voxels on the chunk edge are correctly marked as exposed if they are solid [S1, unofficial].
- **Empty Chunk**: If no exposed voxels are found, an empty `(0, 19)` array is returned [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel is processed a constant number of times (once per normal).
- **Memory Complexity**: $O(N)$ for the `exposed` mask and `assembled_list`.
- **Performance**: Efficient for dense chunks. Performance degrades if many normals are processed for flat assets (though `is_flat_asset` mitigates this).
- **Strengths**: Simple, deterministic, no ray casting overhead.
- **Limitations**: Generates blocky geometry. Does not handle diagonal faces or smooth surfaces.

## Background
Voxel Face Extraction is implemented in the `pyvorengi-sdk-demo` repository by herbal1st [S1, unofficial]. It is part of a voxel engine SDK, likely for Minecraft-like terrain generation. The method is a direct implementation of face culling based on neighbor checks.

## Sources
[S1] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
