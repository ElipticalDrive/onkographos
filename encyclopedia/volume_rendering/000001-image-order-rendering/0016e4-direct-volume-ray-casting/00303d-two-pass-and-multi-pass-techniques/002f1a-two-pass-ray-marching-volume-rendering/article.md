# Two-Pass Ray Marching Volume Rendering

Also called: INTMeshVolumeRendering, Two-Pass Raycasting

## Overview
Two-Pass Ray Marching Volume Rendering is a GPU-based direct volume rendering technique that decouples the determination of ray entry and exit points from the actual volume integration [S1, unofficial][S2, unofficial]. Unlike single-pass raycasting which computes boundaries and compositing in one shader execution, this method uses two distinct rendering passes: the first pass renders the volume bounding box faces to establish depth ranges, and the second pass performs the raymarching integration using those ranges [S1, unofficial]. This separation allows for efficient handling of complex boundary intersections and supports features like Maximum Intensity Projection (MIP) and cutting planes by isolating geometric setup from data sampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D volume data (scalar field), camera position and orientation, volume bounding box dimensions, transfer function definition (color/alpha mapping), and marching parameters (step count).
*   **Output:** A 2D color image (framebuffer) representing the volume rendered from the camera's perspective.

**Data Structures:**
1.  **Volume Data Texture (`dataTex`):**
    *   Format: `THREE.RedFormat` (or `AlphaFormat` in some implementations) with `FloatType` [S1, unofficial][S2, unofficial].
    *   Layout: The 3D volume is flattened into a 2D texture. Width is set to the hardware maximum texture size (`maxTextureSize`) [S2, unofficial]. Height is calculated as `ceil(data.length / maxTextureSize)` [S2, unofficial].
    *   Padding: Data is padded with zeros to fill the rectangular texture dimensions [S2, unofficial].
    *   Normalization: Input voxel values (often 12-bit uint16) are normalized to [0, 1] by dividing by 4095.0 [S1, unofficial].
2.  **Transfer Function Texture (`transfertTex`):**
    *   Format: `RGBAFormat` with `FloatType` or `UnsignedByteType` [S1, unofficial][S2, unofficial].
    *   Layout: 1D texture, typically 8x256 pixels [S2, unofficial]. Each pixel stores [R, G, B, A] values [S1, unofficial].
    *   Generation: A linear gradient is drawn on a canvas where color stops are defined by user control points [S2, unofficial].
3.  **Intermediate Render Target (`rtTexture`):**
    *   Format: `RGBAFormat`, `FloatType` [S2, unofficial].
    *   Dimensions: Matches the screen resolution (e.g., `window.innerWidth` x `window.innerHeight`) [S2, unofficial].
    *   Purpose: Stores depth/position information from the first pass [S1, unofficial].
