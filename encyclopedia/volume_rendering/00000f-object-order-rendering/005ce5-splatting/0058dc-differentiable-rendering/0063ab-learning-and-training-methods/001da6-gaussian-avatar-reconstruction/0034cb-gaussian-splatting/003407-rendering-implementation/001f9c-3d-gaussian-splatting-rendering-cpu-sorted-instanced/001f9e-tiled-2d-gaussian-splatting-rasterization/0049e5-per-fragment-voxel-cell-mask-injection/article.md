# Per-fragment Voxel-Cell Mask Injection

Also called: Fragment SDF Shader Patch, H2′

## Overview
Per-fragment Voxel-Cell Mask Injection is a technique for carving cube-shaped holes in 3D Gaussian Splatting scenes by injecting a Signed Distance Field (SDF) evaluation into the fragment shader at runtime. It solves the visual artifacts caused by deleting whole splats (the "H2" baseline) by operating at the fragment level, producing clean, sharp cube edges without forking the underlying renderer [S1, unofficial]. Unlike its sibling [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) which manages opacity globally, this method uses a 3D texture lookup to discard specific fragments, leaving neighboring voxels untouched [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D Gaussian Splatting scene rendered via a WebGL renderer (specifically tested with Spark.js/Three.js), a voxel grid defining the carving space, and a set of active carve cells.
*   **Output:** A rendered image where fragments falling within active carve cells are discarded, creating a hole with the exact shape of the voxel grid cells.

**Data Structures:**
The core state is managed by a `FragmentSdfShaderPatch` class exposing the following uniforms to the shader:
1.  `uCarveMask`: A `Data3DTexture` (sampler3D) with dimensions matching the voxel grid counts $(N_x, N_y, N_z)$. Data type is `Uint8Array`, initialized to 0 (not carved).
2.  `uVoxelOrigin`: A `vec3` representing the origin of the voxel grid in world space.
3.  `uVoxelSizeInv`: A `float` equal to $1 / \text{voxelSize}$.
4.  `uVoxelCountsInv`: A `vec3` equal to $1 / (N_x, N_y, N_z)$.
5.  `uCarveBoundsMin` / `uCarveBoundsMax`: `vec3` uniforms defining the union Axis-Aligned Bounding Box (AABB) of all active carve cells for early-out optimization.
6.  `uCarveCount`: An integer uniform tracking the number of active carve cells.

**Numbered Steps:**
1.  **Initialization:** Create a `FragmentSdfShaderPatch` instance. Allocate a `Data3DTexture` sized according to the voxel grid counts. Initialize all texels to 0.
2.  **Shader Injection:** Use the renderer's `Material.onBeforeCompile` hook to modify the compiled fragment shader source code.
3.  **Uniform Declaration:** Insert the uniform declarations listed above before the `out vec4 fragColor;` statement.
4.  **Logic Insertion:** Insert the SDF discard logic block immediately before the main body opener `void main() { vec4 rgba = vRgba;`.
5.  **Carve Operation:** To carve a cell, flip the corresponding byte in the backing `Uint8Array` to 1. Update `uCarveMask.needsUpdate = true` to trigger a GPU texture upload. Expand the union AABB to include the new cell.
6.  **Uncarve Operation:** Flip the byte back to 0. Update the texture. Recalculate the union AABB if necessary.

**Pseudocode:**
The injected GLSL logic, inserted at the start of the fragment shader's `main()` function:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
// Uniforms declared before main()
uniform sampler3D uCarveMask;
uniform vec3      uVoxelOrigin;
uniform float     uVoxelSizeInv;
uniform vec3      uVoxelCountsInv;
uniform vec3      uCarveBoundsMin;
uniform vec3      uCarveBoundsMax;
uniform int       uCarveCount;

in vec3 vWorldPos;

void main() {
    // Inserted Logic Start
    if (uCarveCount > 0) {
        // Early-out AABB check
        if (vWorldPos.x >= uCarveBoundsMin.x && vWorldPos.x <= uCarveBoundsMax.x &&
            vWorldPos.y >= uCarveBoundsMin.y && vWorldPos.y <= uCarveBoundsMax.y &&
            vWorldPos.z >= uCarveBoundsMin.z && vWorldPos.z <= uCarveBoundsMax.z) {
            
            // Map world position to voxel grid coordinates
            vec3 coord = (vWorldPos - uVoxelOrigin) * uVoxelSizeInv;
            
            // Normalize to [0, 1] range for texture lookup
            vec3 texCoord = coord * uVoxelCountsInv;
            
            // Check if the fragment falls within a carved cell
            if (texture(uCarveMask, texCoord).r > 0.5) {
                discard;
            }
        }
    }
    // Inserted Logic End

    vec4 rgba = vRgba;
    // ... rest of original shader
}
```

**Parameters:**
*   **Voxel Grid Resolution:** Determined by the scene scale and desired carving precision. Not given in the sources.
*   **Tile Size:** Inherits from parent [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md), default 32x32 pixels.

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on `vWorldPos` being accurate. If world-space reconstruction is imprecise, the cube edges may appear jagged or misaligned.
*   **Boundary Conditions:** The AABB early-out is critical for performance. If the union AABB is not updated correctly when adding/removing carves, fragments may be incorrectly discarded or not discarded.
*   **Texture Updates:** The `needsUpdate` flag must be set on the `Data3DTexture` after modifying the `Uint8Array` to ensure the GPU receives the new data.

## Subtopics

### [SDF-based Voxel Carving](12620-sdf-based-voxel-carving.md)
A variant that renders an axis-aligned box SDF with zero opacity to hide splats inside the box. It avoids thinning artifacts by operating per-sample rather than deleting whole splats. Choose this when you need to hide volumetric regions without affecting neighboring splats.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per-fragment cost via `Data3DTexture` lookup [S1, unofficial].
*   **Memory Complexity:** Proportional to the voxel grid volume $(N_x \times N_y \times N_z)$ bytes for the `Uint8Array` backing the `Data3DTexture`.
*   **Measured Performance:** p95 latency of 9.6 ms for 256 carves on a scene with 177 K splats (butterfly.spz) [S1, unofficial].
*   **Strengths:** Produces sharp, clean cube-shaped edges; does not require forking the renderer; scales well with the number of carves (no per-splat sorting or deletion overhead).
*   **Limitations:** Requires WebGL 2.0+ for `Data3DTexture` support; carving is limited to axis-aligned cubes defined by the voxel grid; performance depends on the cost of the 3D texture lookup.
*   **When to use:** When precise, geometrically clean carving is required and the underlying renderer allows shader injection.
*   **When not to use:** When non-cubic carving shapes are needed, or when the renderer does not support dynamic shader modification or 3D textures.

## Background
Originating from the `splatcarve` project by Steve Kwon (2026), this method was developed to demonstrate that browser-based 3D Gaussian Splatting editors could perform complex volumetric edits without forking the rendering engine [S1, unofficial]. It addresses the limitations of the "H2" baseline, which deleted whole splats grouped by voxel, resulting in rough, artifact-prone masks. The "H2′" breakthrough showed that injecting SDF evaluation via `THREE.Material.onBeforeCompile` could achieve interactive carving latencies with high visual fidelity [S1, unofficial].

## Sources
[S1] stevekwon211/splatcarve, https://github.com/stevekwon211/splatcarve, fetched 2026-10-27 (unofficial)
