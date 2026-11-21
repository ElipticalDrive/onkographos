# Voxel Mesh Generation with Interior Face Culling

Also called: Interior Face Culling, Pixel-to-Mesh Voxelization

## Overview
This method converts a set of 2D pixel coordinates into a 3D triangle mesh representing a voxelized image, where each pixel corresponds to a unit cube [S1, unofficial]. It solves the problem of generating efficient surface geometry from discrete 2D data by emitting only exterior faces and culling interior faces between adjacent same-group pixels [S1, unofficial]. Unlike general surface extraction methods that operate on 3D volumes, this approach operates directly on 2D pixel grids, assuming a fixed depth (Z-axis) extent [S1, unofficial]. It differs from [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by treating the input as a flat set of pixels rather than a 3D block structure, and from [Face Coverage Mask Culling](3397-face-coverage-mask-culling.md) by using simple cardinal neighbor checks rather than projected coverage masks.

## How it works
**Inputs:**
- `pixels`: A list of 2D integer coordinates `List<(int X, int Y)>` representing occupied pixel positions.
- `imageHeight`: An integer representing the height of the source image in pixels, used for the final rotation transformation [S1, unofficial].

**Outputs:**
- `Mesh`: A structure containing:
  - `Vertices`: A `List<Vertex>` where `Vertex` is a record struct with three `float` fields: `X`, `Y`, `Z` [S1, unofficial].
  - `Triangles`: A `List<Triangle>` where `Triangle` is a record struct with three `int` fields: `V1`, `V2`, `V3`, representing indices into `Vertices` [S1, unofficial].

**Parameters:**
- `CubeSize`: The physical size of each voxel cube in world units. The standard value is `1.0f`, representing 1mm per pixel [S1, unofficial].

**Data Structures:**
- `pixelSet`: A `HashSet<(int X, int Y)>` created from the input `pixels` list to enable O(1) neighbor lookup [S1, unofficial].
- `vertexMap`: A `Dictionary<Vertex, int>` mapping unique vertex coordinates to their index in `mesh.Vertices` [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization**: Create an empty `Mesh` object. Initialize `vertexMap` as an empty dictionary. Create `pixelSet` from the input `pixels` list [S1, unofficial].

2.  **Iteration**: For each pixel `(px, py)` in `pixels`:
    a.  **Calculate Bounds**: Compute the world-space coordinates of the voxel cube:
        $$x_0 = px \times CubeSize$$
        $$y_0 = py \times CubeSize$$
        $$x_1 = x_0 + CubeSize$$
        $$y_1 = y_0 + CubeSize$$
        $$z_0 = 0.0f$$
        $$z_1 = CubeSize$$
        [S1, unofficial]

    b.  **Emit Bottom Face**: Emit the face at $Z = z_0$ unconditionally. The vertices are defined in Counter-Clockwise (CCW) order:
        $$v_0 = (x_0, y_1, z_0)$$
        $$v_1 = (x_1, y_1, z_0)$$
        $$v_2 = (x_1, y_0, z_0)$$
        $$v_3 = (x_0, y_0, z_0)$$
        [S1, unofficial]

    c.  **Emit Top Face**: Emit the face at $Z = z_1$ unconditionally. The vertices are defined in CCW order:
        $$v_0 = (x_0, y_0, z_1)$$
        $$v_1 = (x_1, y_0, z_1)$$
        $$v_2 = (x_1, y_1, z_1)$$
        $$v_3 = (x_0, y_1, z_1)$$
        [S1, unofficial]

    d.  **Check Left Neighbor**: Check for a neighbor at $(px-1, py)$. If $(px-1, py)$ is **not** in `pixelSet`, emit the left face at $X = x_0$. Vertices in CCW order:
        $$v_0 = (x_0, y_0, z_1)$$
        $$v_1 = (x_0, y_1, z_1)$$
        $$v_2 = (x_0, y_1, z_0)$$
        $$v_3 = (x_0, y_0, z_0)$$
        [S1, unofficial]

    e.  **Check Right Neighbor**: Check for a neighbor at $(px+1, py)$. If $(px+1, py)$ is **not** in `pixelSet`, emit the right face at $X = x_1$. Vertices in CCW order:
        $$v_0 = (x_1, y_1, z_1)$$
        $$v_1 = (x_1, y_0, z_1)$$
        $$v_2 = (x_1, y_0, z_0)$$
        $$v_3 = (x_1, y_1, z_0)$$
        [S1, unofficial]

    f.  **Check Front Neighbor**: Check for a neighbor at $(px, py-1)$. If $(px, py-1)$ is **not** in `pixelSet`, emit the front face at $Y = y_0$. Vertices in CCW order:
        $$v_0 = (x_1, y_0, z_1)$$
        $$v_1 = (x_0, y_0, z_1)$$
        $$v_2 = (x_0, y_0, z_0)$$
        $$v_3 = (x_1, y_0, z_0)$$
        [S1, unofficial]

    g.  **Check Back Neighbor**: Check for a neighbor at $(px, py+1)$. If $(px, py+1)$ is **not** in `pixelSet`, emit the back face at $Y = y_1$. Vertices in CCW order:
        $$v_0 = (x_0, y_1, z_1)$$
        $$v_1 = (x_1, y_1, z_1)$$
        $$v_2 = (x_1, y_1, z_0)$$
        $$v_3 = (x_0, y_1, z_0)$$
        [S1, unofficial]

3.  **Rotation**: After processing all pixels, apply a 180-degree rotation around the X-axis to orient the mesh "face up". Calculate $y_{max} = imageHeight \times CubeSize$. For each vertex $v$ in `mesh.Vertices`, update it to:
    $$v.X = v.X$$
    $$v.Y = y_{max} - v.Y$$
    $$v.Z = CubeSize - v.Z$$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mesh(pixels: List[Tuple[int, int]], image_height: int, cube_size: float = 1.0f) -> Mesh:
    mesh = Mesh()
    vertex_map = {}
    pixel_set = set(pixels)
    
    def get_or_add_vertex(v: Tuple[float, float, float]) -> int:
        if v in vertex_map:
            return vertex_map[v]
        index = len(mesh.vertices)
        mesh.vertices.append(Vertex(v[0], v[1], v[2]))
        vertex_map[v] = index
        return index

    def emit_face(v0: Tuple[float, float, float], v1: Tuple[float, float, float], 
                  v2: Tuple[float, float, float], v3: Tuple[float, float, float]):
        i0 = get_or_add_vertex(v0)
        i1 = get_or_add_vertex(v1)
        i2 = get_or_add_vertex(v2)
        i3 = get_or_add_vertex(v3)
        mesh.triangles.append(Triangle(i0, i1, i2))
        mesh.triangles.append(Triangle(i0, i2, i3))

    for px, py in pixels:
        x0 = px * cube_size
        y0 = py * cube_size
        x1 = x0 + cube_size
        y1 = y0 + cube_size
        z0 = 0.0f
        z1 = cube_size

        # Bottom face (Z = z0)
        emit_face((x0, y1, z0), (x1, y1, z0), (x1, y0, z0), (x0, y0, z0))
        
        # Top face (Z = z1)
        emit_face((x0, y0, z1), (x1, y0, z1), (x1, y1, z1), (x0, y1, z1))
        
        # Left face (X = x0)
        if (px - 1, py) not in pixel_set:
            emit_face((x0, y0, z1), (x0, y1, z1), (x0, y1, z0), (x0, y0, z0))
            
        # Right face (X = x1)
        if (px + 1, py) not in pixel_set:
            emit_face((x1, y1, z1), (x1, y0, z1), (x1, y0, z0), (x1, y1, z0))
            
        # Front face (Y = y0)
        if (px, py - 1) not in pixel_set:
            emit_face((x1, y0, z1), (x0, y0, z1), (x0, y0, z0), (x1, y0, z0))
            
        # Back face (Y = y1)
        if (px, py + 1) not in pixel_set:
            emit_face((x0, y1, z1), (x1, y1, z1), (x1, y1, z0), (x0, y1, z0))

    # Rotation
    y_max = image_height * cube_size
    for i in range(len(mesh.vertices)):
        v = mesh.vertices[i]
        mesh.vertices[i] = Vertex(v.x, y_max - v.y, cube_size - v.z)
        
    return mesh
```

**Edge Cases and Pitfalls:**
- **Diagonal Adjacency**: Only cardinal neighbors (up, down, left, right) are checked for face culling. Diagonal adjacency does not share a face, so diagonal pixels do not cause face culling [S1, unofficial].
- **Winding Order Preservation**: The rotation transformation (mirroring Y and Z) preserves winding order because two reflections equal a rotation [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$ where $N$ is the number of pixels, as each pixel is processed once with O(1) neighbor lookups.
- **Memory Complexity**: $O(V + T)$ where $V$ is the number of unique vertices and $T$ is the number of triangles. The `vertexMap` and `pixelSet` add $O(N)$ overhead.
- **Performance**: Reduces triangle count significantly by culling interior faces, making it efficient for rendering large voxelized images [S1, unofficial].
- **Strengths**: Simple implementation, fast generation, effective culling for 2D-derived voxel data.
- **Limitations**: Only handles 2D pixel inputs; assumes fixed Z-depth; does not handle transparency or 3D volume data directly [S1, unofficial].
- **When to use**: When converting 2D sprite sheets or pixel art into 3D voxel meshes for rendering.
- **When not to use**: For 3D volumetric data or when diagonal face culling is required.

## Background
This method originates from the `jsonger/voxelizer` project, a simple .NET console application designed to convert pixel coordinates into triangle meshes [S1, unofficial]. It was developed as a practical solution for generating voxelized representations of 2D images, emphasizing efficiency through interior face culling [S1, unofficial].

## Sources
[S1] jsonger/voxelizer, https://github.com/jsonger/voxelizer, fetched 2026-10-26 (unofficial)
