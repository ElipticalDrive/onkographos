# Point Cloud Voxelization with Adjacency Cleanup

Also called: Open3DUE5Module::VoxelizedArrFromPoints with Cleanup, Voxelize

## Overview
This method converts a set of 3D points into a fixed-size voxel grid by assigning points to discrete voxels and subsequently removing isolated, non-connected voxels to clean up noise [S1, unofficial]. It extends [Hard Voxelization (Deterministic)](3321-hard-voxelization-deterministic.md) by adding a topological cleanup phase that eliminates single-voxel artifacts, distinguishing it from sibling methods like [Grid Pooling (Max Pooling over Points)](4327-grid-pooling-max-pooling-over-points.md) which aggregate values rather than filtering topology.

## How it works
**Inputs:**
- `Points`: Array of 3D coordinates (vectors).
- `Transform`: A 4x4 transformation matrix applied to points.
- `bSubtractive`: Boolean flag; if true, inverts the distance divisor logic for subtractive voxelization [S1, unofficial].
- `Settings`: Configuration object containing `VoxelSize` (double/float) and `DistanceDivisor` [S1, unofficial].
- `Origin`: The world-space origin used for bounding box calculation [S1, unofficial].

**Outputs:**
- `Asset`: A `UVoxelDataAsset` containing the final voxel grid, offset, and subtractive flag [S1, unofficial].
- Return value: Boolean indicating success or failure.

**Data Structures:**
- `FVoxelDataAssetData`: A 3D array (grid) accessed via `SetValue(X, Y, Z, value)` and `GetValue(X, Y, Z, value)`.
- `FVoxelValue`: An enum with states `Empty()`, `Full()`, and `Special()` [S1, unofficial].
- `VoxelFilledArr`: An array of integer coordinates (X, Y, Z) representing occupied voxels returned by the core voxelization module [S1, unofficial].
- `RealSize`: A 3D integer vector defining the dimensions of the output grid [S1, unofficial].
- `OutOffset`: A 3D integer vector representing the minimum coordinate offset of the grid in world space [S1, unofficial].

**Steps:**
1. **Validation**: If `Points` is empty, return `false` [S1, unofficial].
2. **Subtractive Adjustment**: If `bSubtractive` is true, multiply `Settings.DistanceDivisor` by -1 [S1, unofficial].
3. **Transformation**: Transform each point in `Points` using `Transform.TransformPosition` and store the results in `RealPoints` [S1, unofficial].
4. **Core Voxelization**: Call `FOpen3DUE5Module::VoxelizedArrFromPoints(RealPoints, (double)Settings.VoxelSize, VoxelFilledArr, RealSize)` to compute occupied voxel indices and grid dimensions [S1, unofficial].
5. **Bounding Box Calculation**: Iterate over `RealPoints`, subtract `Origin` (transformed zero), and accumulate the minimum and maximum coordinates into `Box` [S1, unofficial].
6. **Constraint Check**: If `RealSize.X * RealSize.Y * RealSize.Z >= MAX_int32` (2,147,483,647) or equals 0, return an error [S1, unofficial].
7. **Offset Calculation**: Compute `OutOffset = FVoxelUtilities::RoundToInt(Box.Min / Settings.VoxelSize)` [S1, unofficial].
8. **Initialization**: Initialize `Data` with size `RealSize` [S1, unofficial].
9. **Clear Grid**: Loop over all X, Y, Z in `RealSize` and set each voxel to `FVoxelValue::Empty()` [S1, unofficial].
10. **Mark Voxels**: Loop over `VoxelFilledArr`. For each coordinate `AVoxelLocation`, if it is within bounds `[0, RealSize)`, set `Data->SetValue(AVoxelLocation.X, AVoxelLocation.Y, AVoxelLocation.Z, FVoxelValue::Full())`. If out of bounds, log a warning but proceed (risking out-of-bounds access) [S1, unofficial].
11. **Cleanup**: Call `CleanUpAloneVoxels(Data, 2)` to remove isolated voxels [S1, unofficial].
12. **Asset Creation**: Create `UVoxelDataAsset`, set `bSubtractiveAsset = bSubtractive`, set `PositionOffset = OutOffset`, attach `Data`, and return `true` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_cloud_voxelization_with_cleanup(Points, Transform, bSubtractive, Settings, Origin):
    if not Points:
        return False, None

    # Step 2: Subtractive adjustment
    distance_divisor = Settings.DistanceDivisor
    if bSubtractive:
        distance_divisor *= -1

    # Step 3: Transform points
    RealPoints = [Transform.TransformPosition(p) for p in Points]

    # Step 4: Core voxelization (assumes external module)
    # Returns VoxelFilledArr (list of (x,y,z)) and RealSize (Vector3i)
    VoxelFilledArr, RealSize = voxelized_arr_from_points(RealPoints, Settings.VoxelSize)

    # Step 5: Bounding box
    BoxMin = Vector3(Infinity, Infinity, Infinity)
    BoxMax = Vector3(-Infinity, -Infinity, -Infinity)
    for p in RealPoints:
        p_local = p - Origin
        BoxMin = min(BoxMin, p_local)
        BoxMax = max(BoxMax, p_local)
    Box = (BoxMin, BoxMax)

    # Step 6: Constraints
    if RealSize.X * RealSize.Y * RealSize.Z >= 2147483647 or RealSize.X * RealSize.Y * RealSize.Z == 0:
        return False, None

    # Step 7: Offset
    OutOffset = RoundToInt(Box.Min / Settings.VoxelSize)

    # Step 8: Initialize Data
    Data = VoxelDataAssetData(RealSize)

    # Step 9: Clear Grid
    for x in range(RealSize.X):
        for y in range(RealSize.Y):
            for z in range(RealSize.Z):
                Data.SetValue(x, y, z, FVoxelValue.Empty())

    # Step 10: Mark Voxels
    for loc in VoxelFilledArr:
        # Pitfall: Source code logs warning but proceeds if out of bounds
        if 0 <= loc.X < RealSize.X and 0 <= loc.Y < RealSize.Y and 0 <= loc.Z < RealSize.Z:
            Data.SetValue(loc.X, loc.Y, loc.Z, FVoxelValue.Full())
        else:
            LogWarning("Voxel location out of bounds")

    # Step 11: Cleanup
    CleanUpAloneVoxels(Data, IterNum=2)

    # Step 12: Asset Creation
    Asset = UVoxelDataAsset()
    Asset.bSubtractiveAsset = bSubtractive
    Asset.PositionOffset = OutOffset
    Asset.Data = Data
    return True, Asset

