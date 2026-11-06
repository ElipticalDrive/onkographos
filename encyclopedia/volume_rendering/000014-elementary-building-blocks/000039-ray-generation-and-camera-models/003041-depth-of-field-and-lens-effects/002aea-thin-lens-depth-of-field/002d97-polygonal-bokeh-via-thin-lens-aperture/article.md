# Polygonal Bokeh via Thin Lens Aperture

Also called: N-Blade Aperture Sampling, Polygonal Bokeh

## Overview
Polygonal Bokeh via Thin Lens Aperture is a depth-of-field technique where the camera aperture is modeled as a regular $N$-sided polygon rather than a circle, causing out-of-focus highlights to render as distinct polygonal shapes [S1, S2, unofficial]. It differs from the parent [Thin Lens Depth of Field](4922-thin-lens-depth-of-field.md) method, which uses concentric disk sampling, by introducing geometric blade constraints that simulate mechanical iris limitations [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray origin, ray direction, number of blades $N$, aperture radius $R$, and rotation angle $\theta$.
*   **Output:** A 2D offset vector $(dx, dy)$ representing the sampled point on the polygonal aperture, which is added to the ray origin to generate the final ray for scattering.

**Data Structures:**
*   **Aperture Polygon:** Defined by $N$ vertices on a circle of radius $R$. The vertices $V_i$ for $i \in [0, N-1]$ are calculated as:
    $$ V_i = (R \cos(\phi_i), R \sin(\phi_i)) $$
    where $\phi_i = \frac{2\pi i}{N} + \theta$.

**Steps:**
1.  **Generate Uniform Polygon Sample:** Generate a point uniformly distributed within the area of the regular $N$-gon. This is typically achieved by decomposing the polygon into $N$ triangles sharing the center, selecting a triangle with probability proportional to its area (equal for regular polygons), and then sampling uniformly within that triangle [S2, unofficial].
2.  **Apply Rotation:** Rotate the sampled point by the aperture orientation angle $\theta$ to simulate the physical rotation of camera blades [S2, unofficial].
3.  **Ray Offset:** Add the resulting 2D offset to the camera's principal ray origin to form the final ray origin.

**Parameters:**
*   **$N$ (Number of Blades):** An integer defining the shape. Common values are 6 (hexagonal) or 8 (octagonal) [S1, S2, unofficial].
*   **$R$ (Aperture Radius):** Controls the size of the blur circle.
*   **$\theta$ (Rotation):** Orientation of the polygon.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sample_polygonal_aperture(N, R, theta):
    """
    Samples a point uniformly from a regular N-gon of radius R, rotated by theta.
    Returns (dx, dy) offset.
    """
    # Step 1: Select a triangle index uniformly from 0 to N-1
    # Since the polygon is regular, all N triangles formed by center and edge i
    # have equal area.
    triangle_idx = random.randint(0, N - 1)
    
    # Vertices of the selected triangle: Center (0,0), V_i, V_{i+1}
    # Angle for vertex i
    phi_i = (2 * math.pi * triangle_idx / N) + theta
    phi_next = (2 * math.pi * (triangle_idx + 1) / N) + theta
    
    # Coordinates of the two outer vertices
    x_i = R * math.cos(phi_i)
    y_i = R * math.sin(phi_i)
    
    x_next = R * math.cos(phi_next)
    y_next = R * math.sin(phi_next)
    
    # Step 2: Sample uniformly within the triangle (0,0), (x_i, y_i), (x_next, y_next)
    # Method: Use two uniform random numbers u, v.
    # If u + v > 1, reflect: u = 1 - u, v = 1 - v.
    # Point = u * V_i + v * V_next
    u = random.random()
    v = random.random()
    
    if u + v > 1.0:
        u = 1.0 - u
        v = 1.0 - v
        
    dx = u * x_i + v * x_next
    dy = u * y_i + v * y_next
    
    return dx, dy
```

**Edge Cases and Pitfalls:**
*   **Precision:** For very large $N$, the polygon approximates a circle, and numerical differences between this method and disk sampling may become negligible [S1, S2, unofficial].
*   **Degenerate Input:** $N < 3$ is invalid for a polygon.

## Subtopics

## Operating characteristics
*   **Complexity:** $O(1)$ time and memory per sample.
*   **Performance:** Identical computational cost to circular aperture sampling, but produces visually distinct results for out-of-focus specular highlights [S1, S2, unofficial].
*   **Strengths:** Simulates realistic camera iris shapes (hexagonal/octagonal) [S1, S2, unofficial].
*   **Limitations:** Requires careful implementation of uniform polygon sampling to avoid bias [S2, unofficial].

## Background
This method is implemented in modern path tracers to enhance photographic realism by mimicking the mechanical aperture blades found in physical cameras [S1, S2, unofficial]. It was notably added to renderers like MagicaVoxel and photon-engine to support advanced imager effects [S1, S2, unofficial].

## Sources
[S1] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S2] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
