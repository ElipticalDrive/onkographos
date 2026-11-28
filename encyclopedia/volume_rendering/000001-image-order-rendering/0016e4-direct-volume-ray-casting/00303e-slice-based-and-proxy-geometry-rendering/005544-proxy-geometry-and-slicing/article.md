# Proxy Geometry and Slicing


## Overview
Proxy Geometry and Slicing renders volumetric data by decomposing the 3D volume into a set of 2D planar slices, which are treated as explicit geometric primitives (typically triangles or quads) and rasterized by the GPU [G]. This method solves the problem of interactive volume visualization by leveraging fixed-function rasterization and hardware alpha blending, avoiding the per-pixel ray-marching overhead of direct volume ray casting [G]. It sets itself apart from [Direct Volume Ray Casting](52-direct-volume-ray-casting.md) by using explicit geometry and fixed-function blending, and from [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md) by rendering the entire volume interior rather than just isosurfaces [G]. See [Slice-based and Proxy Geometry Rendering](6225-slice-based-and-proxy-geometry-rendering.md) for shared principles.

## How it works
**Inputs**: 3D volume data (stored as a 3D texture on the GPU [S2, unofficial][S3]), camera view direction, volume bounding box, slice spacing/distance.
**Outputs**: Rasterized 2D slices composited into a 2D image [G].
**Data structures**: An array of `BufferGeometry` objects pre-allocated to `MAX_SLICES_COUNT` [S1, unofficial]. Each geometry holds a position attribute array of 21 floats (7 vertices × 3 components) and an index array of 18 integers (6 triangles × 3 indices) [S1, unofficial].

**Steps**:
1. **Bounding Box Setup**: Calculate the minimum and maximum coordinates of the volume bounding box. Store the 8 corners in a `corners` array and create 12 `Line3` edge objects connecting them [S1, unofficial].
2. **Plane Generation**: Create a clipping plane aligned with the camera view direction. Initialize a `plane.constant` value based on the camera position and view distance [S1, unofficial].
3. **Slice Intersection Loop**: While `sliceIndex < MAX_SLICES_COUNT`, decrement `plane.constant` by `window.sliceDistance` [S1, unofficial]. For each of the 12 edges, compute the intersection of the plane with the edge line segment. Collect valid intersection points into `intersectionVertices` [S1, unofficial].
4. **Polygon Sorting & Triangulation**: If `intersectionVertices` contains points, compute the centroid by summing the vertices and dividing by the count [S1, unofficial]. Calculate the 2D angle of each vertex relative to the centroid using `atan2(y - centroid.y, x - centroid.x)` [S1, unofficial]. Sort the vertices by this angle. Push the centroid to the end of the sorted array to form a fan [S1, unofficial].
5. **Geometry Update**: Update the geometry's position attribute with the sorted vertices and centroid [S1, unofficial]. Build the index array for fan triangulation [S1, unofficial]. Set the draw range to the number of vertices [S1, unofficial]. Mark attributes for update and compute the bounding sphere [S1, unofficial]. Increment `sliceIndex` [S1, unofficial].
6. **Texture Mapping**: Each slice vertex is assigned 3D texture coordinates that map to the volumetric data [G].
7. **Rasterization and Shading**: The polygons are rasterized [G]. In the fragment shader, the 3D texture is sampled at the interpolated texture coordinates to retrieve the scalar value [G].
8. **Classification and Blending**: The scalar value is mapped to color and opacity via a transfer function [G]. The fragments are composited using alpha blending (typically `GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA`) [G].
9. **Sorting**: The slices must be rendered in the correct depth order relative to the camera (back-to-front for standard alpha blending) to ensure correct compositing [G].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class BufferGeometry:
    def __init__(self):
        self.position = [0.0] * 21
        self.index = [0] * 18
        self.draw_range = (0, 0)

    def mark_attributes_for_update(self):
        pass

    def compute_bounding_sphere(self):
        pass

class Line3:
    def __init__(self, start, end):
        self.start = start
        self.end = end

class Plane:
    def __init__(self, normal, constant):
        self.normal = normal
        self.constant = constant

    def intersect_line(self, line):
        # Returns intersection point or None
        return None

