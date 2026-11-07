# Multi-pass GPU Ray-Casting with Early Ray Termination

Also called: DirectX9 volume ray-casting, GPU-based volume ray-casting, Multi-pass GPU Ray Casting, Pixel Shader 2.0 volume ray-casting, Stream model for volume ray-casting on DirectX9-level graphics hardware, Stream programming model for volume ray-casting

## Overview
This method implements direct volume ray casting on DirectX 9-level programmable graphics hardware (specifically Pixel Shader 2.0) by splitting the ray traversal into multiple rendering passes [S1, unofficial]. It solves the performance bottleneck of single-pass ray casting on early GPUs by decoupling ray direction computation, traversal, and termination checking into separate stages, utilizing 2D texture render targets to persist state between passes [S1, unofficial][S2, unofficial]. Unlike single-pass approaches that require dynamic flow control (available in later shader models), this method relies on hardware early z-testing and fixed-function blending to achieve early ray termination and empty-space skipping, significantly reducing fragment operations for opaque structures or void regions [S1, unofficial].

## How it works
The algorithm uses a stream programming model where the GPU rasterizer drives the ray intersection logic via geometry proxies (bounding box faces), and fragment shaders perform the volumetric integration [S1, unofficial].

### Inputs and Outputs
*   **Input:** 3D texture volume data $V$, transfer function $TF$, view matrix, projection matrix.
*   **Output:** 2D color buffer (final image).
*   **Intermediate Textures:**
    *   `TMP`: 2D RGB texture, same resolution as viewport. Stores the 3D texture coordinates of the first intersection point (entry) of the ray with the volume [S1, unofficial].
    *   `DIR`: 2D RGBA floating-point texture. Color components store the normalized ray direction in local texture coordinates; alpha component stores the length of the non-normalized direction (ray length within volume) [S1, unofficial].
    *   `RES`: 2D texture render target. Accumulates the ray-casting result (color and opacity) across main passes [S1, unofficial].

### Parameters
*   $M$: Number of ray steps performed in each main pass. Typical value: 8 [S1, unofficial].
*   $T$: Opacity threshold for early ray termination.
*   $\Delta$: Step size along the ray.
*   $N$: Total number of rendering passes.

### Data Structures and Layout
1.  **TMP**: Resolution $W \times H$ (viewport size). Format: RGB. Content: $(x_{entry}, y_{entry}, z_{entry})$ in texture space [S1, unofficial].
2.  **DIR**: Resolution $W \times H$. Format: RGBA32F (floating point). Content: $(d_x, d_y, d_z, L)$ where $(d_x, d_y, d_z)$ is the normalized direction and $L$ is the unnormalized length [S1, unofficial].
3.  **RES**: Resolution $W \times H$. Format: RGBA. Content: Accumulated color $(C_r, C_g, C_b)$ and accumulated opacity $\alpha_{acc}$ [S1, unofficial].

### Numbered Steps

1.  **Pass 1: Entry Point Determination**
    *   Render the **front faces** of the volume bounding box.
    *   Set depth test to `GREATER`.
    *   Issue 3D texture coordinates of each vertex as per-vertex color `COL`.
    *   Render to `TMP`.
    *   Result: `TMP[x][y]` contains the entry point coordinates for the ray passing through pixel $(x,y)$ [S1, unofficial].

2.  **Pass 2: Ray Direction Determination**
    *   Render the **back faces** of the volume bounding box.
    *   Set depth test to `GREATER`.
    *   For each fragment, fetch `TMP` value at current texture coordinate.
    *   Compute normalized ray direction: $\vec{d} = \text{normalize}(\text{COL} - \text{TMP}[x][y])$.
    *   Compute unnormalized length: $L = \|\text{COL} - \text{TMP}[x][y]\|$.
    *   Render $\vec{d}$ to RGB components and $L$ to the Alpha component of `DIR` [S1, unofficial].

