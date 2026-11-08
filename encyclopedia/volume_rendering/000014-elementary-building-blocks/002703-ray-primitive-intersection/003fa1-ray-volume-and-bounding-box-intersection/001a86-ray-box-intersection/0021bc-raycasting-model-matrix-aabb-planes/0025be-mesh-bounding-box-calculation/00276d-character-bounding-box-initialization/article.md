# Character Bounding Box Initialization
Also called: Bounding Box Initialization for Characters
## Overview
Character Bounding Box Initialization computes the axis-aligned bounding box (AABB) that encloses a set of geometric primitives representing specific characters (such as 'S', 'A', 'N', 'G') and arcs. This method is used to optimize sphere tracing queries by providing a tight spatial bound for ray-primitive intersection tests [S1, unofficial]. It differs from [Mesh Bounding Box Calculation](3612-mesh-bounding-box-calculation.md) by operating on explicit parametric primitives (arcs and line segments) rather than vertex lists, and from [Bounding Cube Creation](4007-bounding-cube-creation.md) by computing a tight AABB rather than a potentially looser cube.
## How it works
**Inputs:**
- A set of character definitions, where each character consists of one or more primitives.
- Primitive types:
  - `SphereArc`: Defined by a center point $C \in \mathbb{R}^3$, a radius $r \in \mathbb{R}$, and an angular range $[\theta_{start}, \theta_{end}]$.
  - `LineSegment`: Defined by two endpoints $P_1, P_2 \in \mathbb{R}^3$.
- An offset value $o$ to expand the bounding box slightly, preventing precision errors or missed intersections near the character boundaries.

**Outputs:**
- `surrounding_bbox`: An AABB defined by minimum coordinates $(x_{min}, y_{min}, z_{min})$ and maximum coordinates $(x_{max}, y_{max}, z_{max})$.

**Data Structures:**
- `Point`: A 3D vector $(x, y, z)$.
- `AABB`: A struct containing `min` and `max` vectors.

**Parameters:**
- `offset`: The expansion margin. Typical value is $0.1$ [S1, unofficial].

**Steps:**
1. Initialize `global_min` to $(+\infty, +\infty, +\infty)$ and `global_max` to $(-\infty, -\infty, -\infty)$.
2. For each character in the set:
   a. For each primitive in the character:
      i. If the primitive is a `SphereArc`:
         - Compute the bounding box of the arc. The arc lies on a sphere of radius $r$ centered at $C$. The bounding box of the full sphere is $[C_x-r, C_x+r] \times [C_y-r, C_y+r] \times [C_z-r, C_z+r]$.
         - Refine this box by intersecting it with the angular constraints. The exact min/max coordinates depend on the specific angular range and the orientation of the arc's plane. [G] In the source implementation, specific arcs for 'S' are defined with explicit centers, radii, and angles, implying the bounding box is computed by evaluating the extremal points of the arc segment within its plane.
      ii. If the primitive is a `LineSegment` (used for 'A', 'N', 'G' legs/connectors):
         - Update `global_min` with $\min(global\_min.x, P_1.x, P_2.x)$, $\min(global\_min.y, P_1.y, P_2.y)$, $\min(global\_min.z, P_1.z, P_2.z)$.
         - Update `global_max` with $\max(global\_max.x, P_1.x, P_2.x)$, $\max(global\_max.y, P_1.y, P_2.y)$, $\max(global\_max.z, P_1.z, P_2.z)$.
3. Apply the offset:
   - $x_{min} = global\_min.x - o$
   - $y_{min} = global\_min.y - o$
   - $z_{min} = global\_min.z - o$
   - $x_{max} = global\_max.x + o$
   - $y_{max} = global\_max.y + o$
   - $z_{max} = global\_max.z + o$
