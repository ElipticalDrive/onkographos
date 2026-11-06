# Disjoint Set Visible Plane Optimization

Also called: DisjointSetVisiblePlaneOptimizer

## Overview
Disjoint Set Visible Plane Optimization is a greedy meshing algorithm that optimizes a 2D slice (VisiblePlane) of solid voxels by merging contiguous regions of identical voxel IDs into single rectangular mesh quads using a Disjoint Set Union (DSU) data structure [S1, unofficial]. It differs from its parent, Greedy Voxel Mesh Optimization, by operating on pre-culled 2D planes rather than the full 3D volume, and from Disjoint Set Mesh Optimization by focusing specifically on the plane-by-plane merging phase after occlusion culling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A `VisiblePlane` object containing a 2D array of nullable `Voxel` references (`Voxel?[,]`), dimensions `width` and `height`, and axis configuration (`MajorAxis`, `MiddleAxis`, `MinorAxis`, `SliceIndex`).
*   **Output:** A list of `MeshQuad` objects, each representing a merged rectangular face with vertices, normal, and voxel ID.

**Data Structures:**
*   `DisjointSet`: A DSU structure initialized with size `width * height`. Each element represents a voxel coordinate in the plane.
*   `Voxel`: An object containing at least an `ID` field (integer/ushort).
*   `MeshQuad`: Contains `Vertex0..3` (Vector3), `Normal` (Vector3), and `VoxelId` (int).
*   `VisiblePlane`: Contains `Voxels` (2D array), `MajorAxis`, `MajorAxisOrder`, `MiddleAxis`, `MiddleAxisOrder`, `MinorAxis`, `MinorAxisOrder`, and `SliceIndex`.

**Index Mapping:**
The 2D coordinate $(x, y)$ is mapped to a 1D index for the DSU:
$$ \text{ToIndex}(x, y) = y \times \text{width} + x $$

**Algorithm Steps:**

1.  **Initialize DSU:** Create a `DisjointSet` with size `width * height`.
2.  **Iterate and Merge:** Loop through each voxel $(x, y)$ in the plane:
    *   If `voxels[x, y]` is null, skip.
    *   If `IsNotAlone(x, y)` returns true, skip.
    *   Otherwise, call `CreateOneSet(x, y)`.

    **`CreateOneSet(x, y)`:**
    1.  Identify `rootVoxel` at `(x, y)`.
    2.  **Expand Right:** Initialize `currentWidth = 1`. While `x + currentWidth < width`:
        *   Check if `voxels[x + currentWidth, y]` is not null.
        *   Check if `voxels[x + currentWidth, y].ID == rootVoxel.ID`.
        *   Check if `IsNotAlone(x + currentWidth, y)` is false.
        *   If all true, increment `currentWidth`. Else, break.
    3.  **Expand Down:** Initialize `currentHeight = 1`. While `y + currentHeight < height`:
        *   Set `canExpand = true`.
        *   For each `dx` from `0` to `currentWidth - 1`:
            *   Let `cx = x + dx`, `cy = y + currentHeight`.
            *   If `voxels[cx, cy]` is null, set `canExpand = false` and break.
            *   If `voxels[cx, cy].ID != rootVoxel.ID`, set `canExpand = false` and break.
            *   If `IsNotAlone(cx, cy)` is true, set `canExpand = false` and break.
        *   If `canExpand` is true, increment `currentHeight`. Else, break.
    4.  **Union:** For all `dx` from `0` to `currentWidth - 1` and `dy` from `0` to `currentHeight - 1`:
        *   Call `disjointSet.Union(ToIndex(x, y), ToIndex(x + dx, y + dy))`.

    **`IsNotAlone(x, y)`:**
    Returns true if:
    *   `voxels[x, y]` is null.
    *   `disjointSet.Find(ToIndex(x, y))` is not the root of its set (i.e., already merged into a larger block).
    *   Any of the 4 orthogonal neighbors `(x-1,y), (x+1,y), (x,y-1), (x,y+1)` exist, are solid (non-null), have the same ID as `voxels[x, y]`, and are in the same disjoint set (checked via `AreSame`).

    **`AreSame(x1, y1, x2, y2)`:**
    Returns true if:
    *   `voxels[x1, y1]` and `voxels[x2, y2]` are non-null.
    *   `voxels[x1, y1].ID == voxels[x2, y2].ID`.
    *   `disjointSet.Find(ToIndex(x1, y1)) == disjointSet.Find(ToIndex(x2, y2))`.

