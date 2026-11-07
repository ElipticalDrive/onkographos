# Focus and Context Multiresolution Rendering

Also called: Interactive lenses multiresolution, Time-critical multiresolution rendering

## Overview
Focus and Context Multiresolution Rendering is a technique to balance processing capacity between a high-detail focus region and a lower-detail context region to improve interactivity in time-critical applications [S1, unofficial]. It differs from seamless integration methods by explicitly accepting visual discontinuities to achieve a better tradeoff between achievable quality and speed [S1, unofficial]. It builds on [Resolution Adaptation](3063-resolution-adaptation.md) by decoupling the rendering parameters of the focus and context regions rather than adapting a single global resolution.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, a transfer function, a camera view, and a focus region definition (e.g., a data probe or specific spatial bounds).
*   **Output:** A composite image where the focus region is rendered at high resolution and the context region is rendered at lower resolution and upscaled.

**Data Structures:**
The method utilizes two distinct rendering configurations, typically defined by the following parameters [S1, unofficial]:

| Parameter | Focus Configuration | Context Configuration |
| :--- | :--- | :--- |
| **Volume Resolution** | $4 \times 256^3$ | $1 \times 128^3$ |
| **Lookup Table Size** | $5 \times 256^2$ | $4 \times 128^2$ |
| **Rendering Model** | Local lighting | Unshaded |
| **Sampling Rate** | 4 | 1 |
| **Viewport Size** | 100% | 50% |

*Note: These values are from the experimental evaluation [S1, unofficial]. The volume resolution for context is reduced along one axis (or effectively subsampled) compared to the focus.*

**Steps:**
1.  **Define Focus and Context:** Identify the spatial or logical region of interest (focus) and the remainder of the scene (context). The focus is typically attached to a data probe or placed closest to the user [S1, unofficial].
2.  **Render Context:**
    *   Render the context volume into a smaller viewport (e.g., 50% of the full viewport dimensions) [S1, unofficial].
    *   Use a simpler rendering model (e.g., unshaded) and lower sampling rate (e.g., 1) [S1, unofficial].
    *   If the transfer function contains only low-frequency components, sparse sampling may be used for the context [S1, unofficial].
    *   Apply a Gaussian filter to the resulting context image to reduce aliasing artifacts introduced by the upsampling [S1, unofficial].
3.  **Upscale Context:** Enlarge the filtered context image to match the full viewport dimensions [S1, unofficial].
4.  **Render Focus:** Render the focus volume into the full viewport (or the specific focus region) using high-resolution settings (e.g., 4x sampling, local lighting) [S1, unofficial].
5.  **Composite:** Composite the high-resolution focus image over the upscaled context image. The focus is placed closest to the user, mimicking typical interface usage [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_focus_and_context(volume, transfer_function, camera, focus_region):
    # 1. Setup Context Rendering
    # Context uses lower resolution and simpler model
    context_volume = subsample(volume, factor=2) # Example: 256^3 -> 128^3
    context_lut = create_lut(transfer_function, resolution=128)
    context_viewport = get_half_viewport()
    
    # Render context to smaller framebuffer
    context_image = render_volume(
        volume=context_volume,
        lut=context_lut,
        camera=camera,
        viewport=context_viewport,
        sampling_rate=1,
        shading_model="unshaded"
    )
    
    # 2. Filter Context to reduce aliasing
    context_image = gaussian_filter(context_image, sigma=1.0)
    
    # 3. Upscale Context
    context_image_upscaled = resize(context_image, target_size=full_viewport_size)
    
    # 4. Setup Focus Rendering
    # Focus uses full resolution and complex model
    focus_volume = volume # Or subset if clipping is used
    focus_lut = create_lut(transfer_function, resolution=256)
    focus_viewport = full_viewport_size
    
    # Render focus
    focus_image = render_volume(
        volume=focus_volume,
        lut=focus_lut,
        camera=camera,
        viewport=focus_viewport,
        sampling_rate=4,
        shading_model="local_lighting"
    )
    
    # 5. Composite
    # Focus is composited over context. 
    # If focus is a region, use alpha blending or depth testing.
    # If focus is the entire view but context is background, 
    # the context is rendered first, then focus.
    final_image = composite(focus_image, context_image_upscaled)
    
    return final_image
```

**Edge Cases and Pitfalls:**
*   **Overhead:** Rendering slices multiple times and setting up clip textures/depth buffers for each pass introduces overhead [S1, unofficial].
*   **Visual Discontinuity:** The goal is not seamless integration; sharp transitions between focus and context are expected and acceptable [S1, unofficial].
*   **Hardware Variability:** Performance can vary significantly based on hardware architecture (e.g., ATI Radeon 9800 Pro vs. others) due to differences in texture sampling and slicing performance [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method reduces the number of fragments processed in the fragment pipeline for the context region, leading to 2-6 times faster rendering rates compared to a single high-resolution mode [S1, unofficial].
*   **Memory Complexity:** Requires memory for two sets of textures/LUTs (focus and context), but the context LUT is smaller ($4 \times 128^2$ vs $5 \times 256^2$) [S1, unofficial].
*   **Measured Performance:**
    *   Hardware: 2.0GHz Pentium 4, 2GB memory, ATI Radeon 9800 Pro [S1, unofficial].
    *   512x512 Viewport:
        *   100% Focus Size: 3.39 FPS
        *   50% Volume Size: 4.22 FPS
        *   25% Volume Size: 7.55 FPS
        *   12.5% Volume Size: 12.99 FPS
        *   6.25% Volume Size: 20.87 FPS
    *   1024x1024 Viewport:
        *   100% Focus Size: 1.06 FPS
        *   50% Volume Size: 1.28 FPS
        *   25% Volume Size: 2.31 FPS
        *   12.5% Volume Size: 3.97 FPS
        *   6.25% Volume Size: 6.33 FPS
    [S1, unofficial]
*   **Strengths:** Improves interactivity in time-critical applications; effective for reducing clutter in multidimensional datasets; speedup depends on rendering parameters, not transfer functions [S1, unofficial].
*   **Limitations:** Not suitable for applications requiring seamless visual integration; overhead from multiple rendering passes; performance sensitive to hardware-specific texture slicing efficiency [S1, unofficial].

## Background
The method was presented in "A Focus and Context Interface for Interactive Volume Rendering" by I. Kits et al. [S1, unofficial]. It addresses the limitations of hardware-accelerated volume rendering where performance is constrained by fragment pipeline throughput and cache utilization. The approach was evaluated on a 2.0GHz Pentium 4 with an ATI Radeon 9800 Pro [S1, unofficial].

## Sources
[S1] A Focus and Context Interface for Interactive Volume Rendering, https://www.sci.utah.edu/~ikits/publications/Ikits_Unpublished_04.pdf, fetched 2026-10-27 (unofficial)
