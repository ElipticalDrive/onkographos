# Froxel Light List Buffer Creation

Also called: Froxel Light List Buffer Allocation

## Overview
This method constructs the data structures required to store the sparse mapping of lights to froxels (frustum voxels). It allocates the buffers for per-froxel light counts, prefix-sum offsets, and the global light index array, then executes compute shader passes to populate these buffers. It differs from [Froxel Light Assignment](13184-froxel-light-assignment.md) by focusing on the memory layout and the two-pass reduction/scan algorithm (counting and offset calculation) rather than the geometric intersection logic.

## How it works
### Inputs and Outputs
**Inputs:**
- `Grid`: 3D grid dimensions $(X, Y, Z)$ defining the froxel resolution [S1, unofficial].
- `NumLights_RT`: Number of real-time lights to consider [S1, unofficial].
- `MaxLightsPerFroxel`: Maximum number of lights any single froxel can hold [S1, unofficial].
- `LightsSB`: Structured buffer containing light data [S1, unofficial].
- `SharedUB`: Uniform buffer containing froxel grid parameters [S1, unofficial].
- `View`: View uniform buffer containing camera/projection data [S1, unofficial].

**Outputs:**
- `FroxelLists`: A container holding four buffers:
  1. `Counts`: Per-froxel light count.
  2. `Offsets`: Per-froxel start index in the global light list.
  3. `TotalIndices`: A single value representing the total number of valid light-froxel pairs.
  4. `LightIndices`: The global array of light indices.

### Data Structures and Layout
The `FroxelLists` structure contains the following buffers, all created with `BUF_ShaderResource | BUF_UnorderedAccess` flags [S1, unofficial]:

1.  **Counts**: `RWStructuredBuffer<uint32>`
    -   Size: `FroxelCount` elements.
    -   `FroxelCount = Grid.X * Grid.Y * Grid.Z` [S1, unofficial].
2.  **Offsets**: `RWStructuredBuffer<uint32>`
    -   Size: `FroxelCount` elements [S1, unofficial].
3.  **TotalIndices**: `RWStructuredBuffer<uint32>`
    -   Size: 1 element [S1, unofficial].
4.  **LightIndices**: `RWStructuredBuffer<uint32>`
    -   Size: `FroxelCount * MaxLightsPerFroxel` elements [S1, unofficial].

### Algorithm Steps

#### 1. Buffer Allocation
Allocate the four buffers described above with the exact sizes calculated from the grid dimensions and light limits [S1, unofficial].

#### 2. Light Counting Pass
Compute the number of lights intersecting each froxel.
1.  Clear `Counts` buffer to 0 [S1, unofficial].
2.  Execute `FFroxelCountCS` (or equivalent):
    -   **Group Count**: `FComputeShaderUtils::GetGroupCount(NumLights_RT, 64)` [S1, unofficial].
    -   **Parameters**: `SharedUB`, `View`, `LightsSB`, `Counts` UAV [S1, unofficial].
    -   **Logic**: Each work group processes a subset of lights. For each light, iterate through froxels (or vice versa, depending on implementation) and increment the count for intersecting froxels using atomic operations or thread-local accumulation followed by atomic add [S1, unofficial].

#### 3. Total Indices Reduction
Compute the sum of all counts to determine the size of the populated light list.
1.  Clear `TotalIndices` buffer to 0 [S1, unofficial].
2.  Execute `FFroxelTotalIndicesCS`:
    -   **Group Count**: `FIntVector(1, 1, 1)` [S1, unofficial].
    -   **Logic**: Perform a parallel reduction (e.g., using shared memory or atomic adds) on the `Counts` buffer to compute the global sum. Store the result in `TotalIndices` [S1, unofficial].

#### 4. Light Index Assignment
Assign specific light indices to each froxel.
1.  Clear `LightIndices` buffer to 0 [S1, unofficial].
2.  Reuse `Counts` buffer as `FroxelHeads` (atomic head pointers) and clear it to 0 [S1, unofficial].
3.  Execute `FFroxelAssignLightsCS`:
    -   **Group Count**: `FComputeShaderUtils::GetGroupCount(NumLights_RT, 64)` [S1, unofficial].
    -   **Parameters**: `SharedUB`, `View`, `LightsSB`, `LightIndices` UAV, `FroxelHeads` UAV (reusing `Counts`) [S1, unofficial].
    -   **Logic**: For each light, determine intersecting froxels. For each intersecting froxel, atomically increment the head pointer in `FroxelHeads` to get a unique index within the froxel's local list. Store the light index in `LightIndices` at the calculated position [S1, unofficial].

#### 5. Offset Calculation (Prefix Sum)
Convert per-froxel counts into global offsets for the `LightIndices` buffer. This is done in three passes to handle the 3D grid structure efficiently [S1, unofficial]:

1.  **Pass A: Intra-Slice Scan**
    -   Allocate temporary buffers `ZSliceTotals` (size `GZ`) and `ZSliceOffsets` (size `GZ`) [S1, unofficial].
    -   Clear `Offsets` and `ZSliceTotals` to 0 [S1, unofficial].
    -   Execute `FFroxelOffsetPassACS`:
        -   **Logic**: Perform a Blelloch scan (inclusive or exclusive prefix sum) on the `Counts` buffer for each Z-slice (a 2D plane of size $GX \times GY$). Store the local offsets in `Offsets` and the sum of each slice in `ZSliceTotals` [S1, unofficial].

