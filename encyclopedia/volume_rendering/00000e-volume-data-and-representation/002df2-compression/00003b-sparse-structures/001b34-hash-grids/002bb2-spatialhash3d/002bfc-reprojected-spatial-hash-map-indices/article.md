# Reprojected Spatial Hash Map Indices

Also called: Temporal Reprojection of Hash Indices

## Overview
This method optimizes spatial hash lookups in temporal volume rendering by reusing hash indices from the previous frame, thereby reducing the frequency of expensive atomic memory operations on the hash grid [S1, unofficial]. It differs from standard spatial hash lookups by decoupling the index computation from the final shading pass, using a texture-based history buffer to validate and propagate indices across frames [S1, unofficial].

## How it works
The method splits the Ray Traced Ambient Occlusion (RTAO) pass into two distinct stages: an index reprojection stage and a fullscreen shading stage. This separation allows the renderer to minimize atomic traffic to the global hash map, which is a primary bottleneck in sparse voxel rendering [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Previous frame’s hash index texture (containing indices and checksums).
    *   Current frame’s camera and scene state (for reprojection).
    *   Hash grid data structure (for fallback lookups).
    *   Frame counter (`uniforms.frame_index`).
    *   Touch mask parameter (`VX_AO_HASH_TOUCH_PERIOD`).
*   **Outputs**:
    *   Current frame’s hash index texture (written to GPU memory).
    *   Updated hash grid frame counters (via atomic operations).

### Data Structures
The method relies on a texture storing the hash indices from the previous frame. While the exact layout of the "history and checksum" is not fully specified in the source, it implies a structure containing at least:
1.  **Index**: The spatial hash index from the previous frame.
2.  **Checksum**: A value used to validate that the spatial location and scene state have not changed significantly [S1, unofficial].

### Steps
1.  **Reproject**: For each pixel/screen position, look up the hash index and checksum from the previous frame’s history texture.
    *   If the history is valid and the checksum matches the current frame’s expected state, use the reprojected index.
    *   Otherwise, compute the new hash index using the standard spatial hash function [S1, unofficial].
2.  **Write**: Write the current frame’s computed or reprojected hash index to a new index texture. This texture is then used in the subsequent shading pass [S1, unofficial].
3.  **Render**: Perform a fullscreen RTAO pass using the g-buffer data and the newly written hashmap index texture. This decouples the heavy shading logic from the index lookup logic [S1, unofficial].
4.  **Update Frame Counter**: Atomically update the frame counter for the hash entry to track its freshness. To reduce atomic traffic, this update is performed only periodically based on a touch mask [S1, unofficial].

### Pseudocode
The following pseudocode illustrates the core logic for index reprojection and frame counter updates, adapted from the source [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
VX_AO_HASH_TOUCH_PERIOD = 8  # Example value; source uses VX_AO_HASH_TOUCH_PERIOD
touch_mask = VX_AO_HASH_TOUCH_PERIOD - 1

def reproject_hash_index(previous_index_texture, current_scene_state, hash_grid, frame_index):
    """
    Determines the hash index for the current frame.
    Returns: (index, is_reprojected)
    """
    # 1. Look up previous index and checksum
    prev_index, prev_checksum = previous_index_texture.read(current_screen_position)
    
    # 2. Validate history
    # Note: Exact checksum validation logic is not detailed in source [G]
    is_valid = check_checksum(prev_checksum, current_scene_state)
    
    if is_valid:
        # Use reprojected index
        return prev_index, True
    else:
        # Compute new hash index (fallback to standard spatial hash)
        # Note: Exact hash function details are in SpatialHash3D [G]
        new_index = compute_spatial_hash(current_scene_state.position, hash_grid)
        return new_index, False

def update_frame_counter(hash_frames, hash_index, frame_index, touch_mask):
    """
    Atomically updates the frame counter for a hash entry.
    Uses a touch mask to scramble the update phase and reduce contention.
    """
    # Scramble frame update phase using hash to spread updates
    if (frame_index & touch_mask) == (hash_index & touch_mask):
        # Perform atomic exchange
        # InterlockedExchange returns the old value, but we only care about the side effect
        old_frame = atomic_interlocked_exchange(hash_frames[hash_index], frame_index)
        return old_frame
    else:
        # Skip update to reduce atomic traffic
        return None

# Main Pass Logic
def rtao_index_pass(previous_index_texture, hash_grid, frame_index):
    # Step 1: Reproject
    current_index, is_reprojected = reproject_hash_index(
        previous_index_texture, current_scene_state, hash_grid, frame_index
    )
    
    # Step 2: Write index to texture
    current_index_texture.write(current_screen_position, current_index)
    
    # Step 3: Update frame counter (only if necessary or based on mask)
    # Note: The source implies the counter update is tied to the hash entry, 
    # not necessarily every pixel. The pseudocode below reflects the conditional update logic.
    update_frame_counter(hash_grid.frame_counters, current_index, frame_index, touch_mask)
    
    return current_index_texture
```

### Parameters
*   **`VX_AO_HASH_TOUCH_PERIOD`**: Determines the frequency of frame counter updates. The touch mask is derived as `VX_AO_HASH_TOUCH_PERIOD - 1`. The source does not specify the exact value used, but implies it is a power of two to facilitate bitwise masking [S1, unofficial].

### Edge Cases and Pitfalls
*   **Atomic Traffic**: Atomic reads and writes to the hash map cause high MMU activity and latency. Without reprojection, the index pass can take over 6 milliseconds on hardware like the MacBook Air (M2) [S1, unofficial].
*   **Invalid History**: If the history is invalid (e.g., due to camera movement or scene changes), the fallback to computing a new hash index incurs the full cost of atomic operations [S1, unofficial].
*   **Checksum Mismatch**: A mismatching checksum triggers a recomputation, which is more expensive than a reprojected lookup [S1, unofficial].

## Subtopics

### SpatialHash3D
The underlying spatial index structure. Reprojected indices rely on the O(1) lookup provided by SpatialHash3D for fallback computations [Parent].

### RTAO
The shading pass that consumes the reprojected indices. The decoupling allows for optimized shading without index lookup overhead [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The reprojection step is O(1) per pixel for valid history, avoiding the hash computation and atomic writes. Fallback cases remain O(1) but with higher constant factors due to atomic operations [S1, unofficial].
*   **Memory Complexity**: The method requires storing the previous frame’s index texture, adding memory overhead proportional to the screen resolution [S1, unofficial].
*   **Measured Performance**:
    *   **monu2.vox**: Average GPU walltime reduced from 6.46 ms to 5.88 ms (-9.01%). FPS remained stable at ~144 [S1, unofficial].
    *   **sponza-1024.vox**: Average GPU walltime reduced from 26.17 ms to 24.95 ms (-4.68%). FPS increased from 64.81 to 67.72 (+4.49%) [S1, unofficial].
*   **Memory Usage Reduction**:
    *   Device Atomic Bytes Read dropped from 309.19 MiB to 4 KiB.
    *   Device Atomic Bytes Written dropped from 617.13 MiB to 19.16 MiB [S1, unofficial].
*   **Strengths**: Significantly reduces atomic memory traffic and MMU activity, leading to lower latency in the index pass [S1, unofficial].
*   **Limitations**: The performance gain is modest in heavy scenes (e.g., Sponza) and depends on the validity of the history buffer. It adds complexity by requiring a two-pass approach [S1, unofficial].
*   **When to Use**: When atomic operations on the hash grid are a bottleneck and the scene/camera motion is smooth enough to allow valid history reprojection [S1, unofficial].
*   **When Not to Use**: In scenes with rapid, discontinuous changes where history validity is low, or when memory bandwidth for the index texture is constrained [S1, unofficial].

## Background
This optimization was introduced in the development of the `vxray` voxel raytracer, specifically in commit `c41b366` dated 2026-09-01 [S1, unofficial]. It was developed to address high MMU activity and latency observed during the spatial hash lookup pass on Apple Silicon hardware (MacBook Air M2) [S1, unofficial]. The technique is part of a broader effort to optimize sparse voxel rendering by reducing the cost of hash map interactions [S1, unofficial].

## Sources
[S1] Voxel Raytracer Devlog // My thought repository, https://nelari.us/post/voxel-ray-tracing/, fetched 2026-10-24 (unofficial)
