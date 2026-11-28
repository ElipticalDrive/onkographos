# Voxel Face Visibility Check

Also called: Back-Face Culling, Camera-relative Block Culling, Cube Face Culling, renderBlock

## Overview
Voxel Face Visibility Check is a geometric optimization that determines which of the six faces of a voxel block are visible to the camera, allowing the renderer to skip generating or uploading geometry for faces pointing away from the viewer [S1, S3, S4, unofficial]. It operates at the individual face level within a block, providing finer-grained culling than methods that discard entire blocks [S1, S3, unofficial].

## How it works
The method computes a visibility mask or list for the six faces of a cube based on the relative position of the camera to the block [S1, S3, unofficial].

**Inputs:**
- `eyePt`: Camera position in world space (float3) [S1, unofficial].
- `worldI`: Inverse of the block's world transformation matrix (4x4) [S1, unofficial].
- `cube_size`: Side length of the voxel block (float) [S3, unofficial].

**Outputs:**
- `visible_mask`: A 6-bit integer where bit $i$ is set if face $i$ is visible [S1, unofficial].
- OR `face_list`: A list of indices of visible faces [S1, unofficial].

**Data Structures:**
- Face indices are mapped as: 0=X−, 1=X+, 2=Y−, 3=Y+, 4=Z−, 5=Z+ [S1, unofficial].
- The visibility mask is a 6-bit integer [S1, unofficial].

**Steps:**
1. **Transform Eye Position:** Convert the camera position from world space to the block's local space: $\mathbf{e}_{local} = \text{Transform}(\mathbf{e}_{world}, \mathbf{M}_{world}^{-1})$ [S1, unofficial].
2. **Define Face Properties:** Define the outward normal $\mathbf{n}_i$ and center offset $\mathbf{o}_i = \mathbf{n}_i \times (cube\_size / 2)$ for each face [S3, unofficial].
3. **Compute Visibility:** For each face $i$, check if the camera lies in the half-space outside the face plane: $v_i = (\mathbf{e}_{local} - \mathbf{o}_i) \cdot \mathbf{n}_i > 0$ [S3, unofficial]. This is geometrically equivalent to checking if the camera is on the "outside" of the face plane [S3, unofficial].
4. **Collect Results:** Set bit $i$ in `visible_mask` if $v_i$ is true, or append $i$ to `face_list` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_face_visibility(eye_world, world_inv, cube_size):
    half_size = cube_size / 2.0
    # Transform eye to block-local space (standard matrix-vector multiplication)
    eye_local = transform_vector(eye_world, world_inv)
    
    visible_mask = 0
    for face in range(6):
        # Map face index to axis and direction
        axis_idx = face >> 1  # 0=X, 1=Y, 2=Z
        direction = face & 1  # 0=negative axis, 1=positive axis
        
        # Get the relevant component of the local eye position
        view_comp = eye_local[axis_idx]
        
        # Check if the camera is outside the block's extent on this axis
        # Face is visible if the camera is on the outside of the face plane
        if direction:
            visible = view_comp > -half_size
        else:
            visible = view_comp < half_size
            
        if visible:
            visible_mask |= (1 << face)
    return visible_mask
```

**Parameters:**
- `_CPU_CUBE_FACE_CULL_`: Compile-time macro controlling culling mode. 0 = disabled, 1 = mask, 2 = list [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Coordinate System Mismatch:** The definition of face normals and axis mapping must match the rendering engine's coordinate system [S1, S3, unofficial].
- **Camera Inside Block:** If the camera is inside the block, the dot product test $\mathbf{v} \cdot \mathbf{n}_i > 0$ will correctly identify faces where the camera is "outside" the plane. If the camera is strictly inside, $\mathbf{v}$ points inwards for all faces, so $\mathbf{v} \cdot \mathbf{n}_i < 0$ and no faces are marked visible [G]. This is a known issue in voxel rendering; usually, the camera is assumed to be outside the voxel grid or the block is treated as solid [G].
- **Incorrect Reference Point:** Using the world origin $(0,0,0)$ as the reference point for the dot product instead of the camera position is a common bug that incorrectly marks faces visible only if they point away from the world origin [S3, unofficial].
- **Precision:** Floating-point precision errors near block boundaries can cause flickering. Using a small epsilon in the comparison is recommended [G].

## Subtopics

### Chunk Face Visibility Culling
Determines visible directions for a chunk of voxels by comparing the player's position against the chunk's bounding box [S5, unofficial]. It initializes six axis-aligned unit vectors representing outward directions [S5, unofficial]. If the player is inside the chunk bounds, all directions remain visible [S5, unofficial]. Otherwise, it removes directions pointing away from the player by comparing player coordinates to chunk bounds (e.g., removes $(1,0,0)$ if `player.x ≤ chunk.position.x`) and applies symmetric checks for the Y and Z axes [S5, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per block, as it involves a fixed number of dot products (6) and vector transformations [S1, S3, unofficial].
- **Memory Complexity:** $O(1)$, storing only a few temporary vectors and a bitmask [S1, unofficial].
- **Strengths:** Simple to implement, effective for large voxel worlds, reduces GPU load by skipping back-facing geometry [S2, S4, unofficial].
- **Limitations:** Does not handle occlusion from other blocks (only back-face culling) [S4, unofficial]. Requires accurate camera position and block transformation [S1, unofficial].

## Background
Back-face culling is a fundamental technique in computer graphics used to avoid rendering geometry facing away from the user [S4, unofficial]. In the context of voxel rendering, it was adopted to optimize the high triangle counts associated with naive voxel meshing [S2, unofficial]. The specific implementation details in [S1, unofficial] are from a DirectX 12 fluid simulation project, while [S3, unofficial] provides a Python-based educational implementation and [S5, unofficial] extends the concept to chunk-level culling.

## Sources
[S1] StarsX/FluidX12, https://github.com/StarsX/FluidX12, fetched 2026-10-26 (unofficial)
[S2] Optimising Voxel World Rendering, https://gamedev.stackexchange.com/questions/117398/optimising-voxel-world-rendering, fetched 2026-10-27 (unofficial)
[S3] anb2473/Python-Voxel-Rendering-Engine, https://github.com/anb2473/Python-Voxel-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S4] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
[S5] edgorman/Voxel-Engine, https://github.com/edgorman/Voxel-Engine, fetched 2026-10-27 (unofficial)