2.  **Pass B: Inter-Slice Scan**
    -   Execute a prefix sum on the `ZSliceTotals` buffer to compute the cumulative offset for each Z-slice. Store results in `ZSliceOffsets` [S1, unofficial].

3.  **Pass C: Final Offset Adjustment**
    -   Execute `FFroxelOffsetPassBCS` (or equivalent):
        -   **Logic**: Add the corresponding Z-slice offset (from `ZSliceOffsets`) to each froxel's local offset (from `Offsets`). This produces the final global start index for each froxel in the `LightIndices` buffer [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
FroxelCount = Grid.X * Grid.Y * Grid.Z
MaxLightsPerFroxel = 64 # Example value
NumLights_RT = 100 # Example value

# 1. Allocation
Counts = RWStructuredBuffer<uint32>(size=FroxelCount)
Offsets = RWStructuredBuffer<uint32>(size=FroxelCount)
TotalIndices = RWStructuredBuffer<uint32>(size=1)
LightIndices = RWStructuredBuffer<uint32>(size=FroxelCount * MaxLightsPerFroxel)

# 2. Counting Pass
Clear(Counts, 0)
Dispatch(FFroxelCountCS, groups=GetGroupCount(NumLights_RT, 64), 
         params={SharedUB, View, LightsSB, Counts})

# 3. Total Indices Reduction
Clear(TotalIndices, 0)
Dispatch(FFroxelTotalIndicesCS, groups=(1,1,1), 
         params={Counts, TotalIndices})

# 4. Light Index Assignment
Clear(LightIndices, 0)
Clear(Counts, 0) # Reuse as Heads
Dispatch(FFroxelAssignLightsCS, groups=GetGroupCount(NumLights_RT, 64), 
         params={SharedUB, View, LightsSB, LightIndices, Counts})

# 5. Offset Calculation
ZSliceSize = Grid.X * Grid.Y
ZSliceTotals = Buffer<uint32>(size=Grid.Z)
ZSliceOffsets = Buffer<uint32>(size=Grid.Z)

Clear(Offsets, 0)
Clear(ZSliceTotals, 0)

# Pass A: Intra-slice scan
Dispatch(FFroxelOffsetPassACS, groups=..., 
         params={Counts, Offsets, ZSliceTotals})

# Pass B: Inter-slice scan on ZSliceTotals
# (Implementation detail: standard prefix sum on ZSliceTotals)
PrefixSum(ZSliceTotals, ZSliceOffsets)

# Pass C: Add slice offsets to local offsets
Dispatch(FFroxelOffsetPassBCS, groups=..., 
         params={Counts, Offsets, ZSliceOffsets})
```

### Edge Cases and Pitfalls
-   **Zero Lights**: If `NumLights_RT` is 0, all counts and offsets should be 0. The reduction pass must handle empty inputs gracefully [S1, unofficial].
-   **Overflow**: `FroxelCount * MaxLightsPerFroxel` must fit in `uint32`. If the scene has many froxels and lights, this may require 64-bit indices or a different data structure [S1, unofficial].
-   **Atomic Contention**: The light assignment pass uses atomic operations on `Counts` (as heads). High contention may reduce performance. Using a two-pass approach (count then assign) mitigates this by separating the counting logic from the index assignment [S1, unofficial].
-   **Precision**: Prefix sum operations must be exact to avoid overlapping or gaps in the light list. Use integer arithmetic [S1, unofficial].

## Subtopics

### Froxel Light Index Assignment
Computes the list of light indices for each froxel, storing results in `FroxelLists.LightIndices`. It first computes the total number of indices using a reduction pass, then assigns indices using an atomic counter. It reuses the `Counts` buffer as `FroxelHeads` for atomic head pointers during assignment [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: $O(N_L + N_F)$, where $N_L$ is the number of lights and $N_F$ is the number of froxels. The counting and assignment passes are proportional to $N_L$, and the prefix sum is proportional to $N_F$ [S1, unofficial].
-   **Memory Complexity**: $O(N_F \cdot M + N_F + 1)$, where $M$ is `MaxLightsPerFroxel`. The dominant buffer is `LightIndices` [S1, unofficial].
-   **Performance**: The use of compute shaders allows parallel processing of lights and froxels. The three-pass offset calculation is efficient for large grids as it limits shared memory usage per group [S1, unofficial].
-   **Strengths**: Scalable to large numbers of lights and froxels. Decouples light assignment from offset calculation.
-   **Limitations**: Requires significant temporary memory for the prefix sum passes. Atomic operations in the assignment pass can cause contention.

## Background
This method is part of the Unreal Engine 5 Froxel Lighting system, an unofficial implementation by Armin Tajik. It builds on the concept of froxel clustering for real-time volumetric lighting. The approach uses standard GPU compute techniques (atomic counters, prefix sums) to build sparse data structures for light transport approximation.

## Sources
[S1] ArminTajik/ue5-froxel-lighting, https://github.com/ArminTajik/ue5-froxel-lighting, fetched 2026-10-27 (unofficial)