4. Return `surrounding_bbox` defined by $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_character_bbox(primitives, offset=0.1):
    """
    Computes the bounding box for a set of character primitives.
    primitives: list of dicts, each with 'type' ('arc' or 'line') and coordinates.
    offset: float, margin to add to the bounding box.
    """
    # Initialize with infinity
    min_x, min_y, min_z = float('inf'), float('inf'), float('inf')
    max_x, max_y, max_z = float('-inf'), float('-inf'), float('-inf')

    for prim in primitives:
        if prim['type'] == 'line':
            p1 = prim['p1']
            p2 = prim['p2']
            # Update min
            min_x = min(min_x, p1['x'], p2['x'])
            min_y = min(min_y, p1['y'], p2['y'])
            min_z = min(min_z, p1['z'], p2['z'])
            # Update max
            max_x = max(max_x, p1['x'], p2['x'])
            max_y = max(max_y, p1['y'], p2['y'])
            max_z = max(max_z, p1['z'], p2['z'])
            
        elif prim['type'] == 'arc':
            center = prim['center']
            radius = prim['radius']
            # Note: The source defines arcs with specific centers, radii, and angles.
            # A precise implementation would compute the bounding box of the arc segment.
            # For simplicity and based on the source's explicit definition of arcs,
            # we assume the bounding box of the arc is computed by its extremal points.
            # Here we use the bounding box of the full circle as a conservative estimate
            # or the source implies specific pre-computed bounds for 'S'.
            # To be strictly implementable from the text which lists "sphere_upper_arc" etc,
            # we must compute the bounds of the arc.
            # The source does not provide the exact formula for arc bounding box, 
            # but implies it is part of the initialization.
            # [G] Standard approach: Project arc endpoints and any tangential extrema 
            # onto axes. For an arc in XY plane centered at (cx, cy, cz) with radius r:
            # x bounds: [cx - r, cx + r] intersected with arc x-range.
            # y bounds: [cy - r, cy + r] intersected with arc y-range.
            # z bounds: [cz, cz] (if planar in XY).
            # Since the source code snippet shows complex trig functions, the arc bounds 
            # are likely computed analytically. We will use the full sphere bounds 
            # as a safe fallback if arc-specific logic is not detailed, 
            # but the source says "Compute bounding box coordinates based on min/max x, y, z of all primitives".
            # We will assume the arc's bounding box is calculated and added to min/max.
            arc_min, arc_max = compute_arc_bbox(prim)
            min_x = min(min_x, arc_min['x'])
            min_y = min(min_y, arc_min['y'])
            min_z = min(min_z, arc_min['z'])
            max_x = max(max_x, arc_max['x'])
            max_y = max(max_y, arc_max['y'])
            max_z = max(max_z, arc_max['z'])

    # Apply offset
    min_x -= offset
    min_y -= offset
    min_z -= offset
    max_x += offset
    max_y += offset
    max_z += offset

    return {
        'min': {'x': min_x, 'y': min_y, 'z': min_z},
        'max': {'x': max_x, 'y': max_y, 'z': max_z}
    }

def compute_arc_bbox(prim):
    """
    Computes the bounding box for a spherical arc.
    This is a placeholder for the specific logic implied by the source's 
    definition of arcs with centers, radii, and angles.
    """
    # [G] Implementation detail: The source defines arcs for 'S' with specific parameters.
    # A robust implementation would calculate the min/max of the arc segment.
    # For the purpose of this article, we assume this helper exists and returns 
    # the tight bounding box of the arc.
    center = prim['center']
    radius = prim['radius']
    # Conservative approximation if exact arc bounds are not provided in text:
    # Use the bounding box of the sphere containing the arc.
    return {
        'min': {'x': center['x'] - radius, 'y': center['y'] - radius, 'z': center['z'] - radius},
        'max': {'x': center['x'] + radius, 'y': center['y'] + radius, 'z': center['z'] + radius}
    }
```

**Edge Cases and Pitfalls:**
- **Precision:** Using floating-point arithmetic for min/max comparisons can lead to slight inaccuracies. The offset $o=0.1$ helps mitigate issues where rays might miss the character due to floating-point errors at the boundary [S1, unofficial].
- **Arc Orientation:** The bounding box of an arc depends on its orientation in 3D space. If the arc is not aligned with the axes, the bounding box calculation must account for the rotation. The source implies specific arcs are defined, so their bounding boxes are computed based on their specific parameters.
- **Empty Input:** If no primitives are provided, the bounding box should be handled gracefully (e.g., return an invalid box or a default size).

## Subtopics

### [Player Axis-Aligned Bounding Box (AABB) Construction](6971-player-axis-aligned-bounding-box-aabb-construction.md)
Computes the player's collision bounding box by offsetting the center position by a bottom offset and capping at a top offset. It provides a simple axis-aligned volume for raycasting against the voxel world, distinct from the more general character bounding box initialization which handles multiple entity types [4016-character-bounding-box-initialization.md].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of primitives in the character set. Each primitive is processed once to update the min/max coordinates.
- **Memory Complexity:** $O(1)$ auxiliary space, as only the global min and max coordinates are stored.
- **Strengths:** Simple to implement, fast, and provides a tight bound for sphere tracing optimization.
- **Limitations:** The bounding box is axis-aligned, which may not be optimal for rotated characters. The offset value is fixed and may need tuning for different character sizes or precision requirements.
- **When to use:** When optimizing sphere tracing for static character models composed of simple primitives.
- **When not to use:** For complex meshes with many vertices, [Mesh Bounding Box Calculation](3612-mesh-bounding-box-calculation.md) is more appropriate. For dynamic characters, the bounding box may need to be recomputed every frame.

## Background
This method is derived from the implementation in the SDF Sphere Tracing project by AlerianEmperor [S1, unofficial]. It is used to initialize the positions of characters ('S', 'A', 'N', 'G') and arcs, and to compute a surrounding bounding box to optimize sphere tracing queries. The specific implementation details, including the definition of arcs and the offset value, are found in the source code repository.

## Sources
[S1] AlerianEmperor/SDF-Sphere-Tracing, https://github.com/AlerianEmperor/SDF-Sphere-Tracing, fetched 2026-10-26 (unofficial)
