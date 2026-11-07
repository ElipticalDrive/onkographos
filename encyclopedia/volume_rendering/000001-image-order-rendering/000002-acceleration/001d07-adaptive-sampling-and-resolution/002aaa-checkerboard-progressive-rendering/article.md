# Checkerboard Progressive Rendering

Also called: Checkerboard sampling

## Overview
Checkerboard Progressive Rendering reduces per-frame compute by tracing only half the pixels each frame while blending results over time [S1, unofficial]. It alternates which pixels are freshly traced, allowing the image to converge progressively without full recomputation [S1, unofficial]. This method distinguishes itself from full-screen adaptive techniques by using a fixed, deterministic spatial pattern (checkerboard) rather than error-driven or random sampling masks [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Previous frame’s color buffer $C_{prev}$, current frame’s partial render buffer $C_{curr}$, and a frame counter $f$.
- Output: Blended color buffer $C_{out}$ for display.

**Data Structures:**
- $C_{prev}$: A 2D array of color values (e.g., `float3` or `vec3`) with dimensions $W \times H$.
- $C_{curr}$: A 2D array of color values, initialized to zero or transparent for each frame.
- $f$: Integer frame index, starting at 0.

**Steps:**
1. Determine the active pixel set for the current frame $f$. A pixel at screen coordinates $(x, y)$ is active if $(x + y + f) \pmod 2 == 0$. This creates a checkerboard pattern that shifts every frame [S1, unofficial].
2. For each active pixel $(x, y)$:
   a. Trace the ray through the volume using the standard volume rendering integral [S1, unofficial].
   b. Compute the color contribution $C_{trace}(x, y)$ using the shading function (e.g., `shadeOpaqueSphere` or `shadeVoxel` depending on material transparency) [S1, unofficial].
   c. Store $C_{trace}(x, y)$ in $C_{curr}(x, y)$.
3. For each inactive pixel $(x, y)$:
   a. Copy the previous frame’s result to the current buffer: $C_{curr}(x, y) = C_{prev}(x, y)$ [S1, unofficial].
4. Blend the buffers to produce the output. A simple average or weighted blend is used. If $B$ is the blend factor (e.g., 0.5 for equal weighting), then:
   $$ C_{out}(x, y) = B \cdot C_{curr}(x, y) + (1 - B) \cdot C_{prev}(x, y) $$
   [S1, unofficial]. Note: In some implementations, the "carry forward" step (Step 3) effectively acts as the blend if $C_{curr}$ is not cleared, but explicit blending is safer for convergence [S1, unofficial].
5. Update $C_{prev} = C_{out}$ for the next frame.
6. Increment $f$.

**Parameters:**
- Blend factor $B$: Typically 0.5, but can be tuned for convergence speed vs. latency [S1, unofficial].
- Frame counter $f$: Resets periodically or wraps around to prevent overflow [G].

**Edge Cases and Pitfalls:**
- **Precision:** Accumulating errors over many frames can lead to bias if the blend factor is not handled correctly. Russian roulette termination may be needed for long paths to remove bias [S2, unofficial].
- **Boundaries:** Ensure the checkerboard pattern covers all pixels. The condition $(x + y + f) \pmod 2 == 0$ ensures exactly half the pixels are active.
- **Degenerate Input:** If the scene is static, the method converges to the full solution. If the scene changes rapidly, the blending may cause ghosting.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def checkerboard_progressive_render(frame_buffer, width, height, trace_ray, blend_factor=0.5):
    # Initialize previous frame buffer if not present
    if not hasattr(checkerboard_progressive_render, 'prev_buffer'):
        checkerboard_progressive_render.prev_buffer = np.zeros((height, width, 3))
        checkerboard_progressive_render.frame_count = 0
    
    curr_buffer = np.zeros((height, width, 3))
    f = checkerboard_progressive_render.frame_count
    
    for y in range(height):
        for x in range(width):
            # Determine if pixel is active in this frame
            if (x + y + f) % 2 == 0:
                # Trace ray for active pixel
                color = trace_ray(x, y)
                curr_buffer[y, x] = color
            else:
                # Carry forward previous frame's result
                curr_buffer[y, x] = checkerboard_progressive_render.prev_buffer[y, x]
    
    # Blend current and previous buffers
    out_buffer = blend_factor * curr_buffer + (1 - blend_factor) * checkerboard_progressive_render.prev_buffer
    
    # Update previous buffer for next frame
    checkerboard_progressive_render.prev_buffer = out_buffer
    checkerboard_progressive_render.frame_count += 1
    
    return out_buffer
```

## Subtopics

### Progressive Ray Tracing
A technique for producing lower quality images which are then improved with time [S2, unofficial]. It is a broader category that includes checkerboard sampling. The main drawback is that high-frequency details tend to get lost [S2, unofficial]. Russian roulette termination helps remove bias at the cost of adding noise [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(W \cdot H / 2)$ per frame, as only half the pixels are traced [S1, unofficial].
- **Memory Complexity:** $O(W \cdot H)$ for storing the previous frame buffer [S1, unofficial].
- **Measured Performance:** Reduces per-frame compute by 50% [S1, unofficial].
- **Scaling:** Scales linearly with resolution. Convergence speed depends on the blend factor and scene dynamics.
- **Strengths:** Low latency, reduced compute, simple implementation.
- **Limitations:** High-frequency details may be lost; ghosting in dynamic scenes; bias if not handled correctly [S2, unofficial].
- **When to Use:** Static or slowly changing scenes where reduced compute is critical.
- **When Not to Use:** Rapidly changing scenes or when high-frequency detail is essential.

## Background
Originated in the context of progressive rendering techniques for ray tracing [S2, unofficial]. It is a specific instance of progressive ray tracing [S2, unofficial]. The method is used in voxel ray tracers and other volume rendering applications where compute resources are limited [S1, unofficial].

## Sources
[S1] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
