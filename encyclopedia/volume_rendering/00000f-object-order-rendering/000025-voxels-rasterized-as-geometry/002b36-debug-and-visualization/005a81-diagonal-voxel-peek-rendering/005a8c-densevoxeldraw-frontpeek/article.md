# DenseVoxelDraw FrontPeek

Also called: DenseVoxelDraw Front, FrontPeek

## Overview
DenseVoxelDraw FrontPeek is a diagonal rendering method that iterates through every pixel column of the front view (X-Z plane) and casts a ray along the Y-axis to find the first visible voxel, stopping at the first non-zero entry [S1, unofficial]. It is similar to the standard Front method but adds scaling and a 'peek' capability, drawing top and front faces if the voxel is on the top boundary or has empty space above it [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel model represented as a 3D array `model` of bytes, with dimensions `SizeX`, `SizeY`, and `SizeZ`.
*   **Inputs:** Scaling factors `scaleX` (horizontal) and `scaleY` (vertical), and an enum `VisibleFace` indicating which face is rendered (default `VisibleFace.Front`) [S1, unofficial].
*   **Output:** Screen-space rectangles drawn via a renderer interface (e.g., `renderer.Rect`).

**Data Structures:**
*   `model`: A 3D array indexed as `model[x, y, z]`. The source notes imply a specific iteration order and coordinate system where `z` corresponds to depth in the view, `x` to horizontal screen position, and `y` to vertical depth into the scene (or vice versa depending on the specific "front view" definition, but the loop structure clarifies the access pattern). Based on the loop structure `z` (outer), `x` (middle), `y` (inner), and the screen coordinate calculation `(x, model.SizeZ - 1 - z)`, `x` maps to screen X, and `z` maps to screen Y (inverted). The ray is cast along `y`.

**Parameters:**
*   `scaleX`: Horizontal scaling factor. Default: 6 [S1, unofficial].
*   `scaleY`: Vertical scaling factor. Default: 6 [S1, unofficial].
*   `VisibleFace`: Enum for face selection. Default: `VisibleFace.Front` [S1, unofficial].

**Steps:**
1.  Initialize outer loop `z` from 0 to `model.SizeZ - 1`.
2.  Initialize middle loop `x` from 0 to `model.SizeX - 1`.
3.  Initialize inner loop `y` from 0 to `model.SizeY - 1`.
4.  Check if `model[x, y, z]` is a non-zero byte.
5.  If a non-zero voxel is found:
    *   Determine if the voxel is on the top boundary or has empty space above it. The condition is `z >= model.SizeZ - 1` OR `model[x, y, z+1] == 0`. Note: The source notes use `z` as the depth index for the ray cast, but the peek condition checks `z+1`. This implies the "top" in the peek context refers to the visual top (highest `z` value in the loop, which is the smallest screen Y). However, the loop iterates `z` from 0 to `SizeZ`. The screen Y is `model.SizeZ - 1 - z`. So `z=0` is the bottom of the screen, `z=SizeZ-1` is the top. The condition `z >= model.SizeZ - 1` checks if we are at the last `z` index (top of the volume). The condition `model[x, y, z+1] == 0` checks the voxel "above" in the `z` dimension. If `z` is the last index, `z+1` is out of bounds. The source notes say `model[x, y, z+1] == 0`. This suggests the ray is cast along `y`, but the "peek" is for the `z` direction? No, the overview says "casts a ray along the Y-axis". The peek draws "top and front faces". In a front view (looking down Y?), the top face would be the one with normal +Y or -Y? Or is the view along Z?
    *   Let's re-read carefully: "front view (X-Z plane)". This usually means looking along the Y-axis. The screen coordinates are `(x, model.SizeZ - 1 - z)`. This maps `x` to screen X and `z` to screen Y. The ray is cast along `y`.
    *   The "top" of the voxel in the screen corresponds to the smallest `z` value (since `ScreenY = SizeZ - 1 - z`). The "front" face is the one facing the camera (along Y).
    *   The condition `z >= model.SizeZ - 1` checks if `z` is the maximum index. This is the *bottom* of the screen.
    *   The condition `model[x, y, z+1] == 0` checks the voxel at `z+1`. If `z` is small, `z+1` is larger, which is lower on the screen.
    *   There is a contradiction in the source notes between "top boundary" and the index check. However, the code logic is explicit:
        *   If `z >= model.SizeZ - 1` OR `model[x, y, z+1] == 0`: Draw two rects.
        *   Else: Draw one rect.
    *   Let's follow the source logic exactly.
    *   Calculate screen position: `screenX = x * scaleX`, `screenY = (model.SizeZ - 1 - z) * scaleY`.
    *   If `z >= model.SizeZ - 1` OR `model[x, y, z+1] == 0`:
        *   Draw top face rect at `(screenX, screenY)` with size `scaleX` by 1.
        *   Draw front face rect at `(screenX, screenY + 1)` with size `scaleX` by `scaleY - 1`.
    *   Else:
        *   Draw single front face rect at `(screenX, screenY)` with size `scaleX` by `scaleY`.
    *   Break the inner loop (stop casting ray along Y).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def DenseVoxelDrawFrontPeek(model, scaleX=6, scaleY=6, VisibleFace=VisibleFace.Front):
    """
    Renders the voxel model using the FrontPeek method.
    
    Args:
        model: 3D array of bytes, indexed as model[x, y, z].
        scaleX: Horizontal scaling factor.
        scaleY: Vertical scaling factor.
        VisibleFace: Enum indicating which face is rendered.
    """
    size_x = model.SizeX
    size_y = model.SizeY
    size_z = model.SizeZ
    
    for z in range(size_z):
        for x in range(size_x):
            for y in range(size_y):
                # Check if voxel is non-zero
                if model[x, y, z] != 0:
                    # Calculate screen coordinates
                    screen_x = x * scaleX
                    screen_y = (size_z - 1 - z) * scaleY
                    
                    # Check peek condition
                    # Note: z+1 access requires bounds check or assumption that 
                    # the condition handles it. The source notes imply direct access.
                    # If z is the last index, z+1 is out of bounds.
                    # The source says: "z >= model.SizeZ - 1 OR model[x, y, z+1] == 0"
                    # This implies if z is the last index, the first part is true.
                    # If z is not the last index, we check the next z.
                    
                    is_top_boundary = (z >= size_z - 1)
                    has_empty_above = False
                    if not is_top_boundary:
                        # Accessing z+1. In the source, this is valid.
                        # We assume model is padded or the check is safe.
                        # [G] Guard against out-of-bounds if model doesn't support z+1
                        if z + 1 < size_z:
                            has_empty_above = (model[x, y, z + 1] == 0)
                        else:
                            has_empty_above = True # Treat out of bounds as empty?
                    
                    if is_top_boundary or has_empty_above:
                        # Draw top face
                        # Size: scaleX by 1
                        renderer.Rect(screen_x, screen_y, scaleX, 1)
                        # Draw front face
                        # Size: scaleX by (scaleY - 1)
                        # Positioned below the top face
                        renderer.Rect(screen_x, screen_y + 1, scaleX, scaleY - 1)
                    else:
                        # Draw single front face
                        # Size: scaleX by scaleY
                        renderer.Rect(screen_x, screen_y, scaleX, scaleY)
                    
                    # Stop ray casting along Y
                    break
```

**Edge Cases and Pitfalls:**
*   **Out of Bounds:** The condition `model[x, y, z+1]` accesses `z+1`. If `z` is `SizeZ - 1`, this is out of bounds. The source notes handle this with the `OR` condition `z >= model.SizeZ - 1`, which short-circuits the evaluation if the first part is true. In implementation, ensure the check order prevents out-of-bounds access.
*   **Precision:** Integer arithmetic is used for coordinates. Scaling factors are integers.
*   **Degenerate Input:** If `scaleY` is 1, the front face rect height is 0. If `scaleY` is 0, the entire rect is 0. Default is 6.

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(N)$ where $N$ is the number of voxels, but typically much faster as it stops at the first non-zero voxel per column. In the worst case (empty volume or all zeros), it checks all voxels. In the best case (solid front), it checks 1 voxel per column.
*   **Memory:** $O(1)$ auxiliary memory, excluding the voxel model storage.

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Scales linearly with the number of pixel columns ($SizeX \times SizeZ$).

**Strengths and Limitations:**
*   **Strengths:** Simple implementation, fast for sparse or front-heavy volumes, provides a "peek" into the volume structure by showing top faces.
*   **Limitations:** Only renders the front-most voxel per column, no depth cueing or transparency, limited to diagonal/peek view.

**When to Use:**
*   Debugging, visualization, and simple voxel previews where performance is critical and full ray-marching is not needed.

## Background
**Origin:**
*   The method is part of the Voxel2Pixel graphics library by Ben McLean [S1, unofficial].

**History and Development:**
*   The library is a graphics library for rendering voxel data [S1, unofficial].

**Where it is Used:**
*   Debugging and visualization of voxel data [S1, unofficial].

## Sources
[S1] Voxel2Pixel, https://github.com/BenMcLean/Voxel2Pixel, fetched 2026-10-29 (unofficial)
