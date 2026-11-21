# Voxel Chunk Data Management and Residency

Also called: Voxel Chunk Compression and Residency Management, Far-Chunk Voxel Data Compression & Residency Management, Chunk residency tracking

## Overview
This method manages the memory residency of voxel chunks in an infinite voxel world by tracking state transitions between raw, uniform, and compressed forms to reduce memory footprint for distant chunks [S1, unofficial]. It solves the problem of unbounded memory usage in dense voxel grids by converting raw voxel arrays into compact representations (Uniform or Compressed) and dropping the resident raw array, while maintaining data consistency through versioning and mutation flags [S1, unofficial]. Unlike sibling structures that focus on spatial partitioning or octree traversal, this method operates on the data lifecycle within a fixed-size chunk, using distance-keyed asynchronous compression to prioritize memory savings for far chunks [S1, unofficial].

## How it works

### Inputs and Outputs
- **Inputs**: Chunk coordinates, local voxel positions, raw voxel data arrays, compression codec, chunk size.
- **Outputs**: Expanded voxel data, compressed buffers, residency state flags, version counters.

### Data Structures

**`FChunkDescriptor`**
Fields:
- `ChunkCoord`: `FIntVector` (32-bit int per axis).
- `LODLevel`: `int32`.
- `ChunkSize`: `int32` (edge length in voxels).
- `VoxelArray`: `TSharedPtr<TArray<FVoxelData>>` (shared pointer to raw voxel data).
- `Bounds`: `FBox`.
- `bIsDirty`: `bool`.
- `bHasEdits`: `bool`.
- `MorphFactor`: `float`.
- `GenerationSeed`: `int32`.
- `State`: `EChunkState` (enum).
- `Residency`: `EVoxelDataResidency` (enum: `Resident`, `Empty`, `Uniform`, `Compressed`).
- `bDataMutated`: `bool`.
- `ContentVersion`: `uint32` (monotonic counter).
- `UniformValue`: `FVoxelData`.
- `bUniformValueValid`: `bool`.
- `bCompressionEvaluated`: `bool`.
- `CompressedVoxelData`: `TArray<uint8>` (dynamic buffer).
- `AsyncCompactTicket`: `uint32`.

**`FVoxelChunkCompactForm`**
Fields:
- `Kind`: `uint8` enum (`EKind`: `Incompressible`, `Uniform`, `Compressed`).
- `UniformValue`: `FVoxelData`.
- `Buffer`: `TArray<uint8>` (compressed data).

**`FVoxelData`**
Per-voxel data structure containing MaterialID, Density, BiomeID, and Metadata. Size is typically 4 bytes (32-bit) to allow word-aligned XOR operations [S1, unofficial].

### Numbered Steps

1.  **Initialization (`AllocateVoxelData`)**:
    - Calculate `TotalVoxels = ChunkSize^3`.
    - Create `VoxelArray` with `SetNumZeroed(TotalVoxels)`.
    - Set `Residency` to `Resident`.
    - Reset `bDataMutated`, `bUniformValueValid`, `bCompressionEvaluated`.
    - Clear `CompressedVoxelData`.
    - Reset `AsyncCompactTicket` to 0.
    - Increment `ContentVersion`.

2.  **Setting Resident Data (`SetResidentVoxelData`)**:
    - Replace `VoxelArray` with input data (moved).
    - Set `Residency` to `Resident`.
    - Reset `bDataMutated`, `bUniformValueValid`, `bCompressionEvaluated`.
    - Clear `CompressedVoxelData`.
    - Reset `AsyncCompactTicket`.
    - Increment `ContentVersion`.

3.  **Clearing Data (`ClearVoxelData`)**:
    - Reset `VoxelArray`.
    - Set `Residency` to `Empty`.
    - Reset `bDataMutated`, `bUniformValueValid`, `bCompressionEvaluated`.
    - Clear `CompressedVoxelData`.
    - Reset `AsyncCompactTicket`.
    - **Do NOT** increment `ContentVersion` (clearing is memory teardown, not a logical change) [S1, unofficial].

4.  **Reading a Voxel (`GetVoxel`)**:
    - Calculate `Index` from `LocalPos` using the linear index formula.
    - Get `Data` from `GetResidentArray()`.
    - Return `Data[Index]` if valid, else `FVoxelData::Air()` [S1, unofficial].