3.  **Main Passes 3 to N: Ray Traversal**
    *   Render the **front faces** of the volume bounding box.
    *   **Vertex Attributes:**
        *   Texture Coordinate 1: Normalized device coordinates $(x,y)$ of the vertex. Used to index `DIR` and `RES`.
        *   Per-vertex Color: 3D texture coordinates of the vertex (entry point $C$).
    *   **Constants:**
        *   $C_1 = \Delta$ (step size).
        *   $C_2 = \Delta \cdot M \cdot (k)$, where $k$ is the number of steps already performed (initially 0 for Pass 3, $M$ for Pass 4, etc.).
    *   **Fragment Shader Logic:**
        1.  Lookup ray direction: $\vec{dir} = \text{DIR}[x][y]$.
        2.  Lookup accumulated result: $Result_{old} = \text{RES}[x][y]$.
        3.  Initialize parametric ray equation: $r(t) = C + t \cdot \vec{dir}$.
        4.  Set initial $t = C_2$.
        5.  Loop $i$ from 0 to $M-1$:
            *   Increment $t \leftarrow t + C_1$.
            *   Fetch volume value: $v = V(r(t))$.
            *   Compute color contribution $c$ and opacity $\alpha$ using transfer function.
            *   Composite with $Result_{old}$ using standard volume rendering equation (e.g., front-to-back compositing).
            *   Update $Result_{old}$.
        6.  Check if ray has left volume (opacity > threshold $T$ or specific iso-value hit). If so, write opacity 1 to `RES` to force termination in subsequent passes [S1, unofficial].
        7.  Write final $Result_{old}$ to `RES` [S1, unofficial].

