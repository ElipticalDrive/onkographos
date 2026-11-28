# Tile-Based Neural Material Classification and Sorting

Also called: ClassificationA, ClassificationB, Mixed Tile Repacking, Tile-based classification

## Overview
This method is a two-pass compute shader pipeline that classifies 8x4 pixel tiles based on the neural material (MLP) index used by their pixels, separating them into uniform tiles (single MLP) and mixed tiles (multiple MLPs), then sorts mixed tiles to ensure uniform MLP usage per tile for efficient hardware matrix multiplication [S1]. It solves the problem of thread divergence in cooperative vector matrix multiplication by ensuring all pixels in a same workgroup use the same base matrix [S1]. Unlike [Coarse Raster](2807-coarse-raster.md), which focuses on early visibility culling, this method operates on the output of the Visibility pass to reorganize pixel data for the Lighting pass [S1].

## How it works
The method consists of two compute shader passes: `ClassificationA` and `ClassificationB` [S1].

### Inputs and Outputs
**Inputs:**
- Visibility buffer containing material IDs for each pixel [S1].
- Screen resolution (`_ScreenSize`) [S1].
- Tile dimensions (`_TileSize`), fixed at 8x4 pixels [S1].

**Outputs:**
- `_UniformTileAppendBuffer`: Append buffer for uniform tiles [S1].
- `_ComplexTileAppendBuffer`: Append buffer for mixed tiles [S1].
- `_ActiveTileAppendBuffer`: Append buffer for all active tiles [S1].
- `_IndexedTilesBufferRW`: Read-write buffer storing sorted pixel coordinates [S1].
- `_MLPUsageBufferRW`: Read-write buffer tracking MLP usage counts and offsets [S1].

### Data Structures
1.  **`_UniformTileAppendBuffer`**: Append buffer storing workgroup IDs of tiles where all valid pixels share the same MLP index [S1].
2.  **`_ComplexTileAppendBuffer`**: Append buffer storing workgroup IDs of tiles containing pixels with multiple different MLP indices [S1].
3.  **`_ActiveTileAppendBuffer`**: Append buffer storing workgroup IDs of all tiles with at least one valid pixel [S1].
4.  **`_ComplexTileConsumeBuffer`**: Consume buffer derived from `_ComplexTileAppendBuffer`, used by Pass 2 to process mixed tiles [S1].
5.  **`_MLPUsageBufferRW`**: A read-write buffer indexed by material ID (`matID`).
    -   Index `matID`: Stores the atomic usage count (number of pixels assigned to this MLP in the current batch) [S1].
    -   Index `UNIQUE_MLP_COUNT + matID`: Stores the group offset for the specific material ID [S1].
6.  **`_IndexedTilesBufferRW`**: A read-write buffer storing linearized pixel coordinates.
    -   Index: `tileGroupOffset * WORK_GROUP_SIZE + targetSlot` [S1].
    -   Value: Linearized screen coordinate `x + y * _ScreenSize.x` [S1].

### Parameters
-   **Tile Size**: 8x4 pixels, matching the workgroup size `[numthreads(8, 4, 1)]` [S1].
-   **Valid Pixel Threshold**: A tile is considered uniform only if it has more than 16 valid pixels (`validPixelsCount > 16`) [S1].
-   **WORK_GROUP_SIZE**: The size of a work group (32 threads), used as a multiplier for the tile group offset in the indexed tiles buffer [S1].
-   **UNIQUE_MLP_COUNT**: An offset index into `_MLPUsageBufferRW` used to retrieve the group offset for a specific material ID [S1].

### Steps

#### Pass 1: ClassificationA
This pass classifies each 8x4 tile (workgroup) based on the material IDs of its valid pixels.

1.  For each thread in the workgroup, retrieve the material ID for the current pixel using `visibility_buffer_mat_id(pixelCoords)` [S1].
2.  Check if the material ID is valid [S1].
3.  If valid, use wave-level intrinsics to find the minimum and maximum material IDs within the workgroup:
    -   `maxID = WaveActiveMax(matID)`
    -   `minID = WaveActiveMin(matID)` [S1]
4.  Count the number of valid pixels in the workgroup:
    -   `validPixelsCount = WaveActiveSum(1)` [S1]
5.  If the thread is the first lane in the wave (`WaveIsFirstLane()`):
    -   Calculate the flattened workgroup ID: `wgID = groupID.x + groupID.y * _TileSize.x` [S1].
    -   If `maxID == minID` and `validPixelsCount > 16`, append `wgID` to `_UniformTileAppendBuffer` [S1].
    -   Otherwise, append `wgID` to `_ComplexTileAppendBuffer` [S1].
    -   Always append `wgID` to `_ActiveTileAppendBuffer` [S1].

#### Pass 2: ClassificationB
This pass processes only the mixed tiles identified in Pass 1, sorting their pixels into uniform groups based on MLP index.

1.  Consume a workgroup ID from `_ComplexTileConsumeBuffer` [S1].
2.  Convert the workgroup ID to coordinates:
    -   `wgX = workGroupID % _TileSize.x`
    -   `wgY = workGroupID / _TileSize.x` [S1]
3.  Compute pixel coordinates for each thread in the 8x4 workgroup:
    -   `pixelCoords = uint2(wgX * 8 + groupThreadID.x, wgY * 4 + groupThreadID.y)` [S1]
