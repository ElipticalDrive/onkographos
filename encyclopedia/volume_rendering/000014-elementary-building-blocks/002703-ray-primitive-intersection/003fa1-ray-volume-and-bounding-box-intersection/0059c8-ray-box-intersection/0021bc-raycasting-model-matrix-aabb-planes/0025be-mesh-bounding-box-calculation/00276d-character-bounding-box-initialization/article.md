# Character Bounding Box Initialization
Also called: Bounding Box Initialization for Characters
## Overview
Character Bounding Box Initialization computes the axis-aligned bounding box (AABB) enclosing a set of geometric primitives representing specific characters ('S', 'A', 'N', 'G') and arcs [S1, unofficial]. It optimizes sphere tracing queries by providing a tight spatial bound for ray-primitive intersection tests [S1, unofficial].
## How it works
**Inputs:** Character definitions for 'S', 'A', 'N', 'G' and arcs; an offset value $o$ [S1, unofficial].
**Outputs:** `surrounding_bbox` defined by minimum $(x_{min}, y_{min}, z_{min})$ and maximum $(x_{max}, y_{max}, z_{max})$ coordinates [S1, unofficial].
**Data Structures:** `Point` $(x, y, z)$; `AABB` with `min` and `max` vectors [G]. Bit packing is Not given in the sources [S1, unofficial].
**Steps:**
1. Define arcs for 'S' (`sphere_upper_arc`, `sphere_lower_arc`) with specific centers, radii, and angles [S1, unofficial].
2. Define vertices for 'A' (left leg, right leg, horizontal line) [S1, unofficial].
3. Define vertices for 'N' (left leg, right leg, connector) [S1, unofficial].
4. Define arc for 'G' [S1, unofficial].
5. Define vertices for 'G' [S1, unofficial].
6. Compute bounding box coordinates by finding the minimum and maximum x, y, z values across all defined primitives [S1, unofficial].
7. Create `surrounding_bbox` with these coordinates [S1, unofficial].
**Formulas:**
- Bounding box update for a line segment with endpoints $P_1(x_1, y_1, z_1)$ and $P_2(x_2, y_2, z_2)$: [G]
  $x_{min} = \min(x_{min}, x_1, x_2)$, $y_{min} = \min(y_{min}, y_1, y_2)$, $z_{min} = \min(z_{min}, z_1, z_2)$
  $x_{max} = \max(x_{max}, x_1, x_2)$, $y_{max} = \max(y_{max}, y_1, y_2)$, $z_{max} = \max(z_{max}, z_1, z_2)$
- Offset application: [S1, unofficial]
  $x_{min} \leftarrow x_{min} - o$, $y_{min} \leftarrow y_{min} - o$, $z_{min} \leftarrow z_{min} - o$
  $x_{max} \leftarrow x_{max} + o$, $y_{max} \leftarrow y_{max} + o$, $z_{max} \leftarrow z_{max} + o$
**Parameters:** `offset` $o = 0.1$ [S1, unofficial].
**Edge Cases and Pitfalls:** The offset $o = 0.1$ mitigates precision issues where rays might miss characters due to floating-point errors at the boundary [S1, unofficial]. Not given in the sources [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_character_bbox(primitives, offset=0.1):
    """
    Computes the bounding box for a set of character primitives.
    primitives: list of dicts, each with 'type' ('arc' or 'line') and coordinates.
    offset: float, margin to add to the bounding box.
    """
    min_x = float('inf')
    min_y = float('inf')
    min_z = float('inf')
    max_x = float('-inf')
    max_y = float('-inf')
    max_z = float('-inf')

    for prim in primitives:
        if prim['type'] == 'line':
            p1 = prim['p1']
            p2 = prim['p2']
            min_x = min(min_x, p1['x'], p2['x'])
            min_y = min(min_y, p1['y'], p2['y'])
            min_z = min(min_z, p1['z'], p2['z'])
            max_x = max(max_x, p1['x'], p2['x'])
            max_y = max(max_y, p1['y'], p2['y'])
            max_z = max(max_z, p1['z'], p2['z'])
        elif prim['type'] == 'arc':
            center = prim['center']
            radius = prim['radius']
            # The source does not provide the exact arc bounding box formula, so a conservative sphere bound is used [G].
            arc_min_x = center['x'] - radius
            arc_min_y = center['y'] - radius
            arc_min_z = center['z'] - radius
            arc_max_x = center['x'] + radius
            arc_max_y = center['y'] + radius
            arc_max_z = center['z'] + radius
            min_x = min(min_x, arc_min_x)
            min_y = min(min_y, arc_min_y)
            min_z = min(min_z, arc_min_z)
            max_x = max(max_x, arc_max_x)
            max_y = max(max_y, arc_max_y)
            max_z = max(max_z, arc_max_z)

    min_x -= offset
    min_y -= offset
    min_z -= offset
    max_x += offset
    max_y += offset
    max_z += offset

    return {'min': (min_x, min_y, min_z), 'max': (max_x, max_y, max_z)}
```
## Subtopics

### [Player Axis-Aligned Bounding Box (AABB) Construction](6971-player-axis-aligned-bounding-box-aabb-construction.md)
Computes the player's collision bounding box by offsetting the center position by a bottom offset and capping at a top offset. It provides a simple axis-aligned volume for raycasting against the voxel world, distinct from the more general character bounding box initialization which handles multiple entity types [4016-character-bounding-box-initialization.md].

## Operating characteristics
Time complexity is $O(N)$ where $N$ is the number of primitives, as each primitive is processed once to update min/max coordinates [G]. Memory complexity is $O(1)$ auxiliary space [G]. Measured performance, scaling, strengths, limitations, and usage guidelines are Not given in the sources [S1, unofficial].
## Background
This method is derived from the implementation in the SDF Sphere Tracing project by AlerianEmperor [S1, unofficial]. It is used to initialize the positions of characters ('S', 'A', 'N', 'G') and arcs, and to compute a surrounding bounding box to optimize sphere tracing queries [S1, unofficial].
## Sources
[S1] AlerianEmperor/SDF-Sphere-Tracing, https://github.com/AlerianEmperor/SDF-Sphere-Tracing, fetched 2026-10-26 (unofficial)