3.  **Generate Mesh Quads:**
    *   Group all voxel coordinates by their DSU root ID.
    *   For each group:
        *   Find `minX`, `maxX`, `minY`, `maxY` of the coordinates in the group.
        *   Determine `voxelId` from `voxels[minX, minY]`.
        *   Create a single `MeshQuad` using the plane's axis configuration and the bounding box `(minX, maxX, minY, maxY)`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class DisjointSet:
    def __init__(self, size):
        self.parent = list(range(size))
        self.rank = [0] * size

    def find(self, i):
        if self.parent[i] != i:
            self.parent[i] = self.find(self.parent[i])
        return self.parent[i]

    def union(self, i, j):
        root_i = self.find(i)
        root_j = self.find(j)
        if root_i != root_j:
            if self.rank[root_i] < self.rank[root_j]:
                self.parent[root_i] = root_j
            elif self.rank[root_i] > self.rank[root_j]:
                self.parent[root_j] = root_i
            else:
                self.parent[root_j] = root_i
                self.rank[root_i] += 1

def disjoint_set_visible_plane_optimization(plane):
    width = plane.width
    height = plane.height
    voxels = plane.voxels
    dsu = DisjointSet(width * height)

    def to_index(x, y):
        return y * width + x

    def are_same(x1, y1, x2, y2):
        v1 = voxels[x1, y1]
        v2 = voxels[x2, y2]
        if v1 is None or v2 is None:
            return False
        if v1.id != v2.id:
            return False
        return dsu.find(to_index(x1, y1)) == dsu.find(to_index(x2, y2))

    def is_not_alone(x, y):
        v = voxels[x, y]
        if v is None:
            return True
        if dsu.find(to_index(x, y)) != to_index(x, y):
            return True
        neighbors = [(x-1, y), (x+1, y), (x, y-1), (x, y+1)]
        for nx, ny in neighbors:
            if 0 <= nx < width and 0 <= ny < height:
                if are_same(x, y, nx, ny):
                    return True
        return False

    def create_one_set(x, y):
        root_voxel = voxels[x, y]
        current_width = 1
        while x + current_width < width:
            next_v = voxels[x + current_width, y]
            if next_v is None or next_v.id != root_voxel.id or is_not_alone(x + current_width, y):
                break
            current_width += 1
        
        current_height = 1
        while y + current_height < height:
            can_expand = True
            for dx in range(current_width):
                cx = x + dx
                cy = y + current_height
                next_v = voxels[cx, cy]
                if next_v is None or next_v.id != root_voxel.id or is_not_alone(cx, cy):
                    can_expand = False
                    break
            if can_expand:
                current_height += 1
            else:
                break
        
        for dy in range(current_height):
            for dx in range(current_width):
                dsu.union(to_index(x, y), to_index(x + dx, y + dy))

    for y in range(height):
        for x in range(width):
            if voxels[x, y] is None:
                continue
            if is_not_alone(x, y):
                continue
            create_one_set(x, y)

    # Group by root
    groups = {}
    for y in range(height):
        for x in range(width):
            if voxels[x, y] is None:
                continue
            root = dsu.find(to_index(x, y))
            if root not in groups:
                groups[root] = []
            groups[root].append((x, y))

    quads = []
    for root, coords in groups.items():
        min_x = min(c[0] for c in coords)
        max_x = max(c[0] for c in coords)
        min_y = min(c[1] for c in coords)
        max_y = max(c[1] for c in coords)
        
        voxel_id = voxels[min_x, min_y].id
        quad = create_mesh_quad(plane, min_x, max_x, min_y, max_y, voxel_id)
        quads.append(quad)
    
    return quads

def create_mesh_quad(plane, min_x, max_x, min_y, max_y, voxel_id):
    # Implementation depends on plane axis configuration
    # See coordinate transformation rules below
    pass