5.  **Writing a Voxel (`SetVoxel`)**:
    - Calculate `Index`.
    - If index is valid in resident array:
        - Detach `VoxelArray` (using `IsUnique()` check to avoid unnecessary copies) [S1, unofficial].
        - Set `DetachVoxelArray()[Index] = Data`.
        - Set `bIsDirty = true`, `bDataMutated = true`.
        - Reset `bUniformValueValid`, `bCompressionEvaluated`.
        - Clear `CompressedVoxelData`.
        - Reset `AsyncCompactTicket`.
        - Increment `ContentVersion`.

6.  **Checking Uniformity (`TryCollapseUniform`)**:
    - If `Residency` is not `Resident`, return `false`.
    - If `bUniformValueValid` is `true`:
        - Reset `VoxelArray`.
        - Set `Residency` to `Uniform`.
        - Reset `AsyncCompactTicket`.
        - Return `true`.
    - Else:
        - Set `bCompressionEvaluated = true`.
        - Call `ComputeUniformValue(GetResidentArray(), Value)`.
        - If true:
            - Set `UniformValue = Value`, `bUniformValueValid = true`.
            - Reset `VoxelArray`.
            - Set `Residency` to `Uniform`.
            - Reset `AsyncCompactTicket`.
            - Return `true`.
        - Else return `false`.

7.  **Compressing (`TryCompress`)**:
    - If `Residency` is not `Resident`, return `false`.
    - If `bUniformValueValid` is `true` or `CompressedVoxelData.Num() > 0`:
        - Call `DropResidentForCachedForm()`.
    - Else:
        - Call `InstallCompactForm(ComputeCompactForm(GetResidentArray(), Codec, ChunkSize))`.

8.  **Computing Compact Form (`ComputeCompactForm`)**:
    - Input: `Data`, `Codec`, `ChunkSize`.
    - Check if all voxels are identical using `ComputeUniformValue`.
    - If true:
        - Set `Form.Kind` to `Uniform`.
        - Return `Form`.
    - Else:
        - Attempt compression using the general codec.
        - If compressed size < raw size (`Data.Num() * sizeof(FVoxelData)`):
            - Set `Form.Kind` to `Compressed`.
            - Return `Form` with buffer.
        - Else:
            - Return empty form (Incompressible).

9.  **Installing Compact Form (`InstallCompactForm`)**:
    - Input: `Form`, `bDropResident`.
    - If `Residency` is not `Resident`, return `false`.
    - Set `AsyncCompactTicket` to 0 and `bCompressionEvaluated` to `true`.
    - If `Kind` is `Uniform`:
        - Set `UniformValue` and `bUniformValueValid` to `true`.
    - If `Kind` is `Compressed`:
        - Move `Form.Buffer` to `CompressedVoxelData`.
        - Set `bDataMutated` to `false`.
    - If `bDropResident` is `true`, call `DropResidentForCachedForm`.

10. **Dropping Resident (`DropResidentForCachedForm`)**:
    - If `Residency` is not `Resident`, return `false`.
    - If `bUniformValueValid` is `true`:
        - Reset `VoxelArray`.
        - Set `Residency` to `Uniform`.
    - Else if `CompressedVoxelData.Num() > 0`:
        - Reset `VoxelArray`.
        - Set `Residency` to `Compressed`.
    - Else return `false`.
    - Set `AsyncCompactTicket` to 0.
    - Return `true` if residency was changed.

11. **Computing Uniform Value (`ComputeUniformValue`)**:
    - Input: `Data`, `Out`.
    - If empty, return `false`.
    - Cast data to `uint32` words.
    - If `sizeof(FVoxelData) == sizeof(uint32)`:
        - Loop in blocks of 64 words.
        - OR-reduce XOR differences against the first word.
        - If any difference found, return `false`.
        - Check remaining elements individually.
    - If all identical, write first element to `Out` and return `true`.

12. **Expanding (`Expand`)**:
    - If `Residency` is `Uniform`:
        - Initialize `Out` with `UniformValue` repeated `TotalVoxels` times.
        - Return `true`.
    - If `Residency` is `Compressed`:
        - Call `FVoxelChunkCodec::Decompress(Buffer, Out)`.
        - Return `Out.Num() == TotalVoxels`.
    - Else return `false`.

### Formulas

**Linear Index Calculation**
$$
Index = LocalPos.X + LocalPos.Y \times ChunkSize + LocalPos.Z \times ChunkSize \times ChunkSize
$$

