# Focus and Context Multiresolution Rendering

Also called: Interactive lenses multiresolution, Time-critical multiresolution rendering

## Overview
Focus and Context Multiresolution Rendering balances processing capacity between a high-detail focus region and a lower-detail context region to improve interactivity in time-critical applications [S1, unofficial]. It solves the problem of limited hardware resources by decomposing the rendering task into two passes with distinct parameters, explicitly trading seamless integration for speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, a transfer function, a camera view, and a focus region definition (e.g., a data probe) [S1, unofficial].
*   **Output:** A composite image where the focus region is rendered at high fidelity and the context region is rendered at reduced fidelity [S1, unofficial].

**Data Structures and Layout:**
The method uses two distinct rendering configurations. The experimental parameters are [S1, unofficial]:

| Parameter | Focus Configuration | Context Configuration |
| :--- | :--- | :--- |
| **Volume Size** | $4 \times 256^3$ | $1 \times 128^3$ |
| **Lookup Table Size** | $5 \times 256^2$ | $4 \times 128^2$ |
| **Rendering Model** | Local lighting | Unshaded |
| **Sampling Rate** | 4 | 1 |
| **Viewport Size** | 100% | 50% |

**Steps:**
1.  **Decomposition:** Divide the rendering task into a Focus pass and a Context pass. The Focus region is placed closest to the user [S1, unofficial].
2.  **Context Rendering:** Render the context volume into a smaller viewport (e.g., 50% of the screen dimensions) using an unshaded model and a lower sampling rate [S1, unofficial].
3.  **Filtering:** Apply a Gaussian filter to the resulting context image to reduce aliasing artifacts [S1, unofficial].
4.  **Scaling:** Enlarge the filtered context image to fill its designated portion of the final framebuffer [S1, unofficial].
5.  **Focus Rendering:** Render the focus volume into the remaining viewport area (or full screen if context is transparent) using local lighting and a higher sampling rate [S1, unofficial].
6.  **Compositing:** Composite the focus and context images. If the context is transparent, it serves as an orientation aid; if opaque, it provides an external view [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_focus_and_context(vol_data, focus_region, camera, vp_w, vp_h):
    # Parameters from [S1, unofficial]
    ctx_vp_scale = 0.5
    ctx_sampling = 1
    focus_sampling = 4

    ctx_w = int(vp_w * ctx_vp_scale)
    ctx_h = int(vp_h * ctx_vp_scale)

    # Step 2: Render Context (unshaded, lower sampling)
    context_image = [[0.0] * ctx_w for _ in range(ctx_h)]
    for y in range(ctx_h):
        for x in range(ctx_w):
            acc = 0.0
            for t in range(0, 100, ctx_sampling):
                acc += vol_data.sample_ray(camera, x, y, t)
            context_image[y][x] = acc / 100.0

    # Step 3: Gaussian filter for aliasing reduction
    filtered_context = [[0.0] * ctx_w for _ in range(ctx_h)]
    for y in range(ctx_h):
        for x in range(ctx_w):
            kernel_sum = 0.0
            for dy in range(-1, 2):
                for dx in range(-1, 2):
                    ny, nx = y + dy, x + dx
                    if 0 <= ny < ctx_h and 0 <= nx < ctx_w:
                        kernel_sum += context_image[ny][nx]
            filtered_context[y][x] = kernel_sum / 9.0

    # Step 4: Scale up context image
    scaled_context = [[0.0] * vp_w for _ in range(vp_h)]
    for y in range(vp_h):
        for x in range(vp_w):
            src_y = min(max(int(y / ctx_vp_scale), 0), ctx_h - 1)
            src_x = min(max(int(x / ctx_vp_scale), 0), ctx_w - 1)
            scaled_context[y][x] = filtered_context[src_y][src_x]

    # Step 5: Render Focus (local lighting, higher sampling)
    focus_image = [[0.0] * vp_w for _ in range(vp_h)]
    for y in range(vp_h):
        for x in range(vp_w):
            acc = 0.0
            for t in range(0, 100, focus_sampling):
                acc += vol_data.sample_ray(camera, x, y, t)
            focus_image[y][x] = acc / 100.0

    # Step 6: Composite
    # [G] Source does not specify exact blending equation; standard alpha compositing assumed.
    final_image = [[0.0] * vp_w for _ in range(vp_h)]
    for y in range(vp_h):
        for x in range(vp_w):
            final_image[y][x] = focus_image[y][x] * 0.8 + scaled_context[y][x] * 0.2

    return final_image
```

**Parameters and Edge Cases:**
*   **Context Mode:** Can be transparent (for orientation) or opaque (for external view) [S1, unofficial].
*   **Sampling:** Context sampling can be sparse if the transfer function contains only low-frequency components [S1, unofficial].
*   **Pitfalls:** Overhead comes from rendering slices multiple times and setting up clip textures/depth buffers for each pass [S1, unofficial]. The goal is not seamless integration, so visible seams or quality drops in the context are acceptable trade-offs for speed [S1, unofficial].

## Operating characteristics
*   **Performance:** Achieves 2-6 times faster rendering rates than single high-resolution mode by reducing focus fragments aggressively [S1, unofficial].
*   **Measured Performance:** Tested on a 2.0GHz Pentium 4, 2GB memory, ATI Radeon 9800 Pro [S1, unofficial].
    *   **512x512 Viewport:** 100% Focus Size: 3.39 FPS; 50% Volume Size: 4.22 FPS; 25% Volume Size: 7.55 FPS; 12.5% Volume Size: 12.99 FPS; 6.25% Volume Size: 20.87 FPS [S1, unofficial].
    *   **1024x1024 Viewport:** 100% Focus Size: 1.06 FPS; 50% Volume Size: 1.28 FPS; 25% Volume Size: 2.31 FPS; 12.5% Volume Size: 3.97 FPS; 6.25% Volume Size: 6.33 FPS [S1, unofficial].
*   **Scaling:** Performance gains increase as the focus size (volume percentage) decreases. The speedup depends on rendering parameters, not transfer functions [S1, unofficial].
*   **Strengths:** Simple, generic approach; effective for time-critical applications; reduces clutter in multidimensional exploration [S1, unofficial].
*   **Limitations:** Not seamless; overhead from multiple rendering passes and depth buffer setup; performance variability on ATI cards depending on slicing direction [S1, unofficial].

## Background
This technique was presented in "A Focus and Context Interface for Interactive Volume Rendering" by I. Kits et al. [S1, unofficial]. It was developed to address limited processing capacity in hardware-accelerated volume rendering, where performance depends on fragment count, work per fragment, and cache utilization [S1, unofficial].

## Sources
[S1] A Focus and Context Interface for Interactive Volume Rendering, https://www.sci.utah.edu/~ikits/publications/Ikits_Unpublished_04.pdf, fetched 2026-10-27 (unofficial)
