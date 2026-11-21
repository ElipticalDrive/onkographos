# Occlusion-Based Face Generation


## Overview
Occlusion-Based Face Generation is a surface extraction method that iterates over voxels in a chunk to generate face primitives only for sides that are not occluded by adjacent voxels [S1, unofficial]. It serves as the core geometry generation step within GPU Face Extraction, distinguishing itself from sibling methods like Validator Reference Face Reconstruction by directly producing visible quads for the rendering pipeline while handling transparency and water occlusion rules [3760-gpu-face-extraction.md][3794-validator-reference-face-reconstruction.md].

## How it works
**Inputs and Outputs:**
- **Input:** A voxel chunk containing voxel data, occlusion masks, and soft light values.
- **Output:** A vector of face primitives (`faces_vertices`), where each face represents a visible side of a voxel.

**Data Structures:**
- `Chunk::BUFFER_SIZE`: The total number of voxels in the chunk.
- `voxel::SIDE_COUNT`: The number of sides per voxel (typically 6).
- `voxel::SIDES`: An iterable collection of the 6 face normals/directions.
- `voxel::Face`: A struct containing:
  - `vertices`: An array of 4 vertex coordinates (defined by the voxel position and side).
  - `side`: The face normal/direction.
  - `kind`: The material type of the voxel.
  - `light`: The soft light value for the specific side.

**Parameters:**
- `FACES_ESTIMATION`: A capacity hint for the output vector, calculated as `(Chunk::BUFFER_SIZE * voxel::SIDE_COUNT) / 2` [S1, unofficial].

**Steps:**
1. Initialize a vector `faces_vertices` with capacity `FACES_ESTIMATION`.
2. Iterate over all voxels in the chunk.
3. For each voxel:
   a. Retrieve the voxel `kind`. If `kind` is empty/none, skip to the next voxel.
   b. Retrieve the voxel's `occlusion` mask and `soft_light` data.
   c. Iterate over all 6 `side`s in `voxel::SIDES`.
   d. Check if the side is occluded using `occlusion.is_occluded(side)`. If occluded, skip to the next side.
   e. Construct a `voxel::Face` object:
      - Set `vertices` based on the voxel coordinates and the current `side`.
      - Set `side` to the current side.
      - Set `kind` to the voxel's material type.
      - Set `light` to the soft light value for the current side (`voxel_soft_light.get(side)`).
   f. Push the face to `faces_vertices`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
FACES_ESTIMATION = (Chunk.BUFFER_SIZE * voxel.SIDE_COUNT) / 2

# Initialize output buffer
faces_vertices = []
faces_vertices.reserve(FACES_ESTIMATION)

# Iterate over all voxels in the chunk
for voxel in chunk.voxels():
    # Get material kind; skip empty voxels
    kind = kind_storage.get(voxel)
    if kind is None:
        continue

    # Get occlusion and soft light data
    occlusion = occlusion_storage.get(voxel)
    voxel_soft_light = soft_light_storage.get(voxel)

    # Iterate over all 6 sides
    for side in voxel.SIDES:
        # Skip if the side is occluded
        if occlusion.is_occluded(side):
            continue

        # Construct face vertices based on voxel position and side
        # Note: Exact vertex calculation depends on voxel coordinate system and side normal
        v1, v2, v3, v4 = voxel.compute_face_vertices(side)

        # Create face primitive
        face = {
            'vertices': [v1, v2, v3, v4],
            'side': side,
            'kind': kind,
            'light': voxel_soft_light.get(side)
        }

        # Add to output buffer
        faces_vertices.append(face)
```

**Edge Cases and Pitfalls:**
- **Empty Voxels:** The algorithm explicitly skips voxels with no material kind to avoid generating unnecessary geometry [S1, unofficial].
- **Occlusion Logic:** The correctness of visible faces depends entirely on the accuracy of the `occlusion` mask. Incorrect occlusion data will lead to either overdraw (if not occluded) or missing geometry (if incorrectly occluded) [S1, unofficial].
- **Capacity Estimation:** `FACES_ESTIMATION` is a heuristic. If the actual number of visible faces exceeds this, the vector will reallocate. If it is too large, memory is wasted. The formula assumes an average of half the sides being visible [S1, unofficial].

## Subtopics

## Operating characteristics
**Time Complexity:** $O(V \times S)$, where $V$ is the number of voxels in the chunk and $S$ is the number of sides per voxel (6). The algorithm performs a constant amount of work per voxel side.
**Memory Complexity:** $O(F)$, where $F$ is the number of visible faces. The output buffer size is bounded by $V \times S$.
**Measured Performance:** Not given in the sources.
**Strengths:** Simple iteration-based approach; easy to implement on GPU; handles transparency via soft light values.
**Limitations:** Relies on pre-computed occlusion masks; may generate more geometry than necessary if occlusion is not tightly packed; does not merge coplanar faces.
**When to Use:** When a straightforward, per-voxel face generation is required within a chunk-based voxel renderer.
**When Not to Use:** When memory efficiency is critical and face merging is needed, or when occlusion data is not available.

## Background
Origin: This method is part of the `projekto` project by afonsolage, an unofficial implementation of voxel rendering techniques [S1, unofficial]. It is designed to work within a chunk-based architecture, likely for real-time voxel rendering applications.

## Sources
[S1] afonsolage/projekto, https://github.com/afonsolage/projekto, fetched 2026-10-27 (unofficial)
