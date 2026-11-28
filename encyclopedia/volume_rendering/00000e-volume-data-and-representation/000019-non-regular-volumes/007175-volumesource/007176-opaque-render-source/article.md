# Opaque Render Source

Also called: Annotation Rendering, LineSource, OpaqueSource, OpaqueSource Rendering, PointSource

## Overview
Opaque Render Sources are a class of `RenderSource` objects used to render fully opaque geometric structures—such as lines, points, boxes, grids, and vectors—onto a volume rendering scene [S1, S2]. Unlike `VolumeSource` objects which render semi-transparent volumetric data via transfer functions, Opaque Sources provide high-contrast annotations for locations, particles, regions, or simulation boundaries [S1]. They are integrated into the scene and processed during the ray-tracing pass alongside other sources to produce the final image [S1].

## How it works
The Opaque Render Source framework operates as a base class for specific geometric renderers. It integrates into the standard `yt` volume rendering pipeline by being added to a `Scene` object, which then manages the ray-tracing process [S1, S2].

### Inputs and Outputs
*   **Input**: A `Scene` object containing one or more `RenderSource` instances (including `VolumeSource` and `OpaqueSource` variants) and a `Camera` object defining the view [S1, S2].
*   **Output**: A 2D image (array) representing the composite of all sources, where opaque sources occlude or overlay semi-transparent volume data based on depth [S1].

### Data Structures
The `OpaqueSource` class inherits from `RenderSource` [S2]. Specific implementations like `PointSource` store geometric data in the following structures:

*   **PointSource Data**:
    *   `positions`: `array_like` of shape $(N, 3)$, storing the $(x, y, z)$ coordinates of $N$ points. If no units are specified, values are assumed to be in code units [S2].
    *   `colors`: `array_like` of shape $(N, 4)$, optional. Stores RGBA values in floating point range $[0.0, 1.0]$ [S2].
    *   `radii`: `array_like` of shape $(N)$, optional. Stores the radius of each point in pixels [S2].
    *   `color_stride`: `int`, optional. The stride for accessing color data [S2].

### Algorithm Steps
1.  **Initialization**: Create an instance of a specific Opaque Source (e.g., `PointSource`) with geometric parameters [S2].
2.  **Scene Integration**: Add the Opaque Source object to the `Scene` using `sc.add_source(source)` [S1, S2].
3.  **Z-Buffer Management**: The source may receive a z-buffer via `set_zbuffer(zbuffer)` to handle depth sorting against other sources [S2].
4.  **Rendering**: Call `sc.render()` or `scene.volume_render()` [S1]. This triggers a ray-tracing pass where rays are cast through all `RenderSources` in the scene [S1].
5.  **Composite**: The renderer combines the results. Opaque sources are rendered as fully visible structures at their projected locations, potentially occluding underlying volume data if they are closer to the camera [S1].

### Pseudocode
The following Python-like pseudocode demonstrates the standard usage pattern for adding a `PointSource` to a volume rendering scene.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from yt.visualization.volume_rendering.api import PointSource

# 1. Define point data
# N points in 3D space
npoints = 1000
positions = np.random.random([npoints, 3]) * 1000  # Code units

# Optional: Define colors (RGBA, 0.0 to 1.0)
colors = np.random.random([npoints, 4])
colors[:, 3] = 1.0  # Set alpha to 1.0 for full opacity

# Optional: Define radii in pixels
radii = np.ones(npoints) * 5

# 2. Create the Opaque Source
# Note: In actual yt usage, units (e.g., kpc) are often applied to positions
points_source = PointSource(
    positions=positions,
    colors=colors,
    radii=radii
)

# 3. Add to Scene
# Assuming 'sc' is an existing Scene object
sc.add_source(points_source)

# 4. Render
# This triggers ray tracing through all sources in the scene
image = sc.render()
```

### Edge Cases and Pitfalls
*   **Units**: If `positions` are provided without explicit units, they are interpreted as code units. Mismatched units between the `VolumeSource` (which may use physical units) and `OpaqueSource` can lead to misalignment [S2].
*   **Alpha Channel**: `PointSource` colors include an alpha channel. To ensure the points are truly opaque, the alpha value (index 3) must be set to $1.0$ [S2].
*   **Depth Sorting**: Opaque sources rely on the z-buffer to determine visibility against volume data. If the z-buffer is not correctly passed or initialized, occlusion artifacts may occur [S2].

## Subtopics

### PointSource
A rendering source for opaque points in the scene [S2]. It allows adding $N$ points with specified positions, colors (RGBA), and radii (in pixels) [S2]. It is chosen when the goal is to annotate specific particle locations or discrete data points within the volume [S1]. It inherits from `OpaqueSource` and supports `color_stride` for memory-efficient color array access [S2].

### LineSource
A rendering source for opaque lines [S1]. It is used to draw lines connecting different regions or vertices, such as outlining regions or continents [S1]. It inherits from `OpaqueSource` [S2].

### BoxSource
An annotation source that draws bounding boxes in data space [S1]. It is used to highlight specific regions or simulation boundaries [S1]. It inherits from `OpaqueSource` [S2].

### GridSource
An annotation source that draws grid information in data space [S1]. It is used to visualize the underlying mesh or grid structure of the simulation [S1]. It inherits from `OpaqueSource` [S2].

### CoordinateVectorSource
An annotation source that provides vector orientation within the image [S1]. It is used to display vector fields or directional indicators at specific locations [S1]. It inherits from `OpaqueSource` [S2].

## Operating characteristics
*   **Time Complexity**: Not given in the sources. The rendering cost depends on the number of rays and the number of opaque objects, but the sources do not provide specific complexity formulas or performance benchmarks for Opaque Sources.
*   **Memory Complexity**: Not given in the sources. Memory usage is proportional to the number of points/lines/boxes stored in the source objects.
*   **Strengths**: Provides high-contrast, unambiguous annotations that are not affected by transfer function opacity settings [S1]. Useful for debugging and highlighting specific features like grid cells or particle locations [S1].
*   **Limitations**: Only supports fully opaque structures; cannot represent semi-transparent annotations [S1]. Performance may degrade with a very large number of individual points or lines if not optimized [G].
*   **When to use**: When annotating specific locations, particles, regions, or grid structures in a volume rendering [S1].
*   **When not to use**: When the annotation needs to be semi-transparent or blended with the volume data [S1].

## Background
Opaque Render Sources were introduced as part of the `yt` visualization library's volume rendering framework [S1, S2]. They extend the `RenderSource` base class to support geometric annotations alongside volumetric data [S2]. The framework allows users to combine semi-transparent volume rendering with fully opaque geometric overlays to create annotated scientific visualizations [S1].

## Sources
[S1] 3D Visualization and Volume Rendering, https://yt-project.org/doc/visualizing/volume_rendering.html, fetched 2026-10-28
[S2] RenderSource yt.visualization.volume_rendering.render_source.RenderSource, https://yt-project.org/doc/reference/api/yt.visualization.volume_rendering.render_source.html, fetched 2026-10-29