4.  **Intermediate Passes 3 to N: Stopping Criterion Check**
    *   *Note: This pass is required because early z-test is disabled when the fragment shader replaces the z-value [S1, unofficial].*
    *   Render the **front faces**.
    *   **Fragment Shader Logic:**
        1.  Fetch accumulated opacity from `RES[x][y]`.
        2.  Conditional statement:
            $$ \text{IF } (\text{RES}[x][y].\alpha > T) \text{ THEN } z = \text{MAX}, \text{out}(0,0,0,0) $$
            $$ \text{ELSE } z = 0, \text{out}(0,0,0,0) $$
        3.  If the ray is terminated (opacity > $T$), set the z-buffer to the maximum value (depth far plane) and output zero color/opacity. This prevents subsequent passes from processing this pixel.
        4.  Otherwise, the shader has no effect on the color buffer, but the z-buffer update enables early z-test for subsequent passes [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Entry Point
def pass_1_entry():
    set_depth_test(GREATER)
    render_target = TMP
    for each pixel (x, y):
        # Rasterizer provides vertex 3D texture coords as color COL
        entry_point = vertex_color
        TMP[x][y] = entry_point

# Pass 2: Ray Direction
def pass_2_direction():
    set_depth_test(GREATER)
    render_target = DIR
    for each pixel (x, y):
        entry_point = TMP[x][y]
        # Rasterizer provides vertex 3D texture coords as color COL (back face)
        exit_point = vertex_color
        direction_vec = exit_point - entry_point
        length = magnitude(direction_vec)
        normalized_dir = direction_vec / length
        DIR[x][y] = (normalized_dir.x, normalized_dir.y, normalized_dir.z, length)

# Pass 3 to N: Ray Traversal
def pass_n_traversal(pass_index):
    set_depth_test(GREATER)
    render_target = RES
    # C2 is the starting t value for this pass
    # C2 = step_size * M * (pass_index - 3)
    start_t = step_size * M * (pass_index - 3)
    
    for each pixel (x, y):
        dir_vec = DIR[x][y].rgb
        accumulated = RES[x][y]
        
        current_t = start_t
        for step in range(M):
            current_t += step_size
            ray_pos = entry_point_from_vertex + current_t * dir_vec
            sample = volume_texture_lookup(ray_pos)
            
            # Compute color and opacity from sample
            c, a = transfer_function(sample)
            
            # Front-to-back compositing
            # new_acc = accumulated + (1 - accumulated.a) * c * a
            # new_acc.a = accumulated.a + (1 - accumulated.a) * a
            accumulated = composite(accumulated, c, a)
            
            if accumulated.a > threshold:
                break
        
        # If ray terminated early, flag it for next pass
        if accumulated.a > threshold:
            accumulated.a = 1.0
            
        RES[x][y] = accumulated

# Intermediate Pass: Stopping Criterion
def pass_n_stop_check(pass_index):
    set_depth_test(GREATER)
    # No render target write for color, only z-buffer
    for each pixel (x, y):
        opacity = RES[x][y].a
        if opacity > threshold:
            set_z_buffer(MAX_DEPTH)
            output_color = (0, 0, 0, 0)
        else:
            set_z_buffer(0) # Reset depth to allow processing
            output_color = (0, 0, 0, 0)
```

### Edge Cases and Pitfalls
*   **Shader Discard Limitation:** Pixel Shader 2.0 does not support conditional execution of expressions or shader discard. Therefore, gradient reconstruction and shading computations are always performed even if no surface is hit, though only once every $M$ steps [S1, unofficial].
*   **Early Z-Test Disablement:** The early z-test is disabled as long as the fragment's z-value is replaced in the shader program. This necessitates the additional intermediate pass for stopping criterion checking [S1, unofficial].
*   **Choice of M:** If $M$ is small, many fragments outside the volume or void samples may be processed unnecessarily (up to $M-1$ void samples per ray segment). If $M$ is large, all rays compute the same number of samples even if shorter, due to the lack of shader discard. The optimal $M$ balances pass overhead against redundant computation. In the reference implementation, $M$ was restricted to 8 due to hardware shader limitations [S1, unofficial].
*   **Empty Space Skipping:** Requires an additional data structure (e.g., octree) encoding empty regions. This is not part of the core multi-pass loop but is integrated via the same acceleration principles [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot W \cdot H \cdot M)$, where $N$ is the number of passes, $W \times H$ is viewport resolution, and $M$ is steps per pass. $N \approx K/M$, where $K$ is the total steps for the longest ray.
*   **Memory Complexity:** $O(W \cdot H)$ for intermediate textures (`TMP`, `DIR`, `RES`).
*   **Performance:** On an ATI 9700 with a 512x512 viewport and 256³ MRI Head data:
    *   Slice-Based Volume Rendering (SBVR): 7.1 fps [S1, unofficial].
    *   Ray Casting (RC, no acceleration): 4.8 fps [S1, unofficial].
    *   RC with Early Termination (RC-α): 14.4 fps [S1, unofficial].
    *   RC with Early Termination + Empty-Space Skipping (RC-β): 23.4 fps [S1, unofficial].
    *   Other datasets (Philips aneurism, GE engine, Philips foot) showed similar trends, with RC-β achieving 17.7–19.6 fps compared to SBVR's 7.1–10.4 fps [S1, unofficial].
*   **Strengths:** Equal sampling density along rays; significant reduction in fragment operations for opaque structures or empty space; effective reduction of texture fetches and arithmetic operations compared to previous texture-based approaches [S1, unofficial].
*   **Limitations:** Requires multiple passes and texture lookups; overhead from rendering front faces and accessing textures in each pass; limited by hardware shader limitations on $M$; early z-test inefficiency due to z-value replacement [S1, unofficial].

## Background
The method was described in a paper detailing a framework using the DirectX 9 SDK (Microsoft, 2002) and programmable via the Pixel Shader 2.0 API [S1, unofficial]. It builds upon prior work such as Cullip and Neumann (1993), Levoy (1990), and Purcell et al. (2002) [S1, unofficial]. The approach leverages the stream programming model to feed data-parallel fragment units, a strategy designed to leverage the intrinsic data parallelism of graphics hardware [S2, unofficial]. It represents a significant advancement in real-time volume rendering on commodity hardware by effectively applying acceleration techniques like early ray termination and empty-space skipping through the graphics pipeline's fixed-function and early programmable features [S1, unofficial][S2, unofficial].

## Sources
[S1] Acceleration Techniques for GPU-based Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Acceleration_Techniques_for_GPU-based_Volume_Rendering/vis03-rc.pdf, fetched 2026-10-26 (unofficial)
[S2] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
