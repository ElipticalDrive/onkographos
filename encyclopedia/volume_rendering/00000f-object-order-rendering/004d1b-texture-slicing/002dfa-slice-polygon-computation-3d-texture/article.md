# Slice Polygon Computation (3D Texture)
## Overview
Slice Polygon Computation (3D Texture) is a texture-based volume rendering method that generates proxy geometry by intersecting slicing planes with the volume's bounding box, then mapping the resulting polygons using 3D texture coordinates [S1]. It differs from the alternative approach of storing data as 2D texture slices, which uses pre-aligned rectangular proxy polygons, by dynamically computing arbitrary slice polygons and relying on 3D texture lookups [S1].

## How it works
**Inputs:**
- A volumetric dataset stored as a 3D texture object [S1].
- A viewing direction defining a series of parallel slicing planes [S1].
- The bounding box of the volume [S1].

**Outputs:**
- A sequence of textured polygons (proxy geometry) representing the intersection of the slicing planes with the volume [S1].

**Data Structures:**
- Vertex arrays containing 3D positions and corresponding 3D texture coordinates [S1].
- The 3D texture object containing the scalar field data [S1].

**Steps:**
1. **Intersect Slicing Planes with Bounding Box:** Calculate the intersection points between the slicing planes and the edges of the volume's bounding box [S1].
2. **Sort Vertices:** Sort the resulting intersection vertices in a clockwise or counterclockwise direction around their center to form a valid polygon [S1].
3. **Calculate Texture Coordinates:** For each vertex of the generated polygon, calculate the corresponding 3D texture coordinate [S1]. This can be done on the CPU, in a vertex program, or via automatic texture-coordinate generation [S1].
4. **Render:** Pass the vertex arrays to the graphics library to render the textured polygons [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vector3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __sub__(self, o): return Vector3(self.x-o.x, self.y-o.y, self.z-o.z)
    def __add__(self, o): return Vector3(self.x+o.x, self.y+o.y, self.z+o.z)
    def __mul__(self, s): return Vector3(self.x*s, self.y*s, self.z*s)
    def dot(self, o): return self.x*o.x + self.y*o.y + self.z*o.z
    def cross(self, o): return Vector3(self.y*o.z-self.z*o.y, self.z*o.x-self.x*o.z, self.x*o.y-self.y*o.x)
    def length(self): return (self.x**2 + self.y**2 + self.z**2)**0.5
    def normalize(self): l = self.length(); return Vector3(self.x/l, self.y/l, self.z/l) if l > 1e-9 else Vector3(0,0,0)

def intersect_line_plane(p1, p2, plane_p0, plane_normal):
    line_vec = p2 - p1
    denom = line_vec.dot(plane_normal)
    if abs(denom) < 1e-9: return None
    t = (plane_p0 - p1).dot(plane_normal) / denom
    if 0 <= t <= 1: return p1 + line_vec * t
    return None

def compute_center(points):
    if not points: return Vector3(0, 0, 0)
    s = Vector3(0, 0, 0)
    for p in points: s = s + p
    return s * (1.0 / len(points))

def sort_clockwise_around_center(points, center, normal):
    if len(points) <= 2: return points
    # Create orthonormal basis for the plane
    ref = Vector3(0, 1, 0) if abs(normal.y) < 0.9 else Vector3(1, 0, 0)
    u = normal.cross(ref).normalize()
    v = u.cross(normal)
    projected = []
    for p in points:
        vec = p - center
        projected.append((vec.dot(u), vec.dot(v), p))
    # Sort by angle using atan2 (pseudo-angle can be used to avoid trig [G])
    projected.sort(key=lambda pt: math.atan2(pt[1], pt[0]))
    return [pt[2] for pt in projected]

def map_position_to_texture_coord(position, bbox_min, bbox_max):
    dx = bbox_max.x - bbox_min.x
    dy = bbox_max.y - bbox_min.y
    dz = bbox_max.z - bbox_min.z
    if dx == 0 or dy == 0 or dz == 0: return (0.0, 0.0, 0.0) # Guard against degenerate bounding box [G]
    return ((position.x - bbox_min.x) / dx, (position.y - bbox_min.y) / dy, (position.z - bbox_min.z) / dz)

def compute_slice_polygons(volume_bbox_min, volume_bbox_max, slicing_planes):
    polygons = []
    # Source does not specify exact edge enumeration; standard 12 bounding box edges used [G]
    edges = [
        (volume_bbox_min, Vector3(volume_bbox_max.x, volume_bbox_min.y, volume_bbox_min.z)),
        (Vector3(volume_bbox_max.x, volume_bbox_min.y, volume_bbox_min.z), volume_bbox_max),
        (volume_bbox_min, Vector3(volume_bbox_min.x, volume_bbox_max.y, volume_bbox_min.z)),
        (Vector3(volume_bbox_min.x, volume_bbox_max.y, volume_bbox_min.z), volume_bbox_max),
        (volume_bbox_min, Vector3(volume_bbox_min.x, volume_bbox_min.y, volume_bbox_max.z)),
        (Vector3(volume_bbox_min.x, volume_bbox_min.y, volume_bbox_max.z), volume_bbox_max),
        (Vector3(volume_bbox_max.x, volume_bbox_min.y, volume_bbox_min.z), Vector3(volume_bbox_max.x, volume_bbox_max.y, volume_bbox_min.z)),
        (Vector3(volume_bbox_min.x, volume_bbox_max.y, volume_bbox_min.z), Vector3(volume_bbox_min.x, volume_bbox_max.y, volume_bbox_max.z)),
        (Vector3(volume_bbox_min.x, volume_bbox_min.y, volume_bbox_max.z), Vector3(volume_bbox_max.x, volume_bbox_min.y, volume_bbox_max.z)),
        (Vector3(volume_bbox_max.x, volume_bbox_min.y, volume_bbox_max.z), Vector3(volume_bbox_max.x, volume_bbox_max.y, volume_bbox_max.z)),
        (Vector3(volume_bbox_min.x, volume_bbox_max.y, volume_bbox_max.z), Vector3(volume_bbox_max.x, volume_bbox_max.y, volume_bbox_max.z)),
        (Vector3(volume_bbox_max.x, volume_bbox_max.y, volume_bbox_min.z), Vector3(volume_bbox_max.x, volume_bbox_max.y, volume_bbox_max.z))
    ]
    for plane in slicing_planes:
        intersection_points = []
        for p1, p2 in edges:
            pt = intersect_line_plane(p1, p2, plane.p0, plane.normal)
            if pt is not None: intersection_points.append(pt)
        center = compute_center(intersection_points)
        sorted_pts = sort_clockwise_around_center(intersection_points, center, plane.normal)
        vertices = []
        for pt in sorted_pts:
            tex = map_position_to_texture_coord(pt, volume_bbox_min, volume_bbox_max)
            vertices.append((pt, tex))
        polygons.append(vertices)
    return polygons
```

**Parameters:**
- Slicing plane spacing: Determines the number of slices [G].
- Bounding box: Defines the spatial extent of the volume [S1].

**Edge Cases and Pitfalls:**
- Precision issues in clipping: The brute-force approach of generating a large triangle and clipping it to the data set can suffer from precision issues when generating texture coordinates for clipped triangles [S2]. The slice polygon computation method avoids this by intersecting directly with the bounding box edges [S1].
- Degenerate input: If a slicing plane does not intersect the bounding box, no vertices are generated, and the slice is skipped [G].

## Subtopics

### Graph-Based Oblique Slice Generation
A method to efficiently generate a stack of oblique slices orthogonal to the viewing direction by treating each slice as a 2D 'scanplane' and exploiting scanplane coherence, avoiding polygon clipping [S2]. It differs from the brute-force approach (calculate plane equation, generate large triangle, clip to data set, generate texture coordinates) by generating slices directly with no clipping, avoiding precision problems and artifacts from numeric imprecision in clipping [S2]. It is chosen when oblique slices are required and efficiency is a concern [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Avoids precision issues associated with polygon clipping [S2].
- **Limitations:** Not given in the sources.
- **When to use:** When using 3D textures and dynamic slicing planes are required [S1].
- **When not to use:** Not given in the sources.

## Background
The method is described in GPU Gems Chapter 39, "Volume rendering techniques" [S1]. It is a standard technique in texture-based volume rendering, building on the principles of texture slicing [S1].

## Sources
[S1] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S2] Interactive Volume Rendering Using 3D Texture-Mapping ..., https://web.cs.ucdavis.edu/~okreylos/PhDStudies/Winter2000/TextureMapping.html, fetched 2026-10-27