class ProxyGeometrySlicer:
    def __init__(self, max_slices_count):
        self.max_slices_count = max_slices_count
        self.geometries = [BufferGeometry() for _ in range(max_slices_count)]
        self.corners = []
        self.edges = []
        self.slice_index = 0
        self.intersection_vertices = []

    def set_bounding_box(self, box_min, box_max):
        self.corners = [
            box_min, [box_min[0], box_min[1], box_max[2]], [box_min[0], box_max[1], box_min[2]], [box_min[0], box_max[1], box_max[2]],
            [box_max[0], box_min[1], box_min[2]], [box_max[0], box_min[1], box_max[2]], [box_max[0], box_max[1], box_min[2]], box_max
        ]
        corner_pairs = [(0,1), (0,2), (0,4), (1,3), (1,5), (2,3), (2,6), (3,7), (4,5), (4,6), (5,7), (6,7)]
        self.edges = [Line3(self.corners[i], self.corners[j]) for i, j in corner_pairs]

    def update_proxy_geometries(self, camera_dir, view_plane_constant, slice_distance):
        self.slice_index = 0
        plane = Plane(normal=camera_dir, constant=view_plane_constant)
        while (len(self.intersection_vertices) > 0 or self.slice_index == 0) and self.slice_index < self.max_slices_count:
            plane.constant -= slice_distance
            self.intersection_vertices = []
            for edge in self.edges:
                pt = plane.intersect_line(edge)
                if pt is not None:
                    self.intersection_vertices.append(pt)
            if len(self.intersection_vertices) > 0:
                self.sort_polygon_edges(self.intersection_vertices)
                self.update_geometry(self.slice_index, self.intersection_vertices)
            self.slice_index += 1

    def sort_polygon_edges(self, vertices):
        centroid = [sum(v[i] for v in vertices) / len(vertices) for i in range(3)]
        angles = [math.atan2(v[1] - centroid[1], v[0] - centroid[0]) for v in vertices]
        sorted_vertices = [v for _, v in sorted(zip(angles, vertices))]
        sorted_vertices.append(centroid)
        vertices[:] = sorted_vertices

    def update_geometry(self, index, vertices):
        self.geometries[index].position = [coord for v in vertices for coord in v]
        # Fan triangulation indices: (0,1,2), (0,2,3), ...
        n = len(vertices)
        indices = []
        for i in range(1, n - 1):
            indices.extend([0, i, i + 1])
        self.geometries[index].index = indices + [0] * (18 - len(indices))
        self.geometries[index].draw_range = (0, len(vertices))
        self.geometries[index].mark_attributes_for_update()
        self.geometries[index].compute_bounding_sphere()
```

**Parameters**: `MAX_SLICES_COUNT` (maximum number of slices supported, used to pre-allocate the geometries array) [S1, unofficial]. `window.sliceDistance` (spacing between consecutive planes) [S1, unofficial].
**Edge cases and pitfalls**: If `plane.intersectLine` returns `undefined` (degenerate intersection), the intersection is skipped [S1, unofficial]. The geometries are pre-allocated with fixed sizes (7 vertices, 18 indices), so `intersectionVertices.length` must not exceed 7; otherwise, index bounds errors occur [S1, unofficial].

## Subtopics

### [Texture-Based Volume Rendering](4707-texture-based-volume-rendering.md)
Texture-Based Volume Rendering (TBVR) is a direct volume rendering technique that accelerates image-order ray casting by decomposing the 3D volume into a set of 2D planar slices. These slices are treated as geometric primitives (proxy geometry) that are rasterized by the GPU, with the volumetric data stored in 3D or 2D texture memory. Unlike [View-Aligned Slicing](6530-view-aligned-slicing.md), which dynamically computes slice geometry to maintain constant sampling distance, TBVR typically relies on fixed object-aligned slices or view-aligned slices that may vary in sampling density, trading memory efficiency and artifact susceptibility for rendering speed. Choose TBVR when maximum rendering throughput on standard graphics hardware is required and slight sampling artifacts are acceptable.

### [View-Aligned Slicing](6530-view-aligned-slicing.md)
View-aligned slicing is a direct volume rendering technique that decomposes a 3D volume into a series of 2D planar slices (proxy geometry) aligned with the camera's view direction. Unlike standard ray casting which samples the volume texture directly per pixel, this method generates explicit geometric primitives (triangles or quads) that are rasterized by the GPU. The primary advantage is consistent sampling distance relative to the camera, which minimizes sampling artifacts and maintains uniform slice thickness regardless of viewing angle. Choose this variant when visual fidelity and uniform sampling are prioritized over the performance cost of per-frame geometry regeneration.

### Multi-Slice Raster Rendering
A raster display technique that generates several 2D slices of 3D data and combines them to produce a 3D image [S2, unofficial]. It leaves gaps between the slices, forcing the user to guess about flow field conditions in those gaps [S2, unofficial].

### Polygonization-based Volume Rendering
A volume rendering technique using the Volumizer library to polygonize volume data and render it in layers from back to front [S3]. The volume data is stored in a 3D texture map, and the subregion corresponding to the newly acquired slice is loaded with the new image data immediately after reconstruction [S3]. The volume data is polygonized using trilinear interpolation, then rendered in layers from back to front [S3]. A predefined blend function combines the new layer with the existing display buffer: alpha blending weights each new layer by scalar alpha and the existing buffer by 1-alpha, imparting translucency, while MIP selects the maximum of the new layer and display buffer contents regardless of alpha, used for contrast agent visualization [S3].

## Operating characteristics
Not given in the sources.

## Background
The technique originated from early efforts to visualize 3D data on raster display systems [S2, unofficial]. Smith, Sperry, and Everton extended two-dimensional techniques by generating several two-dimensional slices and combining them to produce a three-dimensional image, marking an early application of this family [S2, unofficial]. This approach leveraged the advancement of raster display systems and computer graphics techniques like ray tracing to create images that exploited higher resolution systems [S2, unofficial]. Later implementations integrated real-time MRI imaging guidance, utilizing the Volumizer library and OpenGL to polygonize and render volume data in layers with alpha blending or MIP for contrast agent visualization [S3].

## Sources
[S1] Pointifix/Vis2_Schott2010, https://github.com/Pointifix/Vis2_Schott2010, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
[S3] Real-Time Volume Rendered MRI for Interventional Guidance, https://pmc.ncbi.nlm.nih.gov/articles/PMC2570028/, fetched 2026-10-27
