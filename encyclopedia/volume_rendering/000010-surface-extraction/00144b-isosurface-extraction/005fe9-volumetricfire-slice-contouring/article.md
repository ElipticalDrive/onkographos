# VolumetricFire Slice Contouring

Also called: Slice-based contouring, active edge list contouring, VolumetricFire contouring

## Overview
VolumetricFire Slice Contouring is a slice-based isosurface extraction algorithm that processes volumetric data sequentially to generate triangle fans representing the contour [S1, unofficial]. It distinguishes itself from general-purpose methods like Marching Cubes by maintaining an active edge list with split and merge operations to handle topology changes dynamically across slices, specifically optimized for rendering volumetric fire effects [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of corner positions (`_posCorners`), corner texture coordinates (`_texCorners`), corner distances (`cornerDistance`), slice spacing (`_sliceSpacing`), minimum distance (`minDistance`), and a priority queue of edge expirations (`expirations`) [S1, unofficial].
*   **Output:** Buffers of vertex positions (`_points`), texture coordinates (`_texCoords`), and triangle indices (`_indexes`) [S1, unofficial].

**Data Structures:**
*   **Active Edge (`activeEdge`):** An object with the following fields:
    *   `expired`: Boolean, indicates if the edge is no longer active.
    *   `startIndex`: Number, index of the starting corner.
    *   `endIndex`: Number, index of the ending corner.
    *   `deltaPos`: Vector3, position increment per slice.
    *   `deltaTex`: Vector3, texture coordinate increment per slice.
    *   `pos`: Vector3, current position of the edge intersection.
    *   `tex`: Vector3, current texture coordinate of the edge intersection.
    *   `cur`: Number, index in the `activeEdges` array.
    *   `prev`: Number, index of the previous edge in the linked list.
    *   `next`: Number, index of the next edge in the linked list [S1, unofficial].
*   **Active Edges Array (`activeEdges`):** An array storing references to `activeEdge` objects [S1, unofficial].
*   **Priority Queue (`expirations`):** Stores edges with their expiration priority (corner distance) [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize `firstEdge` to 0, `nextEdge` to 0, and `expirations` as an empty priority queue [S1, unofficial].
    *   For each of the 3 corner neighbors of the `maxCorner` (indexed `i` from 0 to 2):
        *   Call `createEdge(maxCorner, cornerNeighbors[maxCorner][i])` to generate an initial edge [S1, unofficial].
        *   Set `activeEdge.prev = (i + 2) % 3` and `activeEdge.next = (i + 1) % 3` to form the initial linked list cycle [S1, unofficial].

2.  **Slice Iteration:**
    *   Iterate `sliceDistance` from `maxCorner` down to `minDistance` using `sliceSpacing` [S1, unofficial].

3.  **Edge Expiration and Topology Management:**
    *   While `expirations.top().priority >= sliceDistance`:
        *   Pop the edge `edge` from `expirations` [S1, unofficial].
        *   If `edge.expired` is true, continue to the next iteration [S1, unofficial].
        *   Check if `edge.endIndex` differs from both `activeEdges[edge.prev].endIndex` and `activeEdges[edge.next].endIndex` [S1, unofficial]:
            *   **Split:** If different, the edge must split.
                *   Mark `edge.expired = true` [S1, unofficial].
                *   Create `activeEdge1` using `createEdge(edge.endIndex, incomingEdges[edge.endIndex][edge.startIndex])` [S1, unofficial].
                *   Set `activeEdge1.prev = edge.prev` and update `activeEdges[edge.prev].next = nextEdge - 1` [S1, unofficial].
                *   Set `activeEdge1.next = nextEdge` [S1, unofficial].
                *   Create `activeEdge2` using `createEdge(edge.endIndex, incomingEdges[edge.endIndex][activeEdge1.endIndex])` [S1, unofficial].
                *   Set `activeEdge2.prev = nextEdge - 2` and `activeEdge2.next = edge.next` [S1, unofficial].
                *   Update `activeEdges[activeEdge2.next].prev = nextEdge - 1` [S1, unofficial].
                *   Set `firstEdge = nextEdge - 1` [S1, unofficial].
            *   **Merge:** If `edge.endIndex` matches either neighbor's `endIndex`:
                *   Identify `prev` and `next` edges to be merged based on which neighbor matches [S1, unofficial].
                *   Mark both `prev.expired = true` and `next.expired = true` [S1, unofficial].
                *   Create a new `activeEdge` using `createEdge(edge.endIndex, incomingEdges[edge.endIndex][prev.startIndex])` [S1, unofficial].
                *   Update links: `activeEdge.prev = prev.prev`, `activeEdges[activeEdge.prev].next = nextEdge - 1`, `activeEdge.next = next.next`, `activeEdges[activeEdge.next].prev = nextEdge - 1` [S1, unofficial].
                *   Set `firstEdge = nextEdge - 1` [S1, unofficial].

4.  **Vertex Collection:**
    *   Initialize `cur = firstEdge` and `count = 0` [S1, unofficial].
    *   Do-While loop while `cur !== firstEdge`:
        *   Increment `count` [S1, unofficial].
        *   Retrieve `activeEdge = activeEdges[cur]` [S1, unofficial].
        *   Push `activeEdge.pos` components to `_points` buffer [S1, unofficial].
        *   Push `activeEdge.tex` components to `_texCoords` buffer [S1, unofficial].
        *   Update `activeEdge.pos.add(activeEdge.deltaPos)` and `activeEdge.tex.add(activeEdge.deltaTex)` [S1, unofficial].
        *   Set `cur = activeEdge.next` [S1, unofficial].

5.  **Index Generation:**
    *   For `i` from 2 to `count - 1`:
        *   Push indices `(nextIndex, nextIndex + i - 1, nextIndex + i)` to `_indexes` buffer to form triangle fans [S1, unofficial].
        *   Increment `nextIndex` by 1 for each vertex added (implicit in the loop logic, though `nextIndex` update is not explicitly shown in the snippet, it is standard for fan generation) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_fire_slice_contouring(pos_corners, tex_corners, corner_distance, slice_spacing, min_distance, max_corner, corner_neighbors, incoming_edges):
    active_edges = []
    expirations = PriorityQueue()
    first_edge = 0
    next_edge = 0
    points = []
    tex_coords = []
    indexes = []
    next_index = 0

    def create_edge(start_index, end_index):
        if next_edge >= 12:
            return None
        
        # Calculate range to avoid division by zero
        range_val = corner_distance[start_index] - corner_distance[end_index]
        
        delta_pos = Vector3()
        delta_tex = Vector3()
        pos = Vector3()
        tex = Vector3()
        
        if range_val != 0.0:
            irange = 1.0 / range_val
            
            # deltaPos = (posCorners[endIndex] - posCorners[startIndex]) * irange
            delta_pos.sub_vectors(pos_corners[end_index], pos_corners[start_index]).multiply_scalar(irange)
            
            # deltaTex = (texCorners[endIndex] - texCorners[startIndex]) * irange
            delta_tex.sub_vectors(tex_corners[end_index], tex_corners[start_index]).multiply_scalar(irange)
            
            step = corner_distance[start_index] - slice_distance
            
            # pos = deltaPos * step + posCorners[startIndex]
            pos.add_vectors(delta_pos.clone().multiply_scalar(step), pos_corners[start_index])
            
            # tex = deltaTex * step + texCorners[startIndex]
            tex.add_vectors(delta_tex.clone().multiply_scalar(step), tex_corners[start_index])
            
            # deltaPos *= sliceSpacing
            delta_pos.multiply_scalar(slice_spacing)
            delta_tex.multiply_scalar(slice_spacing)
        else:
            # Handle zero range case: set deltas to zero, pos/tex to start corner
            delta_pos.set(0, 0, 0)
            delta_tex.set(0, 0, 0)
            pos.copy(pos_corners[start_index])
            tex.copy(tex_corners[start_index])

        active_edge = {
            'expired': False,
            'startIndex': start_index,
            'endIndex': end_index,
            'deltaPos': delta_pos,
            'deltaTex': delta_tex,
            'pos': pos,
            'tex': tex,
            'cur': next_edge,
            'prev': 0,
            'next': 0
        }
        
        expirations.push(active_edge, corner_distance[end_index])
        active_edges.append(active_edge)
        return active_edge

    # Initialization
    for i in range(3):
        active_edge = create_edge(max_corner, corner_neighbors[max_corner][i])
        active_edge['prev'] = (i + 2) % 3
        active_edge['next'] = (i + 1) % 3

    slice_distance = max_corner
    
    while slice_distance > min_distance:
        # Process expirations
        while not expirations.is_empty() and expirations.top().priority >= slice_distance:
            edge = expirations.pop().object
            
            if edge['expired']:
                continue
            
            prev_edge = active_edges[edge['prev']]
            next_edge_obj = active_edges[edge['next']]
            
            if edge['endIndex'] != prev_edge['endIndex'] and edge['endIndex'] != next_edge_obj['endIndex']:
                # Split
                edge['expired'] = True
                
                active_edge1 = create_edge(edge['endIndex'], incoming_edges[edge['endIndex']][edge['startIndex']])
                active_edge1['prev'] = edge['prev']
                active_edges[edge['prev']]['next'] = next_edge - 1
                active_edge1['next'] = next_edge
                
                active_edge2 = create_edge(edge['endIndex'], incoming_edges[edge['endIndex']][active_edge1['endIndex']])
                active_edge2['prev'] = next_edge - 2
                active_edge2['next'] = edge['next']
                active_edges[active_edge2['next']]['prev'] = next_edge - 1
                first_edge = next_edge - 1
            else:
                # Merge
                if edge['endIndex'] == prev_edge['endIndex']:
                    prev = prev_edge
                    next = edge
                else:
                    prev = edge
                    next = next_edge_obj
                
                prev['expired'] = True
                next['expired'] = True
                
                active_edge = create_edge(edge['endIndex'], incoming_edges[edge['endIndex']][prev['startIndex']])
                active_edge['prev'] = prev['prev']
                active_edges[active_edge['prev']]['next'] = next_edge - 1
                active_edge['next'] = next['next']
                active_edges[active_edge['next']]['prev'] = next_edge - 1
                first_edge = next_edge - 1

        # Collect vertices
        cur = first_edge
        count = 0
        
        if first_edge < len(active_edges):
            do:
                count += 1
                active_edge = active_edges[cur]
                points.extend(active_edge['pos'].components)
                tex_coords.extend(active_edge['tex'].components)
                
                active_edge['pos'].add(active_edge['deltaPos'])
                active_edge['tex'].add(active_edge['deltaTex'])
                
                cur = active_edge['next']
            while cur != first_edge and cur < len(active_edges)

        # Generate indices
        for i in range(2, count):
            indexes.append(next_index)
            indexes.append(next_index + i - 1)
            indexes.append(next_index + i)
            next_index += 1

        slice_distance -= slice_spacing

    return points, tex_coords, indexes
```

**Parameters:**
*   `sliceSpacing`: The distance between consecutive slices [S1, unofficial].
*   `minDistance`: The minimum slice distance to process [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If `range` (difference between corner distances) is zero, interpolation is skipped, and the edge position/texture is set to the start corner values [S1, unofficial].
*   **Boundary Conditions:** The algorithm assumes valid `incomingEdges` and `cornerNeighbors` structures. If `nextEdge` exceeds 12, `createEdge` returns undefined, which may break the linked list if not handled [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. The algorithm iterates through slices and processes edges, with split/merge operations being constant time per edge [S1, unofficial].
*   **Memory Complexity:** Not explicitly given. Uses a fixed-size active edge list (up to 12 edges) and buffers for output [S1, unofficial].
*   **Strengths:** Optimized for volumetric fire rendering, handles topology changes dynamically [S1, unofficial].
*   **Limitations:** Specific to the VolumetricFire implementation; not a general-purpose isosurface extractor [S1, unofficial].
*   **When to use:** When rendering volumetric fire effects where slice-based contouring is sufficient [S1, unofficial].
*   **When not to use:** For general isosurface extraction requiring robust topology handling across complex volumes [S1, unofficial].

## Background
VolumetricFire Slice Contouring is part of the VolumetricFire library, used for rendering volumetric fire effects [S1, unofficial]. It is a specialized implementation within the broader context of isosurface extraction methods [S1, unofficial].

## Sources
[S1] yomotsu/VolumetricFire, https://github.com/yomotsu/VolumetricFire, fetched 2026-10-27 (unofficial)
