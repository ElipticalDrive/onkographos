# Block Inspection (Streamed)
Also called: streamed block picking, vox.block.pick.zarr.html
## Overview
Block Inspection (Streamed) enables interactive, high-resolution inspection of a specific region within a streamed, multi-resolution volume dataset. Unlike CPU-resident block picking which extracts voxel data into memory, this method opens a secondary streamed volume instance cropped to the picked brick's bounding box, leveraging the underlying octree to serve the finest available levels for that sub-region. It distinguishes itself from [Streaming MIP Construction](1815-streaming-mip-construction.md) by focusing on interactive point-picking rather than pre-computed aggregation, and from [Conical MImP](7656-conical-mimp.md) by using a standard cropped volume view rather than a conical projection.

## How it works
**Inputs and Outputs:**
- Input: A primary streamed volume object (backed by a Zarr or similar store) and a 2D screen coordinate $(x, y)$ representing the user's pick.
- Output: A secondary volume view (or texture) displaying the high-resolution content of the picked brick's local coordinate space, allowing the user to inspect details that are not visible in the main viewport due to LOD selection.

**Data Structures:**
- `StreamedVolume`: The primary volume object managing chunk loading and LOD selection.
- `Octree`: The hierarchical structure mapping grid coordinates to LOD levels.
- `CommonGridBox`: The axis-aligned bounding box (AABB) in the parent volume's common grid coordinates that defines the extent of the picked brick.

**Steps:**
1. **Primary Pick:** Perform a pick operation on the primary streamed volume at screen coordinates $(x, y)$. Identify the specific brick (chunk) currently rendered at that location. Determine the brick's bounding box in the parent volume's common grid coordinates. Let this box be $B_{brick}$.
2. **Secondary Volume Creation:** Instantiate a second `StreamedVolume` object pointing to the same underlying data store as the primary volume.
3. **Cropping:** Configure the secondary volume's view to be cropped to $B_{brick}$. This is achieved by setting the secondary volume's spatial bounds or crop region to match $B_{brick}$.
4. **LOD Pulling:** The secondary volume's octree is initialized or queried. Because the secondary volume is smaller (cropped to one brick), its octree can pull data from finer LOD levels (closer to level 0) for the cropped region without exceeding memory or bandwidth budgets that would affect the full-volume render. The system fetches the finest available chunks for $B_{brick}$.
5. **Sampling Fix:** Ensure the pick sampling mechanism for the secondary view uses `vol.pickSampler`. This sampler operates in millimeter space and is backed by the coarse floor of the secondary volume's LOD hierarchy. This prevents the pick from landing on the brick's box face (a common artifact when sampling parent CPU images or incorrect LODs) and ensures accurate anatomical coordinate reporting.
6. **Rendering:** Render the secondary volume view. The visual detail comes from the crop source's finest levels, not the brick's own texture (which is already at its native level).

**Formulas:**
- Let $P_{screen}$ be the pick position.
- Let $M_{RAS}$ be the model-view-projection matrix for the primary volume.
- The brick $B$ is identified such that $P_{screen}$ projects into the screen-space bounds of $B$.
- The crop region $C$ for the secondary volume is defined by the grid indices $[x_{min}, x_{max}] \times [y_{min}, y_{max}] \times [z_{min}, z_{max}]$ corresponding to $B_{brick}$.

**Parameters:**
- Crop Region: Defined by the picked brick's common-grid box.
- LOD Level: Determined by the secondary volume's octree based on the cropped size and available budget. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Null Extraction:** `extractSubVolume` returns `null` for streamed volumes because the data is not resident in CPU memory. Do not attempt to copy voxel data to a CPU buffer.
- **Pick Accuracy:** If the pick sampler falls back to sampling the parent's CPU image (which may be null or coarse for streamed data), the pick will land on the brick's box face rather than the actual data. The fix is to use `vol.pickSampler` which handles mm-space sampling correctly for the secondary volume.
- **Redundant Fetching:** Re-fetching the brick at its own level in the secondary volume buys no detail. The detail gain comes from the secondary volume's ability to access finer levels for the cropped region.

## Subtopics

### Block Inspection (CPU)
CPU-side block picking for resident NIfTI volumes. It resolves a pointer onto a visible brick by slab-testing each brick's exploded millimeter AABB and inverting the exploded `matRAS` matrix, as GPU depth picking cannot see per-brick translations. It uses `extractSubVolume` or `extractChunkBlock` to copy a voxel box out as a complete NIfTI volume, walking the affine's origin to the box so oblique parents remain oblique. It uses the brick's DATA region (not halo) to tile the parent without overlap. [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dependent on the octree traversal depth for the cropped region. Since the region is small (one brick), LOD selection is fast.
- **Memory Complexity:** Proportional to the size of the cropped brick's finest available LODs. Significantly less than loading the full volume at high resolution.
- **Strengths:** Allows high-resolution inspection of specific regions in large, streamed datasets without loading the entire volume into memory.
- **Limitations:** Requires opening a second volume instance, which may have overhead. Dependent on the underlying store's ability to serve fine-grained chunks efficiently.
- **When to use:** Interactive exploration of specific features in large multi-resolution volumes.
- **When not to use:** When CPU-side processing of the voxel data is required (use CPU variant). When the dataset is small enough to fit in memory.

## Background
This method was introduced as part of streamed multi-LOD rendering fixes in the niivue library. It addresses the limitation that streamed volumes do not have a resident CPU image, making traditional `extractSubVolume` methods ineffective. The approach leverages the streaming infrastructure itself to provide high-resolution crops. [S1, unofficial].

## Sources
[S1] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
