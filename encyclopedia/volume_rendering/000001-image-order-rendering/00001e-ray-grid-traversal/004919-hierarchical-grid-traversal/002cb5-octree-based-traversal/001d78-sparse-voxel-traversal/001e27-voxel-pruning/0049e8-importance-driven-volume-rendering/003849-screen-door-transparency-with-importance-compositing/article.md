# Screen-Door Transparency with Importance Compositing

Also called: Smart Visibility via Screen-Door Transparency

## Overview
Screen-Door Transparency with Importance Compositing is a rendering technique that resolves occlusion in volumetric data by linking feature importance to spatial sampling density using screen-door transparency [S1, unofficial]. It solves the problem where interesting features are hidden by less important ones by dynamically adjusting sampling density and opacity based on occlusion relationships, rather than using a fixed global transfer function [S1, unofficial]. Unlike Maximum Importance Projection (MImP), which renders only the single most important feature along a ray, this method composites multiple features, suppressing occluding context information while preserving non-occluding context [S1, unofficial].

## How it works
**Inputs:** Volumetric dataset with associated feature importance values; camera position and view direction [S1, unofficial].
**Outputs:** 2D image with transparency effects [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** Stores scalar values and importance values $I$ [S1, unofficial].

**Parameters:**
*   **Grid Spacing:** Inversely proportional to importance [S1, unofficial].

**Steps:**
1.  For each pixel, cast a ray through the volume [S1, unofficial].
2.  Determine the importance $I$ of intersected features [S1, unofficial].
3.  Assign grid spacing $\Delta$ inversely proportional to importance: $\Delta \propto 1/I$ [S1, unofficial]. High importance ($I \approx 1$) uses small grid spacing (dense sampling), while low importance ($I \approx 0$) uses large grid spacing (sparse sampling) [S1, unofficial].
4.  Render focus features with dense sampling [S1, unofficial].
5.  Render context features: if they occlude focus features, use sparse sampling; otherwise, use dense sampling [S1, unofficial].
6.  Composite contributions using importance-based opacity and color: opaque/saturated for high importance, transparent/desaturated for low importance [S1, unofficial]. Occluding context information is suppressed but not entirely removed, and the sum of importance of intersected features along the ray is considered [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_screen_door_transparency(volume, camera):
    height, width = camera.resolution
    image = [[0.0, 0.0, 0.0] for _ in range(height)]
    for y in range(height):
        for x in range(width):
            ray_o = camera.origin
            ray_d = camera.direction_at(x, y)
            t = 0.0
            c_acc = [0.0, 0.0, 0.0]
            a_acc = 0.0
            while t < volume.max_distance:
                voxel = volume.sample_at(ray_o + ray_d * t)
                if voxel is not None:
                    I = voxel.importance
                    step = 1.0 / max(I, 1e-6)
                    occludes_focus = False
                    if not voxel.is_focus:
                        for i in range(1, 5):
                            future_v = volume.sample_at(ray_o + ray_d * (t + i * step))
                            if future_v is not None and future_v.is_focus:
                                occludes_focus = True
                                break
                    if voxel.is_focus or not occludes_focus:
                        pass
                    else:
                        step *= 4.0
                    opacity = I
                    c = voxel.color
                    a = opacity
                    c_acc[0] = c_acc[0] * (1.0 - a) + c[0] * a
                    c_acc[1] = c_acc[1] * (1.0 - a) + c[1] * a
                    c_acc[2] = c_acc[2] * (1.0 - a) + c[2] * a
                    a_acc = a_acc + a * (1.0 - a_acc)
                    if a_acc >= 1.0:
                        break
                t += step
            image[y][x] = c_acc
    return image
```

**Edge Cases:**
*   **Precision:** Sparse sampling may introduce aliasing artifacts if the grid spacing is too large relative to the feature size [G].

## Subtopics

### [Screen-Door Transparency](8930-screen-door-transparency.md)
A voxel pruning technique that simulates optical transparency by spatially subsampling the voxel grid into a uniform, wireframe-like pattern. It solves the problem of rendering dense volumetric data efficiently by reducing visual clutter and computational load, allowing context features to be rendered sparsely while maintaining the structural integrity of focus features. Unlike standard alpha blending which requires sorting and accumulation, this method achieves transparency through geometric sparseness, where the "holes" in the voxel mesh allow background elements to show through, mimicking a semi-transparent view from a distance.

## Operating characteristics
*   **Time Complexity:** Depends on the sampling density [S1, unofficial]. Focus features are rendered densely ($O(N)$), while occluding context features are rendered sparsely ($O(N/k)$ where $k$ is the sparseness factor) [S1, unofficial]. Non-occluding context features are rendered densely [S1, unofficial]. Overall complexity is between dense and sparse volume rendering [S1, unofficial].
*   **Memory Complexity:** Standard volumetric data structures [S1, unofficial]. No additional memory for importance compositing beyond storing importance values per voxel [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Dynamically adjusts sampling density and opacity based on feature importance and occlusion relationships [S1, unofficial]. Reduces visual clutter by suppressing occluding context information [S1, unofficial].
*   **Limitations:** Occluding context is suppressed but not entirely removed, which may still cause some visual clutter [S1, unofficial]. Requires pre-computed or real-time importance values [S1, unofficial].
*   **When to Use:** When the goal is to highlight specific important features in a complex volumetric dataset while providing some context [S1, unofficial].
*   **When Not to Use:** When all features are equally important or when occlusion is not a primary concern [S1, unofficial].

## Background
Originated by Ivan Viola and M. Eduard Gröller in the context of "Smart Visibility" in visualization [S1, unofficial]. The technique is part of a broader effort to apply illustration techniques (cutaway views, ghosted views) to volumetric data visualization [S1, unofficial]. It was presented in the Eurographics conference proceedings [S1, unofficial].

## Sources
[S1] Illustrative Visualization for Medicine and Science, https://doi.org/10.2312/egt.20061068, fetched 2026-10-26 (unofficial)
