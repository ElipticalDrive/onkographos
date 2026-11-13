# Per-fragment Voxel-Cell Mask Injection

Also called: Fragment SDF Shader Patch, H2′

## Overview
Per-fragment Voxel-Cell Mask Injection is a technique for carving 3D Gaussian Splatting scenes by injecting a per-fragment Signed Distance Field (SDF) evaluation into a compiled fragment shader. It solves the problem of creating clean, cube-shaped holes in a splat cloud without forking the underlying renderer or deleting whole splats. It differs from its sibling [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) and the baseline [H2](2061-tiled-2d-gaussian-splatting-rasterization.md) approach by operating at the fragment level using a 3D texture lookup, producing sharp cube edges instead of the rough masking artifacts caused by per-splat deletion.

## How it works
The method injects GLSL code into a renderer's fragment shader (specifically via `THREE.Material.onBeforeCompile`) to evaluate a voxel grid mask at render time.

**Inputs and Outputs:**
- **Input:** A 3D Gaussian Splatting scene, a voxel grid definition (origin, size, counts), and a set of active carve cells.
- **Output:** A rendered image where fragments falling within active carve cells are discarded, creating a hole.

**Data Structures:**
The implementation uses a `FragmentSdfShaderPatch` class managing the following uniforms and textures:
1.  `uCarveMask`: A `Data3DTexture` (sampler3D in GLSL). Dimensions match the voxel grid counts $(N_x, N_y, N_z)$. Data is a `Uint8Array` initialized to 0. A value $> 0.5$ indicates a carved cell.
2.  `uVoxelOrigin`: A `vec3` representing the grid's origin in world space.
3.  `uVoxelSizeInv`: A `float` storing $1 / \text{voxelSize}$.
4.  `uVoxelCountsInv`: A `vec3` storing $1 / (N_x, N_y, N_z)$.
5.  `uCarveBoundsMin` / `uCarveBoundsMax`: `vec3` uniforms defining the union AABB of all active carved cells for early-out optimization.
6.  `uCarveCount`: An integer uniform tracking the number of active carves.

**Numbered Steps:**
1.  **Initialization:** Create a `FragmentSdfShaderPatch` instance with the voxel grid parameters. Allocate `uCarveMask` with dimensions $(N_x, N_y, N_z)$ and initialize all texels to 0.
2.  **Shader Injection:** Use `THREE.Material.onBeforeCompile` to modify the fragment shader source string.
3.  **Uniform Declaration:** Insert uniform declarations for `uCarveMask`, `uVoxelOrigin`, `uVoxelSizeInv`, `uVoxelCountsInv`, `uCarveBoundsMin`, `uCarveBoundsMax`, and `uCarveCount` before the `out vec4 fragColor;` line.
4.  **Logic Insertion:** Insert the SDF discard logic immediately before the main body opener `void main() { vec4 rgba = vRgba;`.
5.  **Carve Operation:** To carve a cell, flip the corresponding byte in the backing `Uint8Array` to 1, set `uCarveMask.needsUpdate = true`, and update the union AABB (`uCarveBoundsMin`, `uCarveBoundsMax`) to include the new cell. Increment `uCarveCount`.
6.  **Render:** The GPU performs a 3D texture lookup for each fragment. If the fragment is inside the AABB and the texture value is $> 0.5$, the fragment is discarded.

**Pseudocode:**
The injected GLSL logic (inserted at the start of `main()`, before `vec4 rgba = vRgba;`):

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
// Injected Uniforms
uniform sampler3D uCarveMask;
uniform vec3      uVoxelOrigin;
uniform float     uVoxelSizeInv;
uniform vec3      uVoxelCountsInv;
uniform vec3      uCarveBoundsMin;
uniform vec3      uCarveBoundsMax;
uniform int       uCarveCount;

// Injected Logic
if (uCarveCount > 0) {
    // Early-out using AABB
    if (vWorldPos.x >= uCarveBoundsMin.x && vWorldPos.x <= uCarveBoundsMax.x &&
        vWorldPos.y >= uCarveBoundsMin.y && vWorldPos.y <= uCarveBoundsMax.y &&
        vWorldPos.z >= uCarveBoundsMin.z && vWorldPos.z <= uCarveBoundsMax.z) {
        
        // Map world position to voxel grid coordinates
        vec3 coord = (vWorldPos - uVoxelOrigin) * uVoxelSizeInv;
        
        // Normalize to [0, 1] for texture lookup
        vec3 texCoord = coord * uVoxelCountsInv;
        
        // Check if this voxel is carved
        if (texture(uCarveMask, texCoord).r > 0.5) {
            discard;
        }
    }
}
```

**Parameters:**
-   **Voxel Grid Resolution:** Determined by the scene scale and desired precision. Not given in the sources.
-   **Tile Size:** Inherits from parent [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) (default 32x32 pixels).

**Edge Cases and Pitfalls:**
-   **Precision:** The method relies on `vWorldPos` being accurate. Degenerate input (e.g., splats with zero size) may cause undefined behavior in coordinate mapping.
-   **Boundary Conditions:** The AABB check is inclusive. If `uCarveBoundsMin` or `Max` are not updated correctly when the last carve is removed, the early-out may incorrectly discard fragments.
-   **Texture Updates:** The `needsUpdate` flag must be set on the `Data3DTexture` after modifying the `Uint8Array` to ensure the GPU sees the change.

## Subtopics

### [SDF-based Voxel Carving](12620-sdf-based-voxel-carving.md)
A technique to carve voxels from a 3D Gaussian Splatting scene by rendering an axis-aligned box Signed Distance Field (SDF) with zero opacity. It acts as a visual mask that hides splat samples inside the box while leaving samples outside unchanged, avoiding the thinning artifacts caused by deleting whole splats. Choose this when you need a visual mask that preserves splat integrity outside the carved region.

## Operating characteristics
-   **Time Complexity:** $O(1)$ per-fragment cost via `Data3DTexture` lookup [S1, unofficial].
-   **Memory Complexity:** $O(N_x \times N_y \times N_z)$ for the `Data3DTexture` storing the carve mask [S1, unofficial].
-   **Measured Performance:** p95 latency of 9.6 ms at 256 carves on the `butterfly.spz` scene (177 K splats) [S1, unofficial].
-   **Strengths:** Produces clean, cube-shaped carves with sharp edges; does not require forking the renderer; scales well with the number of carves (no per-splat processing overhead).
-   **Limitations:** Requires modifying the shader source at runtime; performance depends on the cost of 3D texture sampling; limited to voxel-aligned shapes.
-   **When to use:** When interactive carving of 3D Gaussian Splatting scenes is required in a browser-based WebGL environment without modifying the core renderer code.
-   **When not to use:** When non-voxel-aligned shapes are needed, or when the overhead of 3D texture sampling is prohibitive for the target hardware.

## Background
Originating from the `splatcarve` project by Steve Kwon (2026), this method was developed to demonstrate that per-fragment SDF evaluation could produce clean cube-shaped carves on a real 3D Gaussian Splatting scene without forking the renderer [S1, unofficial]. It addresses the failure of the baseline H2 approach (per-splat deletion) which produced rough masking artifacts. The technique is implemented as a shareable WebGL/VFX demo, leveraging `THREE.Material.onBeforeCompile` for shader injection [S1, unofficial].

## Sources
[S1] stevekwon211/splatcarve, https://github.com/stevekwon211/splatcarve, fetched 2026-10-27 (unofficial)
