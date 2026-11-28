# Web 3D Gaussian Splatting

Also called: SplatNode

## Overview
Web 3D Gaussian Splatting renders radiance-field captures (3D Gaussian Splatting) on the web using Kotlin/JS and Filament.js with a WebGL2 backend [S1, unofficial]. It solves the problem of displaying high-fidelity 3D Gaussian splats in a browser environment without requiring native plugins or CUDA dependencies [S2, unofficial]. It differs from compute-shader-based accumulation approaches by leveraging standard hardware-instanced quads and a back-to-front painter's sort that re-runs on camera motion [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A splat cloud file in `.ply` (INRIA format) or `.spz` (Niantic format) [S1, unofficial]. Alternatively, an in-memory `SplatCloud` object constructed from arrays [S1, unofficial].
*   **Output:** A 2D image rendered via WebGL2, using premultiplied-alpha blending [S1, unofficial].

**Data Structures:**
*   **Per-splat data:** Stored in `RGBA16F` textures [S1, unofficial].
*   **Primitive:** Camera-facing Gaussian discs, implemented as hardware-instanced quads [S1, unofficial].
*   **Material:** `splat_web.filamat` compiled against the web Filament runtime (MATERIAL_VERSION 52) [S1, unofficial].

**Parameters:**
*   **Shape:** Isotropic billboards calculated via max-axis scale [S1, unofficial]. Anisotropic screen-space ellipses are planned but not implemented [S1, unofficial].
*   **Color:** SH degree-0 colour (zeroth-order spherical harmonics) [S1, unofficial].

**Steps:**
1.  **Load:** Fetch the `.ply` or `.spz` file. Parse it using the shared KMP `sceneview-core` parsers (`SplatParser.fromPly`, `.fromSpz`, or `.parse`) [S1, unofficial].
2.  **Initialize:** Create a `NodeHandle` (or `SplatNode`) in the scene graph [S1, unofficial].
3.  **Sort:** Perform a back-to-front painter's sort based on the current camera position [S1, unofficial].
4.  **Render:** Draw the instanced quads using the `RGBA16F` texture data and premultiplied-alpha blending [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
def add_splat_node(viewer, url):
    # Fetch and parse the file
    bytes = fetch_file(url)
    splat_cloud = SplatParser.parse(bytes)
    
    # Create the node handle
    node_handle = viewer.add_splat_node(splat_cloud)
    return node_handle

# Rendering Loop (triggered on camera motion)
def render_splat_node(node_handle, camera_position):
    # 1. Update sort order based on live camera position
    # The web node automatically feeds the live camera position to avoid popping
    sorted_indices = back_to_front_sort(node_handle.splats, camera_position)
    
    # 2. Upload or bind the sorted indices if changed
    # Note: If indices are unchanged, skip upload to save bandwidth
    if indices_changed:
        upload_index_buffer(sorted_indices)
        
    # 3. Draw instanced quads
    # Uses RGBA16F textures for per-splat data
    # Uses isotropic billboards (max-axis scale)
    # Uses SH degree-0 colour
    # Uses premultiplied-alpha blending
    draw_instanced_quads(node_handle.mesh, sorted_indices)
```

**Edge Cases and Pitfalls:**
*   **Compositing Order:** Without a `cameraPositionProvider`, the compositing order stays as loaded, causing popping when orbiting [S1, unofficial]. The web node automatically feeds the live camera position to avoid this [S1, unofficial].
*   **Performance:** Sorting millions of Gaussians can be expensive. Offloading sorting to an asynchronous WebWorker can help break the 120 display FPS limit [S2, unofficial].

## Subtopics

### WebGL Mobile Rendering Framework
A cross-platform real-time rendering framework using WebGL to render Mobile-4DGS in browsers without CUDA dependencies [S2, unofficial]. It uses decoupled sorting to break the 120 FPS display limit [S2, unofficial]. It offloads depth-sorting of millions of Gaussians to an asynchronous WebWorker and uses the CPU for visibility order update [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Sorting is $O(N \log N)$ where $N$ is the number of Gaussians [S2, unofficial].
*   **Memory Complexity:** Per-splat data is stored in `RGBA16F` textures [S1, unofficial].
*   **Performance:** Can break the 120 display FPS limit by decoupling sorting from the main thread [S2, unofficial].
*   **Strengths:** Cross-platform, no CUDA dependencies, real-time rendering in browsers [S2, unofficial].
*   **Limitations:** Isotropic billboards only (no anisotropic ellipses yet), SH degree-0 colour only [S1, unofficial].

## Background
Web 3D Gaussian Splatting is part of the SceneView project, specifically tracked under issue #2646 [S1, unofficial]. It builds on the Android P1 implementation, porting it to the web using Kotlin/JS and Filament.js [S1, unofficial]. The Mobile-4DGS framework extends this to mobile browsers, addressing the non-commutative nature of alpha-compositing through decoupled sorting [S2, unofficial].

## Sources
[S1] feat(sceneview-web): SplatNode — 3D Gaussian Splatting on the web ( #2646, https://github.com/sceneview/sceneview/commit/fb9d3303b9b73fbec55e227395abaef0b39076dc, fetched 2026-10-29 (unofficial)
[S2] Mobile-4DGS: Unified Static-Dynamic Real-time Mobile Gaussian Splatting, https://arxiv.org/abs/2610.05289, fetched 2026-10-29 (unofficial)
