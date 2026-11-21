# Two-Pass Surfel Rendering with Enlarged Splat

Also called: Enlarged Splatting, Two-pass visibility/shading

## Overview
Two-Pass Surfel Rendering with Enlarged Splat resolves sharp square-shaped boundary artifacts inherent in standard alpha-blended surfel splatting [S1, unofficial]. It separates visibility determination from shading into two sequential rendering passes, allowing the splat footprint to be enlarged in the second pass to cover surface holes without causing distant surfels to incorrectly occlude closer ones [S1, unofficial]. The technique avoids the performance penalty of view-dependent surfel sorting required for correct alpha blending [S1, unofficial].

## How it works
The method operates on a set of surfels, where each surfel $k$ is defined by its position $\mathbf{p}_k$, orientation $\mathbf{n}_k$, radius $r_k$, and color/appearance properties [G]. The rendering pipeline consists of two sequential passes [S1, unofficial].

**Inputs:**
- A list of $N$ surfels.
- Camera view and projection matrices.
- A depth buffer (initialized to maximum depth or cleared) [G].

**Outputs:**
- A final color image.

**Data Structures:**
- **Surfel List:** An array of structures containing position, normal, radius, and color [G].
- **Depth Buffer:** A 2D array of depth values, size equal to the screen resolution [G].

**Steps:**

1.  **Pass 1: Visibility Splatting**
    -   Render all surfels to determine visibility [S1, unofficial].
    -   Use a standard splatting kernel (e.g., Gaussian or circular) with the original radius $r_k$ [G].
    -   Enable depth testing and depth writing [S1, unofficial].
    -   Do not write to the color buffer (or write a dummy value) [G].
    -   The result is a depth buffer containing the closest surfel depth for each pixel [S1, unofficial].

2.  **Pass 2: Shading with Enlarged Splat**
    -   Render all surfels again [S1, unofficial].
    -   Use an **enlarged radius** $r'_k = \alpha r_k$ where $\alpha > 1$ (typically chosen to cover expected gaps) [G].
    -   Enable depth testing (read-only depth buffer from Pass 1) but **disable depth writing** [S1, unofficial].
    -   Perform full shading calculations (lighting, material properties) [G].
    -   Write the shaded color to the color buffer using alpha blending [G].

**Formulas:**
The splat footprint in screen space is determined by the radius $r$. In Pass 2, the effective radius is:
$$ r' = \alpha \cdot r $$
where $\alpha$ is the enlargement factor [G].

The alpha blending equation for a fragment $f$ with color $C_f$ and alpha $\alpha_f$ over a background color $C_b$ is:
$$ C_{out} = C_f \cdot \alpha_f + C_b \cdot (1 - \alpha_f) $$ [G]

**Parameters:**
-   $\alpha$: Enlargement factor. Not given in the sources. Must be tuned to balance hole coverage against overdraw [G].
-   Kernel shape: Typically circular or Gaussian. Not given in the sources [G].

**Pitfalls and Edge Cases:**
-   **Non-hermetic surfaces:** The method assumes the surface is closed. It is difficult to reconstruct fast-moving objects with this framework because gaps may be interpreted as holes rather than missing data [S1, unofficial].
-   **Overdraw:** Increasing the splat size increases fragment overdraw, which can become a bottleneck at extreme close-ups [S1, unofficial].
-   **Boundary Artifacts:** If the depth buffer is disabled entirely, distant surfels may blend on top of closer ones [S1, unofficial]. Increasing the splat size to cover holes generally reintroduces sharp square-shaped boundary artifacts, which the two-pass approach mitigates by enlarging only in the second pass [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_two_pass_surfel(surfels, camera, screen_width, screen_height):
    depth_buffer = [[1.0] * screen_width for _ in range(screen_height)]
    color_buffer = [[(0.0, 0.0, 0.0, 0.0)] * screen_width for _ in range(screen_height)]
    
    for surfel in surfels:
        screen_pos = project_to_screen(surfel.position, camera)
        rasterize_splat(screen_pos, surfel.radius, depth_buffer, color_buffer, write_depth=True, write_color=False, screen_width=screen_width, screen_height=screen_height)
        
    enlarged_radius_factor = 1.5
    for surfel in surfels:
        screen_pos = project_to_screen(surfel.position, camera)
        radius = surfel.radius * enlarged_radius_factor
        rasterize_splat(screen_pos, radius, depth_buffer, color_buffer, write_depth=False, write_color=True, shading_function=lambda s: compute_shading(s, camera), screen_width=screen_width, screen_height=screen_height)
                        
    return color_buffer

def project_to_screen(position, camera):
    projected = camera.mvp @ position
    return (projected.x / projected.w, projected.y / projected.w)

def compute_shading(surfel, camera):
    return (1.0, 1.0, 1.0, 1.0)

def rasterize_splat(screen_pos, radius, depth_buffer, color_buffer, write_depth, write_color, shading_function=None, screen_width=0, screen_height=0):
    min_x = max(0, int(screen_pos.x - radius))
    max_x = min(screen_width, int(screen_pos.x + radius))
    min_y = max(0, int(screen_pos.y - radius))
    max_y = min(screen_height, int(screen_pos.y + radius))
    
    for y in range(min_y, max_y):
        for x in range(min_x, max_x):
            dist = ((x - screen_pos.x)**2 + (y - screen_pos.y)**2)**0.5
            if dist <= radius:
                alpha = ((radius - dist) / radius)**2
                surfel_depth = camera.z_at_position(surfel.position)
                
                if write_depth:
                    if surfel_depth < depth_buffer[y][x]:
                        depth_buffer[y][x] = surfel_depth
                else:
                    if surfel_depth > depth_buffer[y][x]:
                        continue 
                
                if write_color and shading_function:
                    color = shading_function()
                    current_color = color_buffer[y][x]
                    new_color = tuple(c_f * alpha + c_b * (1 - alpha) for c_f, c_b in zip(color, current_color))
                    color_buffer[y][x] = new_color
```

## Subtopics

### [Surfel Rendering](12351-surfel-rendering.md)
Rendering surfels as oriented, texture-mapped quadrilaterals (billboards) aligned to surface curvature. This is the base technique that Two-Pass Surfel Rendering extends.

## Operating characteristics
-   **Time Complexity:** $O(N \cdot A)$ where $N$ is the number of surfels and $A$ is the average area of a splat in pixels. Two passes double the rasterization cost compared to a single pass [G].
-   **Memory Complexity:** $O(W \cdot H)$ for the depth buffer, where $W$ and $H$ are screen dimensions [G].
-   **Performance:** Avoids the expensive view-dependent sorting required for correct alpha blending [S1, unofficial]. However, it increases fragment overdraw due to the enlarged splats in Pass 2, which can impact pixel fillrate at close distances [S1, unofficial].
-   **Strengths:** Eliminates sharp square-shaped boundary artifacts; smooth appearance; hardware-friendly (no sorting) [S1, unofficial].
-   **Limitations:** Only suitable for hermetic surfaces; not robust for fast-moving objects with large gaps; increased overdraw [S1, unofficial].

## Background
The method addresses the limitations of standard surfel splatting, which suffers from artifacts due to alpha blending and rendering order [S1, unofficial]. The two-pass approach was previously cited in literature [4, 5] (not provided in sources) as a viable alternative to sorting [S1, unofficial]. It is applied to scanned 4D media where surface reconstruction is challenging [S1, unofficial].

## Sources
[S1] Hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4d media, http://133.5.19.150/papers/2010_2013/1-1-36.pdf, fetched 2026-10-31 (unofficial)
