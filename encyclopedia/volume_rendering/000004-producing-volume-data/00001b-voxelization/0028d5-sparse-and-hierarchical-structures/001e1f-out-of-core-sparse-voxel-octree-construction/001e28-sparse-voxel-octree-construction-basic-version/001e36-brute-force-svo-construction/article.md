# Brute Force SVO Construction

Also called: In-Memory SVO Construction, Sparse Voxel Octree (SVO) Construction, Top-down Octree Construction with Pruning

## Overview
Brute Force SVO Construction is a top-down algorithm that builds a complete Sparse Voxel Octree (SVO) in main memory by recursively subdividing the world space, followed by a depth-first search to prune empty branches [S2, unofficial]. It differs from its siblings, such as [Out-of-Core SVO Construction via Morton Order](1885-out-of-core-svo-construction-via-morton-order.md) and [Sparse Voxel Octree Construction via Morton Code Sorting](2227-sparse-voxel-construction-via-morton-code-sorting.md), by requiring the entire octree structure to reside in memory during construction and relying on post-construction pruning rather than incremental merging or sorting [S2, unofficial]. This approach is chosen for its implementation simplicity and recursive clarity, but is limited by high memory consumption and poor cache coherence due to scattered node allocation [S2, unofficial].

## How it works
The method constructs the SVO by recursively creating nodes from the root down to a specified maximum depth, checking for geometry intersections at each level. If a node contains geometry and is not at the maximum depth, it becomes an index node pointing to a child indirection grid. If it contains geometry at the maximum depth, it becomes a data node. After the full tree is built, a pruning pass removes empty branches.

### Inputs and Outputs
**Inputs:**
- `worldSize`: The side length of the root bounding box in world units [S1, unofficial].
- `maxDepth`: The maximum recursion depth, determining the finest voxel resolution [S1, unofficial].
- `geometries`: A list of scene geometry objects (e.g., spheres) used for intersection testing [S1, unofficial].

**Outputs:**
- A root `IndirectionGrid` representing the SVO, containing `Cell` objects that are either `INDEX` (pointing to child grids) or `DATA` (storing color and location) [S1, unofficial].

### Data Structures
**IndirectionGrid:**
- An array of 8 `Cell` objects, corresponding to the 8 octants of the current voxel [S1, unofficial].

**Cell:**
- `NodeType`: An enum with values `DATA` or `INDEX` [S1, unofficial].
- If `NodeType == DATA`:
  - `color`: A `Vector3f` storing the normalized color of the geometry [S1, unofficial].
  - `realLocation`: A `Vector3f` storing the real-world location of the geometry, normalized by `worldSize` [S1, unofficial].
- If `NodeType == INDEX`:
  - `textureIndex`: A `Vector3f` storing the 3D texture coordinate pointing to the child `IndirectionGrid` [S1, unofficial].

### Parameters
- `maxDepth`: Determines the finest resolution. Typical values depend on the required detail and memory constraints [S1, unofficial].
- `worldSize`: Defines the spatial extent of the root node [S1, unofficial].

### Formulas
- Child box size at a given depth:
  $$ \text{childBoxSize} = \frac{\text{worldSize}}{2^{\text{depth}+1}} $$
  where `depth` is the depth of the parent node [S1, unofficial].

### Steps
1. **Initialization**: Create an empty list for geometries and an `indirectionPool` with capacity $8 \times \text{maxDepth}^2$ [S1, unofficial].
2. **Root Creation**: Call `createNode(0, Vector3f(0, 0, 0))` to create the root node at depth 0 and origin [S1, unofficial].
3. **Recursive Node Creation (`createNode`)**:
   - Input: `depth`, `boxStart` (Vector3f).
   - Calculate `childBoxSize` using the formula above [S1, unofficial].
   - Iterate `i` from 0 to 7 for the 8 sub-nodes:
     - Calculate `childBoxStart` using `childBoxOffsets[i] * childBoxSize + boxStart` [S1, unofficial].
     - Calculate `childBoxEnd = childBoxStart + childBoxSize` [S1, unofficial].
     - Check for intersection between `childBoxStart`/`childBoxEnd` and any geometry in `geometries` using `Geometry.intersects()` [S1, unofficial].
     - If `depth + 1 != maxDepth` (not leaf):
       - If intersection is not null:
         - Recursively call `createNode(depth + 1, childBoxStart)` to create a child indirection grid [S1, unofficial].
         - Create an `Index` cell with a `textureIndex` pointing to the new grid's texture index [S1, unofficial].
         - Set this cell as the child node in the current `IndirectionGrid` [S1, unofficial].
     - If `depth + 1 == maxDepth` (leaf):
       - If intersection is not null:
         - Create a `Data` cell [S1, unofficial].
         - Set `color` by normalizing the geometry's color by dividing by `worldSize` [S1, unofficial].
         - Set `realLocation` by normalizing the geometry's real location by `worldSize` [S1, unofficial].
         - Set this cell as the child node [S1, unofficial].