4.  For each thread, retrieve the material ID: `matID = visibility_buffer_mat_id(pixelCoords)` [S1].
5.  If the material ID is valid:
    -   Atomically increment the usage count for that material ID in `_MLPUsageBufferRW` and book a slot:
        -   `InterlockedAdd(_MLPUsageBufferRW[matID], 1, targetSlot)` [S1]
    -   Retrieve the offset for the current MLP group from `_MLPUsageBufferRW`:
        -   `mlpGroupOffset = _MLPUsageBufferRW[UNIQUE_MLP_COUNT + matID]` [S1]
    -   Store the pixel's linearized coordinate into `_IndexedTilesBufferRW` at the calculated offset:
        -   `_IndexedTilesBufferRW[mlpGroupOffset * WORK_GROUP_SIZE + targetSlot] = pixelCoords.x + pixelCoords.y * _ScreenSize.x` [S1]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: ClassificationA
[numthreads(8, 4, 1)]
def ClassificationA(groupID: uint2, pixelCoords: uint2):
    # Get the material ID of the pixel
    matID = visibility_buffer_mat_id(pixelCoords)
    
    if valid_mat_id(matID):
        # Find min and max material IDs within the workgroup
        maxID = WaveActiveMax(matID)
        minID = WaveActiveMin(matID)
        
        # Count valid pixels in the workgroup
        validPixelsCount = WaveActiveSum(1)
        
        # Flatten the work group ID
        wgID = groupID.x + groupID.y * _TileSize.x
        
        # Only the first lane classifies the tile
        if WaveIsFirstLane():
            # Append to active tiles list always
            _ActiveTileAppendBuffer.Append(wgID)
            
            # Classify as uniform or complex
            if maxID == minID and validPixelsCount > 16:
                _UniformTileAppendBuffer.Append(wgID)
            else:
                _ComplexTileAppendBuffer.Append(wgID)

# Pass 2: ClassificationB
[numthreads(8, 4, 1)]
def ClassificationB(groupThreadID: uint2):
    # Consume a workgroup ID from the complex tile buffer
    workGroupID = _ComplexTileConsumeBuffer.Consume()
    
    # Convert workgroup ID to coordinates
    wgX = workGroupID % _TileSize.x
    wgY = workGroupID / _TileSize.x
    
    # Compute pixel coordinates for this thread
    pixelCoords = uint2(wgX * 8 + groupThreadID.x, wgY * 4 + groupThreadID.y)
    
    # Get material ID
    matID = visibility_buffer_mat_id(pixelCoords)
    
    if valid_mat_id(matID):
        # Book a slot for this pixel in the right tile group using atomic add
        targetSlot = InterlockedAdd(_MLPUsageBufferRW[matID], 1)
        
        # Get the group offset for this material ID
        mlpGroupOffset = _MLPUsageBufferRW[UNIQUE_MLP_COUNT + matID]
        
        # Store the linearized pixel coordinate
        _IndexedTilesBufferRW[mlpGroupOffset * WORK_GROUP_SIZE + targetSlot] = \
            pixelCoords.x + pixelCoords.y * _ScreenSize.x
```

### Edge Cases and Pitfalls
-   **Global Atomics**: The method uses global atomics for sorting mixed tiles. This is inefficient if the screen is filled with mixed tiles; shared memory atomics would be more suitable in that case [S1].
-   **Uniform Tile Threshold**: A tile is only classified as uniform if it has more than 16 valid pixels. Tiles with fewer valid pixels, even if they share a single MLP, are treated as complex tiles [S1].
-   **MLP Sharing**: The current implementation does not share unique MLPs for different materials. Sharing MLPs could reduce the number of mixed tiles and improve performance at the expense of quality [S1].

## Operating characteristics
-   **Time Complexity**: The method requires two passes over the tiles. Pass 1 is O(N) where N is the number of tiles. Pass 2 is O(M) where M is the number of mixed tiles.
-   **Memory Complexity**: Requires additional buffers for classification (`_UniformTileAppendBuffer`, `_ComplexTileAppendBuffer`, `_ActiveTileAppendBuffer`) and sorting (`_MLPUsageBufferRW`, `_IndexedTilesBufferRW`) [S1].
-   **Performance**: When using wider MLPs (D >= 32), using cooperative vectors permits up to a ×40 speedup compared to software FMA [S1]. The classification itself takes approximately 0.035 ms for L=1 layer and D=16, scaling with MLP dimensions [S1].
-   **Strengths**: Avoids divergence in the Lighting pass by ensuring all pixels in a workgroup use the same MLP, which is required for efficient cooperative vector matrix multiplication [S1].
-   **Limitations**: Scaling issue: The classification is not optimal for screens filled with mixed tiles due to global atomics [S1].

## Background
This method was introduced by L. Belcour & A. Benyoub in the paper "Hardware Accelerated Neural Block Texture Compression with Cooperative Vectors" (Eurographics 2025) [S1]. It is part of the HW NBTC pipeline, designed to handle neural material compression and rendering efficiently on hardware supporting cooperative vectors [S1].

## Sources
[S1] Hardware Accelerated Neural Block Texture Compression with Cooperative Vectors, https://doi.org/10.2312/hpg20251173, fetched 2026-11-06
