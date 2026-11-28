# OpenVDB Import via Python

Also called: OpenVDB Python Import, VDB Import

## Overview
This method describes the process of converting seismic SEG-Y data into the OpenVDB format using Python, and subsequently importing that VDB file into a rendering engine (specifically Unreal Engine 5) for sparse voxel texture (SVT) rendering. It addresses the problem of handling massive datasets (e.g., 4.4 GB) that exceed standard 32-bit integer limits for memory addressing and buffer sizes, which causes import crashes and rendering artifacts. It is distinguished from generic VDB loading by its specific focus on the Python-side conversion pipeline and the subsequent engine-side integer overflow fixes required for large-scale sparse volumes.

## How it works
The process involves two distinct phases: Python-based conversion to VDB and engine-side import with overflow corrections.

### Phase 1: Python Conversion
**Inputs:** SEG-Y formatted seismic data files.
**Outputs:** OpenVDB 14 file (e.g., `Kolumbo.vdb`).
**Data Structures:**
*   **SEG-Y Data:** Raw seismic traces.
*   **OpenVDB Grid:** Sparse voxel grid using hierarchical tree structures (Octree).
*   **Precision:** 8-bit per voxel value is specified for the target dataset [S1, unofficial].

**Steps:**
1.  Initialize a Python environment with `segyio` for SEG-Y parsing and OpenVDB Python bindings [S1, unofficial].
2.  Load the SEG-Y data using `segyio`.
3.  Map the seismic data to voxel coordinates.
4.  Construct the OpenVDB grid, ensuring sparse storage (occupancy bits) to handle non-empty voxels efficiently [S1, unofficial].
5.  Save the grid to an OpenVDB 14 file [S1, unofficial].

**Pitfalls:**
*   The resulting VDB file may be large (e.g., 4.4 GB). Direct loading without chunking or overflow handling causes crashes [S1, unofficial].

### Phase 2: Engine Import and Overflow Fixes
**Inputs:** OpenVDB 14 file.
**Outputs:** Rendered Sparse Volume Texture (SVT) in the engine.
**Data Structures:**
*   **CPU Representation:** Internal compression scheme using occupancy bits. Stores only non-empty voxels (~2.6 billion in the Kolumbo dataset) [S1, unofficial].
*   **GPU Upload Buffer:** Contains compressed tile data padded for interpolation and mip levels [S1, unofficial].

**Steps:**
1.  **Chunking:** If the initial import crashes, chunk the VDB data into at least four chunks. Previous approaches required nine chunks, but four is sufficient for this specific dataset [S1, unofficial].
2.  **CPU Index Type Fix:** Identify integer types in the import function `SVT::FResources::CompressTiles(...)` that cause access violations. Change index types from `int32` (signed) to `uint32` (unsigned) to support the full range of non-empty voxel counts (~2.6 billion) [S1, unofficial].
3.  **GPU Upload Buffer Size Fix:** Identify integer overflows in the upload buffer calculations. Modify the return type of the Render Hardware Interface (RHI) function to `uint64`:
    ```python
    # Pseudocode for the RHI function signature fix
    def RHIComputeMemorySize(TextureRHI):
        # Original: virtual uint32 ...
        # Fixed:
        return uint64(computed_size)
    ```
    [S1, unofficial]
4.  **Verify Buffer Size:** Ensure the total upload buffer size does not exceed $2^{32}$ bytes. The calculation is:
    $$ \text{Total Size} = N_{voxels} \times \text{Padding Factor} \times \text{Mip Factor} $$
    Where:
    *   $N_{voxels} = 2.6 \times 10^9$ bytes (compressed size of non-empty voxels) [S1, unofficial].
    *   $\text{Padding Factor} = (18/16)^3 \approx 1.424$ (padding $16^3$ tiles to $18^3$ voxels for trilinear interpolation) [S1, unofficial].
    *   $\text{Mip Factor} = \sum_{n=0}^{\infty} (1/2)^{3n} = \frac{1}{1 - (1/2)^3} \approx 1.143$ (sum of all 3D mip levels) [S1, unofficial].
    *   $\text{Total Size} \approx 2.6 \times 10^9 \times 1.424 \times 1.143 = 4.2318432 \times 10^9$ bytes [S1, unofficial].
    *   Limit: $2^{32} = 4.294 \times 10^9$ bytes [S1, unofficial].
    Since $4.2318432 \times 10^9 < 4.294 \times 10^9$, the buffer fits in a 32-bit address space if `uint32` is used correctly, but intermediate calculations or other indices may require `uint64` to prevent overflow during computation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Rendering Artifacts:** After fixing the CPU import crash, lower parts of the dataset may still display incorrectly due to a second `int32` overflow during GPU upload. Approximately 2.1 billion of 2.6 billion voxels upload correctly, leaving one-fifth incorrect [S1, unofficial].
*   **Shader Types:** The compute shader (`UpdateSparseVolumeTexture.usf`) uses `uint32` for upload, expecting 4GB support. The overflow is on the CPU side in the RHI calculation [S1, unofficial].
*   **Performance:** Correct implementation achieves ~20 fps for the Kolumbo dataset without artifacts [S1, unofficial].

## Subtopics

### GPU Upload Buffer Overflow Fix
A specific variant of the import process focusing on the GPU-side memory allocation. It addresses the issue where the upload buffer size calculation overflows a 32-bit integer, causing incorrect indexing of lower tiles. It requires modifying the RHI function `RHIComputeMemorySize` to return `uint64` and ensuring the total buffer size ($4.23 \times 10^9$ bytes) is correctly computed and allocated. This fix is critical for datasets with ~2.6 billion non-empty voxels where the padded and mip-mapped size approaches the 4GB limit [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly given, but rendering performance is measured at ~20 fps for the full Kolumbo dataset [S1, unofficial].
*   **Memory Complexity:** The dataset uses sparse storage with occupancy bits, storing only non-empty voxels (~2.6 billion) [S1, unofficial]. The upload buffer size is ~4.23 GB [S1, unofficial].
*   **Performance:** ~20 fps for the Kolumbo dataset after fixes [S1, unofficial]. Initial import without fixes crashes; with chunking but without overflow fixes, ~10 fps with artifacts [S1, unofficial].
*   **Strengths:** Handles massive datasets (4.4 GB) that exceed standard 32-bit limits. Efficient sparse storage.
*   **Limitations:** Requires specific engine-side code modifications (RHI return type, index types). Chunking is still required (at least 4 chunks), which is less ideal than loading the entire dataset in one go [S1, unofficial].
*   **When to use:** When rendering large-scale seismic or scientific data in Unreal Engine 5 using OpenVDB.
*   **When not to use:** For smaller datasets that fit within 32-bit integer limits without overflow, or if engine source code modification is not possible.

## Background
The method originates from the challenge of rendering large volume datasets in Unreal Engine 5, specifically the Kolumbo seismic dataset. The conversion from SEG-Y to VDB was performed using a Jupyter Notebook with `segyio` and OpenVDB Python bindings [S1, unofficial]. The import issues and fixes were documented in a survey on rendering large volume datasets in UE5 [S1, unofficial]. The OpenVDB format version used is 14 [S1, unofficial].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://arxiv.org/abs/2504.07485, fetched 2026-10-27 (unofficial)