4. **Pruning**: Perform a depth-first search on the constructed tree. If a node has no children with data (all children are empty or null), remove the node or mark it as empty [S2, unofficial].
5. **Gathering (Optional)**: To improve cache coherence, perform an extra pass to gather the pruned nodes into a sequential array structure [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3f:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

class Cell:
    def __init__(self, node_type, color=None, real_location=None, texture_index=None):
        self.node_type = node_type  # 'DATA' or 'INDEX'
        self.color = color
        self.real_location = real_location
        self.texture_index = texture_index

class IndirectionGrid:
    def __init__(self):
        self.cells = [None] * 8

class Geometry:
    @staticmethod
    def intersects(box_start, box_end, geometries):
        # Returns the first geometry intersecting the box, or None
        for geom in geometries:
            if geom.intersects_box(box_start, box_end):
                return geom
        return None

class Sphere:
    def __init__(self, center, radius, color):
        self.center = center
        self.radius = radius
        self.color = color

    def intersects_box(self, box_start, box_end):
        # Calculate closest and furthest points on the box relative to sphere center
        # Return True if distance <= radius
        closest = Vector3f(
            max(box_start.x, min(self.center.x, box_end.x)),
            max(box_start.y, min(self.center.y, box_end.y)),
            max(box_start.z, min(self.center.z, box_end.z))
        )
        dist_sq = (self.center.x - closest.x)**2 + \
                  (self.center.y - closest.y)**2 + \
                  (self.center.z - closest.z)**2
        return dist_sq <= self.radius**2

def create_node(depth, box_start, world_size, max_depth, geometries, indirection_pool):
    child_box_size = world_size / (2 ** (depth + 1))
    
    # Offsets for the 8 octants (0 to 7)
    # Assuming standard octree ordering: 0=000, 1=001, 2=010, 3=011, 4=100, 5=101, 6=110, 7=111
    # Each bit represents X, Y, Z direction
    child_offsets = [
        Vector3f(0, 0, 0), Vector3f(0, 0, 1), Vector3f(0, 1, 0), Vector3f(0, 1, 1),
        Vector3f(1, 0, 0), Vector3f(1, 0, 1), Vector3f(1, 1, 0), Vector3f(1, 1, 1)
    ]

    grid = IndirectionGrid()

    for i in range(8):
        offset = child_offsets[i]
        child_box_start = Vector3f(
            offset.x * child_box_size + box_start.x,
            offset.y * child_box_size + box_start.y,
            offset.z * child_box_size + box_start.z
        )
        child_box_end = Vector3f(
            child_box_start.x + child_box_size,
            child_box_start.y + child_box_size,
            child_box_start.z + child_box_size
        )

        intersecting_geom = Geometry.intersects(child_box_start, child_box_end, geometries)

        if intersecting_geom is not None:
            if depth + 1 < max_depth:
                # Not leaf level: create child indirection grid
                child_grid = create_node(depth + 1, child_box_start, world_size, max_depth, geometries, indirection_pool)
                # Store reference to child grid in pool and get texture index
                # Note: In a real implementation, textureIndex would be an index into the pool
                texture_index = Vector3f(0, 0, 0) # Placeholder for actual indexing logic
                cell = Cell(node_type='INDEX', texture_index=texture_index)
                grid.cells[i] = cell
            else:
                # Leaf level: create data cell
                # Normalize color by worldSize
                normalized_color = Vector3f(
                    intersecting_geom.color.x / world_size,
                    intersecting_geom.color.y / world_size,
                    intersecting_geom.color.z / world_size
                )
                # Normalize real location by worldSize
                # Assuming intersecting_geom has a 'center' or 'location' field
                real_location = Vector3f(
                    intersecting_geom.center.x / world_size,
                    intersecting_geom.center.y / world_size,
                    intersecting_geom.center.z / world_size
                )
                cell = Cell(node_type='DATA', color=normalized_color, real_location=real_location)
                grid.cells[i] = cell

    return grid

def prune_empty_nodes(grid):
    # Depth-first search to remove empty branches
    # If all children are None, return None
    has_data = False
    for i in range(8):
        if grid.cells[i] is not None:
            if grid.cells[i].node_type == 'INDEX':
                # Recursively prune child
                # Note: In a real implementation, we would need to access the child grid
                # This pseudocode assumes the child grid is accessible via the texture_index
                # For simplicity, we assume the child grid is passed or stored in a way
                # that allows recursive pruning.
                # Here we just mark it as potentially empty if the child is empty
                pass 
            else:
                has_data = True
    
    if not has_data:
        return None
    return grid
```

### Edge Cases and Pitfalls
- **Memory Usage**: The entire octree structure must be held in memory at runtime, leading to high memory usage [S2, unofficial].
- **Cache Coherence**: After pruning, octree nodes are spread all over memory; an extra pass is required to gather them into a sequential structure for cache coherence [S2, unofficial].
- **Texture Size Limit**: Index nodes can only point to $2^8=255$ values along each axis, limiting the max texture size to 256 [S1, unofficial].
- **Density Assumption**: The texture size calculation assumes 1/2 of the space is empty, which may break if the scene is dense [S1, unofficial].

## Subtopics

### [Sparse Voxel Octree Construction from Heightmap](13131-sparse-voxel-octree-construction-from-heightmap.md)
This method constructs a Sparse Voxel Octree (SVO) by sampling a 2D heightmap to determine the occupancy of 3D voxel nodes, creating a child node only if the height value at a corner lies within the vertical span of the voxel bounding box. It differs from general brute-force SVO construction by bypassing the need for a pre-existing 3D volume or point cloud, instead deriving the 3D structure directly from 2D terrain data. This approach is specifically suited for terrain rendering where the geometry is defined by a scalar field over a 2D plane.

## Operating characteristics
- **Time Complexity**: Not explicitly given in the sources, but involves a full top-down traversal of the octree up to `maxDepth`, followed by a pruning pass. The complexity is proportional to the number of nodes in the full octree, which is $O(8^{\text{maxDepth}})$ in the worst case (fully populated), but significantly less for sparse scenes.
- **Memory Complexity**: High, as the entire octree structure is held in memory during construction [S2, unofficial]. The memory usage is proportional to the number of nodes created before pruning.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales poorly with increasing `maxDepth` due to exponential growth in node count and memory requirements [S2, unofficial].
- **Strengths**: Simple implementation, recursive structure is easy to reason about, and sparsify code can be written recursively [S2, unofficial].
- **Limitations**: High memory usage, poor cache coherence without an extra gathering pass, and limited by texture size constraints for index nodes [S1, S2, unofficial].
- **When to use**: When implementation simplicity is prioritized over performance and memory efficiency, and when the scene size and depth are small enough to fit in memory.
- **When not to use**: For large scenes, high-resolution requirements, or when memory constraints are tight.

## Background
The brute force approach was one of the first methods proposed for constructing Sparse Voxel Octrees [S2, unofficial]. It was introduced as a baseline to demonstrate the concept of sparsifying a dense voxel grid by pruning empty branches [S2, unofficial]. The method is described in Jeroen Baert's blog post on out-of-core construction of SVOs, where it is contrasted with more efficient methods using Morton codes [S2, unofficial]. The implementation details, including data structures and intersection tests, are also found in the `lwjgl-voxel-raycasting` repository [S1, unofficial].

## Sources
[S1] RvanderLaan/lwjgl-voxel-raycasting, https://github.com/RvanderLaan/lwjgl-voxel-raycasting, fetched 2026-10-24 (unofficial)
[S2] Out-Of-Core construction of Sparse Voxel Octrees - Jeroen Baert's Blog, https://www.forceflow.be/2012/07/24/out-of-core-construction-of-sparse-voxel-octrees/, fetched 2026-10-30 (unofficial)