4.  **Geometry:**
    *   A `BoxGeometry` representing the volume bounding box, configured as `doubleSided = true` [S2, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Create the `BoxGeometry` with the volume's physical dimensions [S2, unofficial].
    *   Create the `rtTexture` render target with screen dimensions and linear filtering [S2, unofficial].
    *   Load volume data: Compute `minData` and `maxData` if not provided. Pad the data array to fit the 2D texture constraints. Create `dataTex` [S2, unofficial].
    *   Create `transfertTex` from color/alpha control points [S2, unofficial].

2.  **First Pass (Boundary Determination):**
    *   Create a scene (`_sceneFirstPass`) containing the volume box mesh.
    *   Set the mesh material to use a vertex/fragment shader pair (`firstPassVertex`, `firstPassFragment`) and set `side: THREE.BackSide` [S2, unofficial].
    *   Render this scene to `rtTexture` [S2, unofficial].
    *   *Note:* Some implementations render both front and back faces to separate targets (`frontData`, `backData`) to explicitly capture entry and exit depths [S1, unofficial]. The provided notes emphasize rendering the back face to `rtTexture` [S2, unofficial].

3.  **Second Pass (Ray Marching Integration):**
    *   Create a fullscreen quad scene (`quadData`) using a `RaycastingShader` [S1, unofficial].
    *   Configure the shader uniforms:
        *   `tex`: The `rtTexture` from Pass 1 [S2, unofficial].
        *   `dataTex`: The volume data texture [S2, unofficial].
        *   `transfertTex`: The transfer function texture [S2, unofficial].
        *   `steps`: Number of marching steps (default 50) [S2, unofficial].
        *   `alphaCorrection`: Opacity correction factor (default 0.2) [S2, unofficial].
        *   `size`: Volume bounding box dimensions [S2, unofficial].
        *   `minData`, `maxData`: Data range for normalization [S2, unofficial].
        *   `widthDataTex`, `heightDataTex`: Dimensions of the data texture [S2, unofficial].
    *   Render the fullscreen quad to the default framebuffer (screen) [S1, unofficial].

**Formulas:**

*   **Transfer Function Lookup:**
    For a voxel value $v$ normalized to $[0, 1]$:
    $$ \text{color} = \text{sample}(\text{transfertTex}, v) $$
    The transfer function texture is generated such that for index $i$ from $0$ to $resolution-1$:
    $$ pos = \frac{i}{resolution - 1} $$
    $$ c = \text{d3.color}(\text{colorScale}(pos)) $$
    $$ \alpha = \text{alphaScale}(pos) $$
    $$ \text{tfData}[i \cdot 4 \dots i \cdot 4 + 3] = [c.r, c.g, c.b, \text{Math.round}(\alpha \cdot 255)] $$
    where `colorScale` maps positions to RGB colors and `alphaScale` maps positions to intensity [S1, unofficial].

*   **Ray Marching Integration (Standard Over-Compositing):**
    For each ray, starting from entry point $t_{start}$ to exit point $t_{end}$ with step size $\Delta t$:
    $$ C_{out} = 0, \quad \alpha_{out} = 0 $$
    For $t = t_{start}$ to $t_{end}$ with step $\Delta t$:
    $$ v = \text{sample}(\text{dataTex}, \text{rayPos}(t)) $$
    $$ \text{normalized\_v} = \frac{v - \text{minData}}{\text{maxData} - \text{minData}} $$
    $$ \text{tf} = \text{sample}(\text{transfertTex}, \text{normalized\_v}) $$
    $$ \alpha_{step} = \text{tf}.a \cdot \alpha_{Correction} $$
    $$ C_{out} = C_{out} + \text{tf}.rgb \cdot \alpha_{step} \cdot (1 - \alpha_{out}) $$
    $$ \alpha_{out} = \alpha_{out} + \alpha_{step} \cdot (1 - \alpha_{out}) $$
    $$ \text{if } \alpha_{out} > \text{threshold} \text{ break (Early Ray Termination)} $$ [G]

**Parameters:**
*   `steps`: Number of marching steps per ray. Default: 50. Higher values improve quality but reduce FPS [S2, unofficial].
*   `alphaCorrection`: General opacity correction factor. Default: 0.2 [S2, unofficial].
*   `resolution`: Resolution of the transfer function texture. Default: 255 [S1, unofficial].
*   `size`: Dimensions (x, y, z) of the volume bounding box [S2, unofficial].
*   `dataMin` / `dataMax`: Min/max of dataset. Computed automatically if undefined [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Missing Parameters:** Omitting renderer, camera, or size causes silent failure or runtime errors. Guards should check for these [S2, unofficial].
*   **Cutting Planes:** If using cutting planes, the `u_enableCut` uniform must be updated. `u_planeOrigin` and `u_planeNormal` must reflect the world position and orientation of the cutting plane mesh before rendering [S1, unofficial].
*   **Auto Clear:** When rendering additional elements (like cutting plane visualizations) after the raycaster, `renderer.autoClear` must be set to `false` before the raycaster pass and `true` afterwards to prevent framebuffer clearing [S1, unofficial].
*   **Precision:** Using `FloatType` for render targets and data textures is critical for accurate transfer function lookups and compositing [S2, unofficial].

## Subtopics

### [Two-Pass GPU Volume Ray Casting for Light Field Displays](6523-two-pass-gpu-volume-ray-casting-for-light-field-displays.md)
Specialized for light field displays, decoupling 3D ray-casting from 2D post-processing for multi-projector distortion correction.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of marching steps. The two-pass structure adds a constant overhead for the boundary pass but allows for optimized integration in the second pass.
*   **Memory Complexity:** $O(W \cdot H + V)$, where $W, H$ are screen dimensions (for `rtTexture`) and $V$ is the volume data size (for `dataTex`). The `rtTexture` requires `FloatType` RGBA storage, which is memory-intensive [S2, unofficial].
*   **Performance:** Depends heavily on `steps` and texture memory bandwidth. Default 50 steps provide a balance for interactive rates on modern GPUs [S2, unofficial].
*   **Strengths:** Separates geometric complexity from data sampling; supports MIP and cutting planes easily; compatible with older hardware that lacks conditional discard in fragment shaders (via segmented traversal principles, though this specific implementation focuses on boundary decoupling) [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires two full-screen passes; high memory usage for float render targets; padding artifacts if data dimensions do not align with texture width [S2, unofficial].

## Background
This method is implemented in the `INTMeshVolumeRendering` class for WebGL-based volume visualization in Three.js [S2, unofficial]. It was developed as part of a Visualization course project (TU Wien) and open-source implementations exist on GitHub [S1, unofficial][S2, unofficial]. It builds on the general principles of two-pass rendering where state is persisted across passes via off-screen textures [Parent].

## Sources
[S1] SubairProject1/Volume-Rendering-Visualization-Project, https://github.com/SubairProject1/Volume-Rendering-Visualization-Project, fetched 2026-10-26 (unofficial)
[S2] pyx61198/volume_rendering, https://github.com/pyx61198/volume_rendering, fetched 2026-10-27 (unofficial)