**Inverse Index Calculation**
$$
X = Index \pmod{ChunkSize}
$$
$$
Y = (Index / ChunkSize) \pmod{ChunkSize}
$$
$$
Z = Index / (ChunkSize \times ChunkSize)
$$

**World-to-Local Voxel Conversion (Negative Coordinate Handling)**
For `WorldToLocalVoxel`:
If `LocalWorld.X < 0`, add `ChunkWorldSize`.
If `LocalWorld.Y < 0`, add `ChunkWorldSize`.
If `LocalWorld.Z < 0`, add `ChunkWorldSize`.

For `VoxelToLocal`:
If `Local.X < 0`, add `ChunkSize`.
If `Local.Y < 0`, add `ChunkSize`.
If `Local.Z < 0`, add `ChunkSize`.

### Parameters
- **ChunkSize**: Integer representing the edge length of a voxel chunk in voxels. Typical values: 32 or 64 [S1, unofficial].
- **VoxelSize**: Float representing world units per voxel [S1, unofficial].
- **Memory Footprint**: For a 32^3 chunk, voxel data only is ~128 KB [S1, unofficial].

### Edge Cases and Pitfalls
- **Negative Coordinates**: `WorldToLocalVoxel` and `VoxelToLocal` require modulo correction for negative coordinates to handle wrapping or boundary conditions correctly [S1, unofficial].
- **ContentVersion Bumping**: Must be bumped for logical changes (e.g., edit-layer changes) to invalidate caches, but NOT for `ClearVoxelData` (memory teardown only) [S1, unofficial].
- **Water-Flag Propagation**: Writes through `GetVoxelDataMutable()` do not bump `ContentVersion` because they are material changes, not density changes causing geometric boundary seams. Bumping would churn neighbor boundary revalidation for no geometric benefit [S1, unofficial].
- **Async Compression Validation**: Async compression results must be validated against `AsyncCompactTicket` to prevent installing stale compressed data after concurrent writes [S1, unofficial].
- **DetachVoxelArray**: Uses `IsUnique()` on the game thread. If a reader let go meanwhile, a needless copy occurs, but no reader can take the array again except from the game thread [S1, unofficial].

## Operating characteristics

- **Time Complexity**:
    - `GetVoxel`: O(1) for Uniform, O(1) + decompression time for Compressed, O(1) for Resident.
    - `SetVoxel`: O(1) amortized, O(N) if detachment or re-allocation is needed.
    - `TryCompress`: O(N) for uniformity check, O(N) for compression.
    - `ComputeUniformValue`: O(N) with word-aligned optimizations.
- **Memory Complexity**:
    - Resident: O(N) where N = ChunkSize^3.
    - Uniform: O(1) (stores one value).
    - Compressed: O(C) where C is compressed size, typically < N.
    - Empty: O(1).
- **Measured Performance**:
    - For a 32^3 chunk, raw data is ~128 KB. Compression can significantly reduce this for uniform or repetitive data [S1, unofficial].
    - Async compression is keyed on distance, allowing far chunks to be compressed without blocking the main thread [S1, unofficial].
- **Scaling**:
    - Scales well with chunk size; larger chunks benefit more from compression if data is uniform or compressible.
    - Distance-based streaming ensures only relevant chunks are kept in high-residency states.
- **Strengths**:
    - Reduces memory footprint for large worlds.
    - Asynchronous compression minimizes frame-time spikes.
    - Versioning ensures data consistency across concurrent accesses.
- **Limitations**:
    - Compression overhead for small or highly varied chunks.
    - Complexity in managing residency states and versioning.
    - Requires careful handling of negative coordinates and boundary conditions.
- **When to Use**:
    - Infinite voxel worlds with distance-based streaming.
    - Scenes with large uniform areas (e.g., sky, deep ocean).
    - When memory is a constraint and compression ratios are favorable.
- **When Not to Use**:
    - Small, static worlds where memory is not a concern.
    - Highly dynamic, non-uniform data where compression overhead outweighs benefits.

## Background

This method originates from the VoxelWorlds project by Daniel Raquel, implemented in C++ for Unreal Engine [S1, unofficial]. It is part of a broader system that includes streaming, hybrid rendering, procedural generation, editing, and materials. The residency management is specifically designed to support distance-based chunk loading and unloading, with async mesh generation and far-chunk voxel data compression [S1, unofficial]. The approach is documented in the VoxelWorlds GitHub repository [S1, unofficial].

## Sources

[S1] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
