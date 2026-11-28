# LINE and CHECK Chunks

Also called: 45 degree lines, Checkerboxes

## Overview
LINE and CHECK chunks are optional compression primitives for the XYZI+CUBE format that handle specific geometric patterns—45-degree lines and checkerboards—respectively, which standard cubes cannot compress efficiently [S1, unofficial]. They are distinct from the primary CUBE primitive by targeting linear and alternating spatial distributions rather than solid volumetric blocks [S1, unofficial].

## How it works
The method operates as a post-processing step on the voxel data after standard cube detection. The implementation must first identify and remove all valid cubes from the dataset, then apply CHECK detection, and finally apply LINE detection to the remaining voxels [S1, unofficial].

### Data Structures and Layout

**LINE Chunk**
*   **Overhead:** 12 bytes per chunk (header/metadata not detailed in sources) [S1, unofficial].
*   **Per-Line Data:** 6 bytes per line [S1, unofficial].
*   **Fields:**
    1.  Starting point (3 coordinates, likely 1-byte integers given the 6-byte total for point+dir+len+color, though exact bit-packing is not specified) [S1, unofficial].
    2.  Direction (1 vector component, representing a 45-degree axis) [S1, unofficial].
    3.  Length (1 integer) [S1, unofficial].
    4.  Colour (1 index) [S1, unofficial].
    *Note: The source states "6 bytes per line (starting point, direction, length, colour)" but does not specify the bit-width of each field. A decoder must assume a fixed layout provided by the file header or standard convention.* [G]

**CHECK Chunk**
*   **Overhead:** Not specified in sources.
*   **Per-Checkerbox Data:** 8 bytes per checkerbox [S1, unofficial].
*   **Fields:**
    1.  Bounds/Position (defining the extent of the checkerboard pattern) [S1, unofficial].
    2.  Pattern parameters (phase/orientation) [S1, unofficial].
    3.  Colours (typically two colors for a checkerboard) [S1, unofficial].
    *Note: Exact field breakdown for the 8 bytes is not provided in the sources.* [G]

### Algorithm Steps

1.  **Input:** A 3D voxel grid $V$ with dimensions $W \times H \times D$, where each voxel has a color index $c$ and an opacity flag.
2.  **Preprocessing:** Load $V$ into a 3D matrix structure for $O(1)$ neighbor access [S1, unofficial].
3.  **Cube Removal:**
    *   Iterate through $V$ to find all maximal cubes of uniform color.
    *   Mark all voxels belonging to these cubes as "processed".
    *   Store these cubes in the CUBE chunk list.
4.  **CHECK Detection:**
    *   Iterate through unprocessed voxels.
    *   Identify regions forming a checkerboard pattern (alternating colors in a grid).
    *   If a valid checkerbox is found, store it in the CHECK chunk list.
    *   Mark these voxels as "processed".
5.  **LINE Detection:**
    *   Iterate through remaining unprocessed voxels.
    *   Identify contiguous lines aligned with 45-degree diagonals (e.g., $(1,1,0), (1,0,1), (0,1,1)$ and their negatives) [S1, unofficial].
    *   For each line, record the start point, direction, length, and color.
    *   Store in the LINE chunk list.
    *   Mark these voxels as "processed".
6.  **Output:** The union of CUBE, CHECK, and LINE chunks. Any remaining unprocessed voxels are stored in the standard XYZI format (or handled as an error/edge case depending on implementation) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_voxels(voxel_grid):
    """
    Compress voxel grid using CUBE, CHECK, and LINE chunks.
    voxel_grid: 3D array [x][y][z] containing color indices.
    Returns: List of chunk objects.
    """
    width, height, depth = voxel_grid.dimensions()
    processed = set() # Set of (x,y,z) tuples
    chunks = []

    # Step 1: Find Cubes
    # (Assuming a standard cube-finding algorithm exists or is implemented)
    cubes = find_maximal_cubes(voxel_grid, processed)
    chunks.extend(cubes)

    # Step 2: Find CHECK (Checkerboxes)
    # Iterate over remaining voxels to find checkerboard patterns
    checkers = find_checkerboxes(voxel_grid, processed)
    chunks.extend(checkers)

    # Step 3: Find LINE (45-degree lines)
    # Iterate over remaining voxels to find diagonal lines
    lines = find_45_degree_lines(voxel_grid, processed)
    chunks.extend(lines)

    # Step 4: Handle remaining voxels
    # These are not compressed by the special chunks
    remaining = get_remaining_voxels(voxel_grid, processed)
    if remaining:
        chunks.append(XYZIChunk(remaining))

    return chunks

