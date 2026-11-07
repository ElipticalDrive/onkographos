# Orthographic (Parallel) Camera Ray Generation

Also called: Parallel Ray Generation

## Overview
Orthographic (Parallel) Camera Ray Generation constructs ray origins distributed on a grid perpendicular to the viewing direction and assigns identical normalized directions to all rays for an orthographic camera [S1, unofficial]. It avoids perspective distortion by maintaining parallel ray directions and linear spacing of origins across the image plane [S1, unofficial]. Orthographic projection ensures parallel lines in the 3D scene remain parallel on the screen [S3, unofficial].

## How it works
**Inputs:**
- `eye`: Camera position in world space (3D vector) [G].
- `target`: Look-at point in world space (3D vector) [G].
- `up`: Up vector in world space (3D vector) [G].
- `width`, `height`: Image dimensions in pixels (integers) [G].
- `extent`: Half-size of the viewing region in world units along both right and up axes (float, default 1.0) [S1, unofficial].

**Outputs:**
- `origins`: Array of ray origins with shape `(height, width, 3)` [G].
- `directions`: Array of ray directions with shape `(height, width, 3)` [G].

**Data Structures:**
- Vectors are 3D floating-point arrays [G].
- `origins` and `directions` are dense arrays matching the image resolution [G].

**Steps:**
1. Compute the camera basis vectors (forward, right, cam_up) using the eye, target, and up vectors [S1, unofficial].
2. Compute image plane dimensions based on aspect ratio. Let $\text{aspect} = \text{width} / \text{height}$ [S1, unofficial]. Then $\text{half\_w} = \text{extent} \times \text{aspect}$ and $\text{half\_h} = \text{extent}$ [S1, unofficial].
3. Generate pixel coordinates. Create a meshgrid of column indices $j$ (from $0$ to $\text{width}-1$) and row indices $i$ (from $0$ to $\text{height}-1$) [S1, unofficial]. Compute normalized coordinates $u$ and $v$:
   $$ u = \left( \frac{2.0 \cdot (j + 0.5)}{\text{width}} - 1.0 \right) \cdot \text{half\_w} $$
   $$ v = \left( 1.0 - \frac{2.0 \cdot (i + 0.5)}{\text{height}} \right) \cdot \text{half\_h} $$
   This maps pixel centers to a centered coordinate system where the center is $(0,0)$ [S1, unofficial].
4. Compute origins. For each pixel $(i, j)$, the origin is:
   $$ \text{origin}_{i,j} = \text{eye} + u_{i,j} \cdot \text{right} + v_{i,j} \cdot \text{cam\_up} $$
   This is implemented by indexing `eye` with `[None, :]` and `u`, `v` with `[:, :, None]` to align dimensions for vector addition [S1, unofficial].
5. Set directions. The direction for all rays is the forward vector $\text{forward}$, broadcast to match the shape of `origins` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def orthographic_ray_generation(eye, target, up, width, height, extent=1.0):
    # Step 1: Compute camera basis
    forward = target - eye
    forward = forward / np.linalg.norm(forward)
    right = np.cross(forward, up)
    right = right / np.linalg.norm(right)
    cam_up = np.cross(right, forward)
    
    # Step 2: Compute image plane dimensions
    aspect = width / height
    half_w = extent * aspect
    half_h = extent
    
    # Step 3: Generate pixel coordinates
    j, i = np.meshgrid(np.arange(width), np.arange(height))
    
    # Compute u and v coordinates
    u = (2.0 * (j + 0.5) / width - 1.0) * half_w
    v = (1.0 - 2.0 * (i + 0.5) / height) * half_h
    
    # Step 4: Compute origins
    # Align dimensions for vector addition as per source indexing
    origins = eye[None, :] + u[:, :, None] * right[None, :] + v[:, :, None] * cam_up[None, :]
    
    # Step 5: Set directions
    directions = np.full((height, width, 3), forward)
    
    return origins, directions
```

**Parameters:**
- `extent`: Half-size of the viewing region in world units along both right and up axes. Default value is 1.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `up` is parallel to `forward`, the cross product for `right` will be zero, causing division by zero during normalization. A guard should ensure `up` is not parallel to the viewing direction [G].
- Precision issues may arise if `extent` is very large or very small relative to the scene scale, potentially affecting ray intersection accuracy [G].

## Subtopics

### [Viewing Plane Ray Generation](2424-viewing-plane-ray-generation.md)
Viewing Plane Ray Generation constructs the initial ray origins and directions for an orthographic camera by defining a 2D grid of points in normalized screen space and transforming them into world space. It builds on [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md).

### Orthographic Camera Handling for Point Clouds
In the context of differentiable volume rendering for point clouds (e.g., VoGE), the camera type is alternated from Orthographic to PerspectiveCamera with a similar viewing scope to enable rendering and improve boundary smoothness [S2, unofficial].

### Orthographic Projection
Orthographic projection is a size-preserving projection where objects in the foreground take up the same amount of screen space as objects in the background, and parallel lines in the 3D scene remain parallel when drawn on the screen [S3, unofficial]. It is defined by six clipping planes bounding the box in the orthographic projection: `fleft`, `fright`, `fdown`, `fup`, `fnear`, and `ffar` [S3, unofficial]. In OpenGL, this is implemented via `glOrtho` [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(W \cdot H)$ to generate origins and directions, where $W$ and $H$ are image width and height [G].
- **Memory Complexity:** $O(W \cdot H)$ to store the origins and directions arrays [G].
- **Strengths:** No perspective distortion; parallel lines remain parallel; uniform sampling density across the image plane regardless of depth [G].
- **Limitations:** No depth cueing via size reduction; not suitable for scenes requiring realistic depth perception [G].
- **When to use:** Technical visualization, CAD, scientific data visualization where accurate spatial relationships are critical [G].
- **When not to use:** Realistic rendering requiring depth cues, scenes with large depth ranges where perspective is needed for clarity [G].

## Background
Orthographic projection is a fundamental concept in computer graphics, used to flatten the 3D scene onto a two-dimensional plane without perspective distortion [S3, unofficial]. It is commonly used in technical drawing and visualization tools such as Smokeview for fire dynamics simulation data [S3, unofficial]. The method is implemented in OpenGL via `glOrtho` [S3, unofficial].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S2] VoGE: A Differentiable Volume Renderer using Gaussian Ellipsoids for Analysis-by-Synthesis, https://doi.org/10.48550/arxiv.2205.15401, fetched 2026-10-26 (unofficial)
[S3] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
