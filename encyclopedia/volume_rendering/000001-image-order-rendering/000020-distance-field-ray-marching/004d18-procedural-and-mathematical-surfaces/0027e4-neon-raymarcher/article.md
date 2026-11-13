# Neon Raymarcher

Also called: Procedural Synthwave City Renderer, SDF Ray Marching in WebGL2

## Overview
Neon Raymarcher is a real-time WebGL2 renderer that generates an infinite procedural synthwave city using signed-distance fields (SDFs) within a single fullscreen fragment shader, requiring no meshes, textures, or scene graph [S1, unofficial]. It distinguishes itself from general-purpose SDF renderers by employing a deterministic integer-hashing contract shared between CPU and GPU to generate building layouts, and an analytic sine-wave curve to define the road centerline, ensuring perfect synchronization between the rendered visual and the camera path [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Frame time (`uTime`), camera state (`uCamPos`, `uCamRight`, `uCamUp`, `uCamForward`), and resolution (`uResolution`).
*   **Outputs:** A color buffer representing the rendered city scene.

**Data Structures:**
The scene is defined implicitly. No vertex buffers or texture arrays are used for geometry. The only persistent data structures are the uniform variables passed from the CPU and the local variables within the fragment shader.

**Parameters:**
*   `MAX_STEPS`: 128 [S1, unofficial].
*   `MAX_DIST`: 180.0 [S1, unofficial].
*   `BLOCK_WIDTH`: 13.0 [S1, unofficial].
*   `BLOCK_DEPTH`: 14.0 [S1, unofficial].
*   `MARGIN`: 1.5 [S1, unofficial].
*   Road Amplitudes: `RD_A1 = 6.0`, `RD_A2 = 3.0`, `RD_A3 = 1.2` [S1, unofficial].
*   Road Frequencies: `RD_F1 = 0.03141593` ($2\pi/200$), `RD_F2 = 0.06613879` ($2\pi/95$), `RD_F3 = 0.11424174` ($2\pi/55$) [S1, unofficial].
*   Road Phases: `RD_P2 = 1.7`, `RD_P3 = 4.2` [S1, unofficial].

**Steps:**

1.  **Initialization:** Create a WebGL2 context with antialiasing disabled and alpha disabled. Generate a fullscreen triangle using `gl_VertexID` in the vertex shader. Compile and link the vertex shader (`fullscreen.vert.glsl`) and fragment shader (`raymarch.frag.glsl`) [S1, unofficial].
2.  **Uniform Setup:** Cache locations for `uResolution`, `uTime`, `uCamPos`, `uCamRight`, `uCamUp`, and `uCamForward` [S1, unofficial].
3.  **Frame Loop:** On each frame, resize the canvas to `clientSize * devicePixelRatio * renderScale`. Set the viewport. Pass the current time and camera basis to the shader uniforms. Draw a single triangle using `gl.drawArrays(gl.TRIANGLES, 0, 3)` [S1, unofficial].
4.  **Ray Generation (Fragment Shader):** For each pixel, compute the ray origin $\mathbf{o}$ from `uCamPos`. Compute the ray direction $\mathbf{d}$ using the pixel's normalized UV coordinates $(u, v)$, the resolution, and the camera basis vectors:
    $$ \mathbf{d} = \text{normalize}(\mathbf{f} + u \cdot \mathbf{r} + v \cdot \mathbf{u}) $$
    where $\mathbf{f}$ is `uCamForward`, $\mathbf{r}$ is `uCamRight`, and $\mathbf{u}$ is `uCamUp` [S1, unofficial].
5.  **Sphere Tracing:** Iterate up to `MAX_STEPS` (128). In each step $i$:
    *   Evaluate the scene SDF at the current point $\mathbf{p} = \mathbf{o} + t \cdot \mathbf{d}$.
    *   If the distance $d < \epsilon$ (convergence threshold), return the hit color.
    *   If $t > \text{MAX\_DIST}$ (180.0), return the background color.
    *   Advance $t$ by $d$ (sphere tracing step).
6.  **Scene SDF Evaluation:** The scene SDF $S(\mathbf{p})$ is the union of the ground plane and the buildings.
    *   **Ground Plane:** $S_{ground}(\mathbf{p}) = \mathbf{p}_y$ (assuming $y=0$ is the ground) [S1, unofficial].
    *   **Road Offset:** Compute the lateral offset of the road centerline at the ray's $z$ coordinate:
        $$ \text{roadX}(z) = \text{RD\_A1} \sin(z \cdot \text{RD\_F1}) + \text{RD\_A2} \sin(z \cdot \text{RD\_F2} + \text{RD\_P2}) + \text{RD\_A3} \sin(z \cdot \text{RD\_F3} + \text{RD\_P3}) $$
    *   **Building Placement:** Determine the building cell $(c_x, c_y, c_z)$ for point $\mathbf{p}$. The $z$-cell is derived from the ray's $z$ position relative to the road. The $x$-cell is derived from the lateral distance from the road centerline $\text{roadX}(z)$. The $y$-cell is derived from the height.
    *   **Hashing:** Compute a deterministic 32-bit seed for the cell using `hashCell(c_x, c_y, c_z)`. This function uses integer bit-mixing to ensure CPU/GPU consistency [S1, unofficial].
    *   **Building SDF:** Use the hash seed to determine building properties (footprint, height, material). Construct the SDF for the building box, shifted to the cell's position and offset by the road curve. Apply the margin (1.5 units) to prevent overlap [S1, unofficial].
    *   **Union:** $S(\mathbf{p}) = \min(S_{ground}(\mathbf{p}), S_{buildings}(\mathbf{p}))$ [S39, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_STEPS = 128
MAX_DIST = 180.0
BLOCK_WIDTH = 13.0
BLOCK_DEPTH = 14.0
MARGIN = 1.5
RD_A1 = 6.0
RD_A2 = 3.0
RD_A3 = 1.2
RD_F1 = 0.03141593
RD_F2 = 0.06613879
RD_F3 = 0.11424174
RD_P2 = 1.7
RD_P3 = 4.2

def road_x(z):
    return (RD_A1 * sin(z * RD_F1) +
            RD_A2 * sin(z * RD_F2 + RD_P2) +
            RD_A3 * sin(z * RD_F3 + RD_P3))

def uhash(x):
    x = x & 0xFFFFFFFF
    x ^= (x >> 16)
    x = imul(x, 0x7feb352d)
    x ^= (x >> 15)
    x = imul(x, 0x846ca68b)
    x ^= (x >> 16)
    return x & 0xFFFFFFFF

def hash_cell(cx, cy, cz):
    h = uhash(imul(cx, 0x9e3779b9))
    h = uhash(h ^ imul(cy, 0x85ebca6b))
    h = uhash(h ^ imul(cz, 0xc2b2ae35))
    return h & 0xFFFFFFFF

def hash_to_float(h):
    return ((h >> 8) / 16777216.0)

def sdf_ground(p):
    return p.y

def sdf_building(p, cell_seed):
    # Determine building dimensions and position based on cell_seed
    # This is a placeholder for the specific building generation logic
    # which uses hash_to_float to derive width, height, and offset
    # within the block.
    # For implementation, one would decode the seed to get:
    #   width = BLOCK_WIDTH * (0.5 + hash_to_float(seed) * 0.5)
    #   height = ...
    #   offset_x = ...
    #   offset_y = ...
    # Then compute the distance to the box centered at (offset_x, offset_y, 0)
    # in the local cell space.
    pass

def scene_sdf(p):
    # Calculate road offset at current z
    rx = road_x(p.z)
    
    # Transform point to "road space"
    # The road runs along Z, so we shift X by the road curve
    p_local_x = p.x - rx
    
    # Determine cell coordinates
    # Assuming cells are aligned to BLOCK_WIDTH and BLOCK_DEPTH
    cx = floor(p_local_x / BLOCK_WIDTH)
    cz = floor(p.z / BLOCK_DEPTH)
    cy = floor(p.y / BLOCK_DEPTH) # Assuming similar vertical tiling
    
    # Get building SDF
    seed = hash_cell(cx, cy, cz)
    d_build = sdf_building(p, seed)
    
    # Union with ground
    d_ground = sdf_ground(p)
    
    return min(d_ground, d_build)

def render_pixel(uv, cam_pos, cam_right, cam_up, cam_forward):
    # Ray origin and direction
    o = cam_pos
    d = normalize(cam_forward + uv.x * cam_right + uv.y * cam_up)
    
    t = 0.0
    for i in range(MAX_STEPS):
        p = o + t * d
        d_val = scene_sdf(p)
        
        if d_val < 0.001:
            return compute_color(p, d) # Placeholder for shading
        if t > MAX_DIST:
            return background_color()
        
        t += d_val
        
    return background_color()
```

**Edge Cases and Pitfalls:**
*   **Precision:** The hash function uses 32-bit integer arithmetic to ensure bit-identical results between JavaScript (CPU) and GLSL (GPU). Using floating-point hashing (e.g., `fract(sin(...))`) would cause drift and mismatched geometry [S1, unofficial].
*   **Road Curvature:** The road curve is designed to have a maximum heading of ~28 degrees and a minimum radius of curvature of ~29 units to ensure drivability and visual coherence [S1, unofficial].
*   **Overlap:** The margin of 1.5 units prevents adjacent building cells from overlapping, ensuring the SDF union remains valid [S1, unofficial].

## Subtopics

### Terrain Raymarching
Renders heightmap-based terrain using SDFs. Differs from Neon Raymarcher by using a 2D heightmap instead of a 3D procedural hash for geometry. Choose for natural landscapes. [Terrain Raymarching](1357-terrain-raymarching.md)

### Menger Sponge Raymarching
Renders fractal geometry using recursive SDF operations. Differs by using iterative min/max operations for fractal detail rather than hash-based placement. Choose for mathematical fractals. [Menger Sponge Raymarching](4875-menger-sponge-raymarching.md)

### Fractals and Mathematical Surfaces
General category for rendering fractals via SDFs. Neon Raymarcher is a specific application using hash-based placement. [Fractals and Mathematical Surfaces](10024-fractals-and-mathematical-surfaces.md)

### Signed Distance Field Representations and Operations
Defines the mathematical foundation (union, intersection, subtraction) used by Neon Raymarcher. [Signed Distance Field Representations and Operations](13339-signed-distance-field-representations-and-operations.md)

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$ where $N$ is the number of pixels and $S$ is the number of steps (128). The SDF evaluation is $O(1)$ per step.
*   **Memory Complexity:** $O(1)$ auxiliary memory. No scene graph or geometry buffers are stored.
*   **Performance:** Runs in real-time on WebGL2-capable hardware. Performance is limited by fragment shader complexity and fill rate.
*   **Strengths:** Infinite procedural generation, no asset loading, deterministic CPU/GPU sync.
*   **Limitations:** Limited to procedural geometry; cannot render arbitrary meshes. Resolution limited by WebGL2 capabilities.

## Background
Neon Raymarcher is a WebGL2 experiment by PIGJET, published on GitHub [S1, unofficial]. It demonstrates the capability of modern fragment shaders to render complex procedural scenes without traditional graphics pipeline assets. The project emphasizes the importance of deterministic hashing for cross-platform consistency in procedural generation [S1, unofficial].

## Sources
[S1] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
