# VolumeSource

Also called: yt VolumeSource

## Overview
`VolumeSource` is a rendering primitive in the `yt` visualization library that decomposes arbitrary volumetric data sources (such as AMR datasets, spheres, or cylinders) into traversable bricks for volume rendering [S1]. It differs from sibling methods like [MFA-DVR](1091-mfa-dvr.md) or [Tetrahedral Mesh Representation](10537-tetrahedral-mesh-representation.md) by operating as a high-level framework that manages the decomposition of `yt` data objects into bricks and relies on the `embree` ray-tracing engine for intersection calculations, rather than implementing its own low-level mesh traversal or basis function evaluation [S1].

## How it works
**Inputs:**
- `data_source`: An `AMR3DData` object or a `Dataset` instance representing the volumetric data [S1].
- `field`: A string specifying the name of the scalar field to render [S1].
- `camera`: A `Camera` instance defining the view parameters [S1].
- `transfer_function`: A mapping from field values to optical properties (color and opacity) [S1].

**Outputs:**
- An `ImageArray` instance containing the rendered image [S1].

**Data Structures:**
The `VolumeSource` maintains an abstract volume object (`self.volume`) which is an `AMRKDTree`. This tree is used to efficiently access data in an efficient manner using a KDTree structure [S1]. The source is decomposed into "bricks" for traversal [S1].

**Parameters:**
- `log_field` (boolean): If `True`, rendering is performed in log space; if `False`, in linear space. Large dynamic range data typically requires `True` [S1].
- `use_ghost_zones` (boolean): If `True`, the `AMRKDTree` estimates vertex-centered data using ghost zones to eliminate seams at grid boundaries. Defaults to `False` for performance [S1].
- `weight_field` (field name): Used for off-axis projections and `ProjectionTransferFunction` renderings [S1].
- `interpolated` (boolean, sampler-specific): Relevant for projections. If `True`, data is interpolated to cell vertices then tri-linearly interpolated to ray positions. If `False`, cell-centered data is accumulated. Interpolation is always performed for volume renderings [S1].

**Algorithm:**
1.  **Initialization**: Create the `VolumeSource` with the `data_source` and `field`. This automatically creates the underlying `AMRKDTree` volume object [S1].
2.  **Configuration**:
    -   Set the transfer function using `set_transfer_function`.
    -   Configure sampling behavior using `set_sampler`. For volume rendering, interpolation is mandatory. For projections, choose `interpolated=True` for tri-linear interpolation or `interpolated=False` for cell-centered accumulation [S1].
    -   Optionally enable `use_ghost_zones` to handle boundary seams [S1].
    -   Optionally set `log_field` for dynamic range handling [S1].
3.  **Rendering**: Call `render(camera)`. The source decomposes the data into bricks. The `embree` engine (via `pyembree`) is used to ray-trace through these bricks [S1].
4.  **Z-Buffering**: If multiple sources are rendered manually, a `zbuffer` can be passed to `render` to determine depth ordering for opaque sources [S1].
5.  **Finalization**: The resulting image is returned as an `ImageArray` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import yt
from yt.visualization.volume_rendering.api import Camera, Scene, create_volume_source

# 1. Load data and create source
ds = yt.load("path/to/dataset")
source = create_volume_source(ds.all_data(), "density")

# 2. Configure parameters
source.set_log(True)  # Use log space for large dynamic range
source.set_use_ghost_zones(True)  # Eliminate seams at grid boundaries

# 3. Setup scene and camera
scene = Scene()
scene.add_source(source)
camera = Camera(scene, render_source=source)

# 4. Render
image = scene.render()
```

**Edge Cases and Pitfalls:**
-   **Seams**: Without `use_ghost_zones=True`, seams may appear at grid boundaries in AMR data [S1].
-   **Dynamic Range**: Linear rendering (`log_field=False`) may fail to visualize features in data with large dynamic ranges [S1].
-   **Dependencies**: Requires `embree` and `pyembree` to be installed [S1].

## Subtopics

### [Opaque Render Source](22285-opaque-render-source.md)
Renders opaque geometric structures (lines, points, boxes) for annotations. Chosen when high-contrast overlays are needed rather than semi-transparent volumetric data.

## Operating characteristics
-   **Performance**: Relies on the `embree` ray-tracing engine for intersection calculations [S1].
-   **Memory**: Decomposes data into bricks for traversal [S1].
-   **Strengths**: High-level abstraction for `yt` data objects; handles AMR decomposition automatically [S1].
-   **Limitations**: Requires specific dependencies (`embree`); limited to `yt` data structures [S1].

## Background
`VolumeSource` is part of the `yt` visualization library's volume rendering API [S1]. It provides the framework to decompose arbitrary `yt` data sources into bricks that can be traversed and volume rendered [S1].

## Sources
[S1] RenderSource yt.visualization.volume_rendering.render_source.RenderSource, https://yt-project.org/doc/reference/api/yt.visualization.volume_rendering.render_source.html, fetched 2026-10-29