```

**Coordinate Transformation:**
The vertices of the `MeshQuad` are determined by the `VisiblePlane`'s axis configuration. Let `chunk` be the containing 3D chunk with dimensions `XDepth`, `YDepth`, `ZDepth`.

*   **Axis X, Ascending:**
    *   $x = \text{plane.SliceIndex}$
    *   $y_1 = \text{minY}, y_2 = \text{maxX} + 1$
    *   $z_1 = \text{minY}, z_2 = \text{maxY} + 1$
    *   Vertices: $V_1(x, y_1, z_1)$, $V_0(x, y_2, z_1)$, $V_3(x, y_2, z_2)$, $V_2(x, y_1, z_2)$
    *   Normal: $(1, 0, 0)$

*   **Axis X, Descending:**
    *   $x = \text{chunk.XDepth} - \text{plane.SliceIndex}$
    *   $y_1 = \text{chunk.YDepth} - \text{minX}, y_2 = \text{chunk.YDepth} - \text{maxX} - 1$
    *   $z_1 = \text{chunk.ZDepth} - \text{minY}, z_2 = \text{chunk.ZDepth} - \text{maxY} - 1$
    *   Vertices: $V_0(x, y_1, z_1)$, $V_1(x, y_2, z_1)$, $V_2(x, y_2, z_2)$, $V_3(x, y_1, z_2)$
    *   Normal: $(1, 0, 0)$

*   **Axis Y, Ascending:**
    *   $x_1 = \text{minY}, x_2 = \text{maxY} + 1$
    *   $y = \text{plane.SliceIndex}$
    *   $z_1 = \text{minX}, z_2 = \text{maxX} + 1$
    *   Vertices: $V_1(x_1, y, z_1)$, $V_0(x_1, y, z_2)$, $V_3(x_2, y, z_2)$, $V_2(x_2, y, z_1)$
    *   Normal: $(1, 0, 0)$ [Note: Source code snippet indicates this normal for Y-axis cases, which may be implementation-specific or an error in the source, but is cited as such]

*   **Axis Y, Descending:**
    *   $x_1 = \text{chunk.XDepth} - \text{minY}, x_2 = \text{chunk.XDepth} - \text{maxY} - 1$
    *   $y = \text{chunk.YDepth} - \text{plane.SliceIndex}$
    *   $z_1 = \text{chunk.ZDepth} - \text{minX}, z_2 = \text{chunk.ZDepth} - \text{maxX} - 1$
    *   Vertices: $V_0(x_1, y, z_1)$, $V_1(x_1, y, z_2)$, $V_2(x_2, y, z_2)$, $V_3(x_2, y, z_1)$
    *   Normal: $(1, 0, 0)$

*   **Axis Z, Descending:**
    *   $x_1 = \text{chunk.XDepth} - \text{minY}, x_2 = \text{chunk.XDepth} - \text{maxY} - 1$
    *   $y_1 = \text{chunk.YDepth} - \text{minX}, y_2 = \text{chunk.YDepth} - \text{maxX} - 1$
    *   $z = \text{chunk.ZDepth} - \text{plane.SliceIndex}$
    *   Vertices: $V_1(x_1, y_1, z)$, $V_0(x_1, y_2, z)$, $V_3(x_2, y_2, z)$, $V_2(x_2, y_1, z)$
    *   Normal: $(1, 0, 0)$

**Edge Cases and Pitfalls:**
*   **Boundary Checks:** Ensure all neighbor accesses in `IsNotAlone` and expansion loops are bounded by `width` and `height` to prevent out-of-bounds array access.
*   **Null Voxels:** The algorithm explicitly skips null voxels. If the input plane contains holes, they are ignored, and merging stops at the hole boundary.
*   **Precision:** The algorithm relies on integer coordinates and IDs. Floating-point precision issues do not apply to the merging logic, but vertex generation assumes unit-sized voxels.

## Subtopics

### [Disjoint Set Mesh Optimization](4495-disjoint-set-mesh-optimization.md)
An optimization pipeline that first culls occluded faces using a voxel occlusion algorithm, then processes the remaining visible faces plane-by-plane using a disjoint set data structure to merge adjacent quads. It is chosen when full 3D occlusion culling is required before meshing to reduce the number of planes processed. It differs from Disjoint Set Visible Plane Optimization by including the initial culling step.

### Visible Plane Representation
A data structure representing a 2D slice of visible voxels, defined by a specific iteration order (Major, Middle, Minor axes) and a slice index along the minor axis [S1, unofficial]. It is used to decompose the 3D voxel space into 2D planes for efficient greedy meshing. It sets itself apart by encapsulating the axis configuration and voxel data for a single slice.

## Operating characteristics
*   **Time Complexity:** $O(W \times H \times \alpha(W \times H))$ per plane, where $W$ and $H$ are the plane dimensions and $\alpha$ is the inverse Ackermann function (nearly constant) due to DSU operations. The expansion step is bounded by the plane size.
*   **Memory Complexity:** $O(W \times H)$ for the DSU structure and the voxel array.
*   **Performance:** Efficient for large contiguous regions of identical voxels, significantly reducing draw calls. Performance degrades if the plane is highly fragmented with many small unique regions.
*   **Strengths:** Simple to implement, low memory overhead, effective for voxel games with large solid areas.
*   **Limitations:** Only optimizes 2D planes; does not handle 3D occlusion or merging across planes. Requires pre-culled visible planes.
*   **When to Use:** When generating meshes for 2D slices of voxel data, such as in Minecraft-like terrain generation.
*   **When Not to Use:** When 3D occlusion culling is needed or when merging across multiple planes is required.

## Background
The method is part of the Voxel Mesh Optimization Library by Nicolas Goyon [S1, unofficial]. It builds on the concept of greedy meshing, adapted for 2D visible planes in a 3D voxel context. The use of Disjoint Set Union allows for efficient merging of adjacent identical regions.

## Sources
[S1] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
