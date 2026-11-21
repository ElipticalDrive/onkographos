# VoxlapGL

Also called: VoxelRenderer

## Overview
VoxlapGL is a raycasting technique that renders a 3D voxel grid by exploiting vertical ray coherence to cut the space with multiple planes [S1, unofficial]. It determines visible voxel segments in software and projects them as rectangles onto the screen using an OpenGL stencil buffer [S1, unofficial]. Unlike its sibling [Voxel Raycasting with Vertical Slicing](2296-voxel-raycasting-with-vertical-slicing.md), which renders and combines lines, VoxlapGL projects rectangles and contains no quadrant distinctions in its traversal code [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel array (Voxel Space) and a camera position/orientation.
*   **Output:** A 2D screen image.

**Data Structures:**
The memory layout matches the VXL format [S1, unofficial].
1.  **Column Pointers:** A 1024x1024 array of pointers, each pointing to the start of a column's data [S1, unofficial].
2.  **Column Data:** Each column stores a simple array containing:
    *   The number of visible parts.
    *   The length of each visible part.
    *   The actual voxel data [S1, unofficial].
    *   *Note:* Storing solid/air information in a simple boolean array for a 1024x1024x256 map would require 32MB of memory without any actual voxel data [S1, unofficial].

**Algorithm Steps:**
1.  **Plane Selection:** Determine the number of raycast planes to cast based on the view direction [S1, unofficial]:
    *   ~300 planes when looking towards the horizon.
    *   ~800 planes when viewing from above.
    *   ~1000 planes when looking straight up or down [S1, unofficial].
    *   Viewspace culling is applied to optimize this count [S1, unofficial].
2.  **Grid Traversal:** For each plane, step through all columns hit by the raycast plane. The traversal uses efficient grid stepping without quadrant distinctions, using bitwise operations and incremental updates [S1, unofficial].
3.  **Visibility Determination:** In software, determine which parts of the columns are visible [S1, unofficial].
4.  **Projection:** Project all visible parts in the plane onto a virtual projection line that maintains a constant distance to the eye point and is perpendicular to the ground [S1, unofficial].
5.  **Rendering:** Use OpenGL to project a rectangle onto the screen using a stencil buffer, filling the entire screen with the projected data [S1, unofficial].

**Grid Traversal Pseudocode:**
The core grid stepping logic handles all 4 cases in a single loop using incremental operations [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization variables
# ip: current (x,y) grid offset
# d: x-y dot product distance
# tp: array of two integers representing target positions
# ti: array of two integers representing step increments
# ipi: array of two integers representing index increments

# Loop condition: while traversal is valid
# Inside loop:
k = (((unsigned long)(tp[1] - tp[0])) >> 31)
d = tp[k]
tp[k] += ti[k]
ip += ipi[k]
```
*Note: `BLOCKSCAN.KC` from EVALDRAW provides a working example of this algorithm [S1, unofficial].*

**Parameters:**
*   **Field of View:** 90 degrees [S1, unofficial].
*   **Viewspace Culling:** Provides a 4x speed boost when looking towards the horizon [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Usage:** Storing solid/air information in a simple array for a 1024x1024x256 map requires 32MB of memory without any actual voxel data [S1, unofficial].
*   **Performance:** Using OpenGL for the second pass (copy & project color data) yields minimal or negative speedup on older graphics hardware and may cause deceleration on newer hardware [S1, unofficial].
*   **Boundary Calculation:** Calculating grid boundaries for "perfect" looking cubes using interpolation requires one multiplication per column [S1, unofficial].

## Operating characteristics
*   **Hardware Requirements:** Requires at least a Pentium Pro CPU and a graphics adapter supporting OpenGL [S1, unofficial].
*   **Performance:** Viewspace culling is critical for performance, providing a 4x speed boost at a 90-degree FOV when looking towards the horizon [S1, unofficial].
*   **Limitations:** The reliance on OpenGL for the second pass can be a bottleneck; software-based projection might be faster on older hardware, while newer hardware may also suffer from deceleration due to this approach [S1, unofficial].
*   **Complexity:** The grid traversal is optimized to avoid quadrant distinctions, reducing branching overhead [S1, unofficial].

## Background
VoxlapGL was developed by Sebastian Scholz in 2004/2005 as a forum demo [S1, unofficial]. It was created as an alternative to other voxel rendering techniques like VoxLap, aiming to exploit vertical ray coherence more efficiently [S1, unofficial]. The demo was distributed as a ZIP archive with batch files for execution [S1, unofficial].

## Sources
[S1] Forum archive / Ken Silverman / VoxlapGL, https://jonof.id.au/forum/topic-166.html, fetched 2026-10-24 (unofficial)
