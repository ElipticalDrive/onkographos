# Voxel Edit Brush Operations — Place/Carve/Fill/Paint, Real-time Voxel Editing, VoxelEdit

Also called: FragmentSdfShaderPatch, Voxel Carving via Shader Patching

## Overview
This method implements real-time voxel carving in a browser-based 3D Gaussian Splatting (3DGS) renderer by injecting a discrete voxel occupancy mask into the fragment shader via a shader patching hook [S1, unofficial]. It solves the problem of crisp, axis-aligned removal of splats using a `discard` instruction, avoiding the performance penalty of per-fragment analytic SDF evaluations or per-splat geometric tests [S1, unofficial]. Unlike parent methods that may rely on sparse structures or offline processing, this approach uses a dense 3D texture sized to the voxel grid, enabling O(1) per-fragment lookup regardless of the number of carved cells [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3DGS scene rendered by a framework supporting `Material.onBeforeCompile` (e.g., Spark 2.0) [S1, unofficial].
- **Input:** Voxel grid parameters: origin, size, and counts ($N_x, N_y, N_z$) [S1, unofficial].
- **Input:** Brush operations (Carve/Uncarve) triggered by user interaction [S1, unofficial].
- **Output:** Modified fragment shader code that samples a `sampler3D` mask to determine pixel visibility [S1, unofficial].
- **Output:** Updated `Data3DTexture` backing the mask [S1, unofficial].

**Data Structures:**
1.  **Voxel Mask Texture (`uCarveMask`):** A `Data3DTexture` (or `Uint8Array`-backed 3D texture) with dimensions $N_x \times N_y \times N_z$. Each voxel is 1 byte (0 or 255).
2.  **Patch State:**
    -   `patch.count`: Integer tracking the number of active carved voxels.
    -   `union AABB`: `vec3 uCarveBoundsMin`, `vec3 uCarveBoundsMax` defining the bounding box of all carved voxels.
    -   `tex.version`: Texture version counter to trigger GPU updates.

**Uniforms Injected into Fragment Shader:**
-   `sampler3D uCarveMask`: The occupancy mask.
-   `vec3 uVoxelOrigin`: World-space origin of the voxel grid.
-   `float uVoxelSizeInv`: Inverse of the voxel size ($1/s$).
-   `vec3 uVoxelCountsInv`: Inverse of grid dimensions ($1/N_x, 1/N_y, 1/N_z$).
-   `vec3 uCarveBoundsMin`: Minimum corner of the union AABB.
-   `vec3 uCarveBoundsMax`: Maximum corner of the union AABB.
-   `int uCarveCount`: Number of active carved voxels (used for debugging or fallback logic).

**Uniforms Injected into Vertex Shader:**
-   `mat4 uClipToLocal`: Transformation matrix from clip space to local voxel space.
-   `out vec3 vWorldPos`: Output world position of the vertex.

**Numbered Steps:**

1.  **Initialization:**
    -   Create `uCarveMask` with dimensions $N_x \times N_y \times N_z$. Initialize all bytes to 0.
    -   Initialize `patch.count = 0`.
    -   Initialize `uCarveBoundsMin` to $(\infty, \infty, \infty)$ and `uCarveBoundsMax` to $(-\infty, -\infty, -\infty)$.
    -   Set `tex.version = 0`.

2.  **Carve Operation (`carve(key, i, j, k)`):**
    -   **Validation:** Check if $(i, j, k)$ is within bounds $[0, N_x-1] \times [0, N_y-1] \times [0, N_z-1]$. If not, return [S1, unofficial].
    -   **Check Duplicate:** If the texel at linear index $L$ is already 255, return [S1, unofficial].
    -   **Update Mask:** Set texel at linear index $L$ to 255.
        -   Linear index formula: $L = i + j \cdot N_x + k \cdot N_x \cdot N_y$ [S1, unofficial].
    -   **Update State:**
        -   `patch.count++`.
        -   Update `union AABB`:
            -   $v_{min} = \min(uCarveBoundsMin, \text{voxel center})$
            -   $v_{max} = \max(uCarveBoundsMax, \text{voxel center})$
            -   Voxel center in world space: $C = uVoxelOrigin + (i+0.5, j+0.5, k+0.5) \cdot s$, where $s$ is voxel size.
        -   `tex.version++`.
        -   Trigger texture update: `texture.needsUpdate = true` [S1, unofficial].

3.  **Uncarve Operation (`uncarve(key)`):**
    -   **Validation:** Check if key exists in the active set. If not, return [S1, unofficial].
    -   **Retrieve Indices:** Get $(i, j, k)$ from key.
    -   **Update Mask:** Set texel at linear index $L$ to 0.
    -   **Update State:**
        -   `patch.count--`.
        -   Update `union AABB`: Recompute min/max over all remaining active voxels (or maintain a more complex structure; sources imply recomputation or simple update) [S1, unofficial].
        -   `tex.version++`.
        -   Trigger texture update: `texture.needsUpdate = true` [S1, unofficial].

4.  **Shader Compilation (`compile(shader)`):**
    -   Inject vertex shader code:
        ```glsl
        uniform mat4 uClipToLocal;
        out vec3 vWorldPos;
        void main() {
            // Standard projection logic...
            vWorldPos = (uClipToLocal * vec4(position, 1.0)).xyz;
        }
        ```
    -   Inject fragment shader code:
        ```glsl
        uniform sampler3D uCarveMask;
        uniform vec3 uVoxelOrigin;
        uniform float uVoxelSizeInv;
        uniform vec3 uVoxelCountsInv;
        uniform vec3 uCarveBoundsMin;
        uniform vec3 uCarveBoundsMax;

        in vec3 vWorldPos;

        void main() {
            // Early-out: Check if world position is outside union AABB
            if (vWorldPos.x < uCarveBoundsMin.x || vWorldPos.x > uCarveBoundsMax.x ||
                vWorldPos.y < uCarveBoundsMin.y || vWorldPos.y > uCarveBoundsMax.y ||
                vWorldPos.z < uCarveBoundsMin.z || vWorldPos.z > uCarveBoundsMax.z) {
                // Continue rendering
            } else {
                // Map world position to voxel coordinates
                vec3 voxelCoord = (vWorldPos - uVoxelOrigin) * uVoxelSizeInv;
                // Clamp to [0, 1] for texture lookup
                voxelCoord = clamp(voxelCoord, 0.0, 1.0);
                
                // Sample mask
                float mask = texture(uCarveMask, voxelCoord).r;
                if (mask > 0.5) {
                    discard; // Carved voxel
                }
            }
            // Continue with standard 3DGS fragment processing
        }
        ```
    -   **Anchor Check:** Verify that the shader patching hook finds the expected anchor strings in the original shader source. If missing, throw an error [S1, unofficial].

**Parameters:**
-   **Voxel Resolution ($N$):** Default 64 ($64^3 = 262,144$ cells) [S1, unofficial].
-   **Voxel Size ($s$):** Determined by scene bounds and $N$.
-   **Brush Radius:** World-space spherical radius defining the set of voxels to modify per interaction [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Out-of-Bounds Indices:** Clamp or reject indices outside $[0, N-1]$ to prevent texture access errors [S1, unofficial].
-   **Missing Anchors:** If the upstream shader changes, the string injection may fail. A CI guard hashes the shader region to detect changes [S1, unofficial].
-   **2DGS Interference:** Carving while 2D Gaussian Splatting is enabled may cause incorrect masking due to different vertex output assignments [S1, unofficial].
-   **Performance:** Fragment cost is O(1) per fragment, independent of `uCarveCount` [S1, unofficial]. However, texture upload cost scales with the number of changed voxels.

## Operating characteristics
-   **Time Complexity:** O(1) per fragment for the carve test, regardless of the number of carved voxels [S1, unofficial]. Carve/Uncarve operations are O(1) per voxel, with AABB recomputation potentially O(K) where K is the number of active voxels (though sources suggest simple updates) [S1, unofficial].
-   **Memory Complexity:** O($N_x \cdot N_y \cdot N_z$) bytes for the mask texture. For $N=64$, this is 256 KB [S1, unofficial].
-   **Measured Performance:** ~4 splat counts per frame instead of ~pixels per frame for carving logic, leveraging GPU parallelism [S1, unofficial].
-   **Strengths:** Crisp, axis-aligned carving; real-time performance; simple implementation via shader patching.
-   **Limitations:** Memory usage scales cubically with resolution; fragile to upstream shader changes; limited to binary (carved/not carved) states; no per-session carve count cap other than grid size [S1, unofficial].
-   **When to Use:** Real-time editing of 3DGS scenes in the browser where crisp, voxel-based removal is required.
-   **When Not to Use:** When smooth, sub-voxel carving is needed; when memory is extremely constrained; when the renderer does not support shader patching hooks.

## Background
Originating from the `splatcarve` project by stevekwon211 (2024–2026), this method leverages the `Material.onBeforeCompile` hook introduced in Spark 2.0 [S1, unofficial]. It is the first public open-source demonstration of applying a voxel-cell-resolution `discard` mask to a live 3DGS rasterizer in a web browser [S1, unofficial]. Prior art includes analytic SDF-based selection (e.g., Santos & Soares, SVR 2025), but those target modulation effects rather than crisp removal [S1, unofficial]. The class names `FragmentSdfShaderPatch` are vestigial, as the method uses a discrete binary mask, not a continuous SDF [S1, unofficial].

## Sources
[S1] stevekwon211/splatcarve, https://github.com/stevekwon211/splatcarve, fetched 2026-10-27 (unofficial)