def find_45_degree_lines(grid, processed):
    """
    Finds contiguous lines along 45-degree diagonals.
    Directions: (1,1,0), (1,0,1), (0,1,1) and their inverses.
    """
    lines = []
    directions = [
        (1, 1, 0), (1, 0, 1), (0, 1, 1),
        (-1, -1, 0), (-1, 0, -1), (0, -1, -1)
    ]
    
    # Note: A full implementation requires scanning for start points
    # and tracing length. The source does not provide the specific
    # start-point detection logic, only the storage format.
    # [G] Placeholder for the specific traversal logic required to
    # identify unique line segments without double-counting.
    
    for x in range(grid.width):
        for y in range(grid.height):
            for z in range(grid.depth):
                if (x, y, z) in processed:
                    continue
                
                color = grid.get(x, y, z)
                if color is empty:
                    continue

                for dx, dy, dz in directions:
                    # Check if this is a valid start of a line
                    # (e.g., previous voxel in direction is not same color/processed)
                    prev_x, prev_y, prev_z = x - dx, y - dy, z - dz
                    if is_valid_start(grid, prev_x, prev_y, prev_z, color):
                        length = 0
                        cx, cy, cz = x, y, z
                        while is_valid_voxel(grid, cx, cy, cz, color):
                            processed.add((cx, cy, cz))
                            length += 1
                            cx += dx
                            cy += dy
                            cz += dz
                        
                        if length > 1: # Minimum length for a line
                            lines.append(LineChunk(
                                start=(x, y, z),
                                direction=(dx, dy, dz),
                                length=length,
                                color=color
                            ))
    return lines
```

### Edge Cases and Pitfalls
*   **Order of Operations:** The order (Cubes $\rightarrow$ CHECK $\rightarrow$ LINE) is critical. If lines are checked before cubes, large solid blocks might be fragmented into many small lines, increasing overhead [S1, unofficial].
*   **Checkerboard Worst-Case:** The checkerboard pattern is considered a "worst-case scenario" for compression efficiency relative to the overhead, potentially equivalent to raw XYZI storage if not handled well [S1, unofficial].
*   **Overhead:** LINE chunks have a 12-byte overhead. Short lines may result in net size increase if the 6-byte data + overhead exceeds the raw voxel storage [S1, unofficial].

## Subtopics

### Hierarchical Chunk Overlapping
A hierarchy system allowing chunk overlapping to handle patterns like checkerboards that standard chunks cannot efficiently compress [S1, unofficial].
*   **Definition:** One color (typically the most frequent) is assigned to a "parent" chunk filling the entire model. Remaining voxels (including air) are stored as "children" at their specific locations, with the decoder resolving overlaps based on priority [S1, unofficial].
*   **Trade-offs:** Reduces checkerboard size by half but increases encoding/decoding speed cost. Storing air as children can increase file size compared to standard XYZI+CUBE if the hierarchy does not sufficiently reduce the volume of other chunks [S1, unofficial].
*   **Implementation Pitfall:** To ensure optimal size, the encoder may need to test all compression schemes (standard vs. hierarchical) and pick the smallest result, due to variable size reduction benefits [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Encoding is slower than standard XYZI+CUBE due to the multiple passes required for cube, check, and line detection [S1, unofficial]. Hierarchical overlapping adds further encoding/decoding overhead [S1, unofficial].
*   **Memory Complexity:** Depends on the number of detected patterns. LINE chunks use 6 bytes + 12 byte overhead per line; CHECK chunks use 8 bytes per checkerbox [S1, unofficial].
*   **Performance:** The checkerbox problem is a known weakness of the base format. Implementing CHECK or Hierarchical Overlapping mitigates this but introduces speed costs [S1, unofficial].
*   **When to Use:** Use LINE and CHECK when the source data contains significant 45-degree diagonal structures or checkerboard patterns. Use Hierarchical Overlapping only if the checkerboard reduction outweighs the overhead of storing air and the performance cost is acceptable [S1, unofficial].

## Background
The LINE and CHECK chunks, along with the Hierarchical Chunk Overlapping variant, were proposed in an unofficial discussion on the `ephtracy/voxel-model` GitHub repository regarding the optimization of the vox format [S1, unofficial]. The ideas were contributed by users Foxiquek and Jelle Bouma in August 2022 [S1, unofficial].

## Sources
[S1] Optional optimised format, https://github.com/ephtracy/voxel-model/issues/52, fetched 2026-10-28 (unofficial)
