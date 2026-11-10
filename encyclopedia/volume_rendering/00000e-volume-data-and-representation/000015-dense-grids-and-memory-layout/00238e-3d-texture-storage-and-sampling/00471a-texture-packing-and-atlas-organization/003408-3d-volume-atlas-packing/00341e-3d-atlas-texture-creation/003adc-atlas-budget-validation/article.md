# Atlas Budget Validation

## Overview
Atlas Budget Validation ensures that the coarsest pyramid level's brick grid fits within the GPU's available VRAM budget, preventing out-of-memory (OOM) crashes during initialization, particularly on mobile GPUs [S1, unofficial]. It acts as a gatekeeper before texture allocation, calculating the required memory for the base level and comparing it against a configurable limit [S1, unofficial].
## How it works
**Inputs:**
- `metadata`: Object containing volume hierarchy.
  - `metadata.maxLod`: Integer, the highest level of detail index.
  - `metadata.levels`: Array of level objects. Each level object contains:
    - `lod`: Integer, the level index.
    - `brickGrid`: Object representing the 3D grid of bricks for that level. The product of its X, Y, and Z dimensions gives the total brick count [S1, unofficial].
- `textureFormat`: String, the format of the volume data.
- `numChannels`: Integer, number of channels per voxel.
- `atlasBudgetBytes`: Integer, the maximum VRAM budget in bytes allocated for the atlas [S1, unofficial].

**Outputs:**
- Success: Validation passes, allowing allocation.
- Failure: Throws `UnsupportedDatasetError` with details if the budget is exceeded [S1, unofficial].

**Data Structures:**
The validation relies on the `brickGrid` dimensions from the metadata [S1, unofficial]. The `brickGrid` is a 3D grid where the total number of bricks is the product of its X, Y, and Z dimensions [S1, unofficial].

**Steps:**
1. Calculate `bytesPerVoxel`: 1 if `textureFormat` is `'r8unorm'`, else 2 [S1, unofficial].
2. Compute `totalSlots` by calling `computeAtlasGrid(numChannels, bytesPerVoxel, atlasBudgetBytes)` and taking the result's `gridSize` cubed [S1, unofficial].
3. Retrieve `baseBricks` from the level where `level.lod === metadata.maxLod` by taking the product of its `brickGrid` dimensions [S1, unofficial].
4. Set `workingSlots` to 1 if `metadata.maxLod > 0`, else 0 [S1, unofficial].
5. If `baseBricks + workingSlots > totalSlots`, throw `UnsupportedDatasetError` with details [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class UnsupportedDatasetError(Exception):
    pass

def compute_atlas_grid(num_channels, bytes_per_voxel, atlas_budget_bytes):
    # Finds the maximum integer gridSize such that gridSize^3 * num_channels * bytes_per_voxel <= atlas_budget_bytes
    # Returns an object with a 'gridSize' field
    budget_per_voxel = num_channels * bytes_per_voxel
    if budget_per_voxel <= 0:
        return {"gridSize": 0}
    gridSize = int((atlas_budget_bytes / budget_per_voxel) ** (1/3))
    while (gridSize + 1) ** 3 * budget_per_voxel <= atlas_budget_bytes:
        gridSize += 1
    return {"gridSize": gridSize}

def validate_atlas_budget(metadata, texture_format, num_channels, atlas_budget_bytes):
    # Step 1: Calculate bytes per voxel
    bytes_per_voxel = 1 if texture_format == 'r8unorm' else 2

    # Step 2: Calculate total atlas slots
    atlas_grid_info = compute_atlas_grid(num_channels, bytes_per_voxel, atlas_budget_bytes)
    total_slots = atlas_grid_info["gridSize"] ** 3

    # Step 3: Retrieve base bricks from the coarsest level (maxLod)
    max_lod = metadata["maxLod"]
    coarsest_level = None
    for level in metadata["levels"]:
        if level["lod"] == max_lod:
            coarsest_level = level
            break

    if coarsest_level is None:
        raise ValueError("Coarsest level not found in metadata")

    # brickGrid product is the total number of bricks in the 3D grid
    base_bricks = coarsest_level["brickGrid"]["product"]

    # Step 4: Calculate working slots
    working_slots = 1 if max_lod > 0 else 0

    # Step 5: Validate
    if base_bricks + working_slots > total_slots:
        raise UnsupportedDatasetError(
            f"Atlas budget exceeded. Required: {base_bricks + working_slots} slots, "
            f"Available: {total_slots} slots."
        )
```

**Parameters:**
- `atlasBudgetBytes`: Maximum VRAM budget for the atlas in bytes. The default is ~1.3 GiB, lower for mobile GPUs, and raised to keep a $660^3$ grid for many channels [S1, unofficial].

**Edge Cases and Pitfalls:**
- Mobile GPUs are prone to OOM crashes at startup if the atlas is too large, mitigated by shrinking the atlas grid based on channel count and budget before allocation [S1, unofficial].

## Subtopics

### Atlas VRAM Budget Configuration
Configures the maximum VRAM allocated for the volume atlas, allowing the grid to shrink to fit constrained hardware [S1, unofficial]. It involves passing `atlasBudgetBytes` to `GpuSetup` and `DatasetConfig` to limit atlas memory usage [S1, unofficial]. The default budget is ~1.3 GiB. Lower values are recommended for mobile GPUs, while higher values should be raised to maintain a $660^3$ grid for datasets with many channels [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Prevents fatal OOM errors on resource-constrained devices (especially mobile GPUs) by validating memory requirements before allocation [S1, unofficial].
- **Limitations:** Requires accurate estimation of `atlasBudgetBytes`. Not given in the sources.
- **When to use:** Always before allocating the 3D atlas texture, especially in cross-platform or mobile-aware renderers [S1, unofficial].

## Background
The method addresses the challenge of managing VRAM budgets on diverse hardware, including mobile GPUs, which have significantly lower memory constraints than desktop GPUs [S1, unofficial]. Origin and development history are not given in the sources.

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