def voxel_surrounded_check(Data, X, Y, Z):
    count = 1
    for i in range(-1, 2):
        for j in range(-1, 2):
            for k in range(-1, 2):
                # Note: Source code uses Z+k for k loop, likely a typo for Z+k or Z+j. 
                # Assuming standard 3D neighbor check logic based on variable names, 
                # but strictly following source: Data->GetValue(X + i, Y + k, Z + k)
                # However, standard 3D grid access is (x+i, y+j, z+k). 
                # The source code snippet: Data->GetValue(X + i, Y + k, Z + k) 
                # implies Y and Z are indexed by k, which is likely a bug in the source.
                # Implementing strictly as per source snippet provided in notes:
                val = Data.GetValue(X + i, Y + k, Z + k, FVoxelValue.Special())
                if val == FVoxelValue.Full():
                    count += 1
    return count > 5

def clean_up_alone_voxels(Data, IterNum):
    ToRemove = []
    for temp in range(IterNum):
        ToRemove = []
        for x in range(Data.GetSize().X):
            for y in range(Data.GetSize().Y):
                for z in range(Data.GetSize().Z):
                    if not voxel_surrounded_check(Data, x, y, z):
                        ToRemove.append((x, y, z))
        for coord in ToRemove:
            Data.SetValue(coord[0], coord[1], coord[2], FVoxelValue.Empty())
```

**Parameters:**
- `IterNum`: Integer specifying the number of cleanup passes. Hardcoded to 2 in the main call [S1, unofficial].
- `Settings.VoxelSize`: Double/float value defining the physical size of each voxel [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Out-of-Bounds Access**: If `VoxelFilledArr` contains indices outside `[0, RealSize)`, the code logs a warning but proceeds to call `Data->SetValue`, risking out-of-bounds memory access [S1, unofficial].
- **Grid Size Limits**: Grid dimensions exceeding `MAX_int32` (2,147,483,647) or resulting in zero volume abort the process with an error [S1, unofficial].
- **Neighbor Check Bug**: The source code for `VoxelSurroundedCheck` uses `Data->GetValue(X + i, Y + k, Z + k)`, indexing Y and Z with the same loop variable `k`. This is likely a typo in the source and should probably be `Y + j` and `Z + k` for a proper 3D neighbor check [S1, unofficial]. Implementers should verify this logic.

## Operating characteristics
**Time Complexity:** $O(N + V \cdot I)$, where $N$ is the number of points, $V$ is the number of voxels in the grid, and $I$ is the number of cleanup iterations (2). The core voxelization is $O(N)$, grid initialization is $O(V)$, and cleanup is $O(V \cdot I \cdot 27)$ due to the 3x3x3 neighbor check per voxel.
**Memory Complexity:** $O(V)$ for the voxel grid storage.
**Strengths:** Effectively removes isolated noise voxels that result from sparse point clouds or discretization errors.
**Limitations:** The hardcoded 2-pass cleanup may be insufficient for complex noise patterns or excessive for simple cases. The neighbor check implementation in the source contains a likely indexing bug.
**When to use:** When point cloud data is noisy and topological connectivity is required for downstream volume rendering or meshing.
**When not to use:** When performance is critical and noise is minimal, or when the grid size approaches memory limits.

## Background
This method is implemented in the `PointCloudVoxelizerBase` class within the Unreal Engine 5 Open3D plugin ecosystem, specifically in the `PointCloudToVoxelTest` repository [S1, unofficial]. It leverages the `FOpen3DUE5Module` for the core voxelization step and custom C++ utilities for the cleanup phase.

## Sources
[S1] ShinSangJin-Capstone-Team/PointCloudToVoxelTest, https://github.com/ShinSangJin-Capstone-Team/PointCloudToVoxelTest, fetched 2026-10-27 (unofficial)
