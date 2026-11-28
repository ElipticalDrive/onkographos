# Span-based Surface Extraction

Also called: Connected Surface Construction

## Overview
Span-based Surface Extraction is a method for extracting connected surface topology from voxelized particle data by identifying depth intervals (spans) in voxel columns and linking overlapping spans in neighboring columns [S1]. It builds on [Connectivity Generation via Recursive Traversal](7454-connectivity-generation-via-recursive-traversal.md) by converting minimal edge connectivity into a continuous surface representation suitable for filtering and rendering [S1]. Unlike dual contouring methods that rely on quadric error metrics and vertex optimization, this approach operates directly on compressed depth lists [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A compressed list of depth values marking entry and exit points of the volume along each voxel column. These depths define "spans" (intervals between successive entry and exit depths) [S1].
*   **Output:** A set of surface connection links stored in parallel to the surface depth values, enabling traversal of the continuous surface [S1].

**Data Structures:**
*   **Span:** Defined by an entry depth ($d_{in}$) and an exit depth ($d_{out}$) within a specific voxel column [S1].
*   **Link Storage:** For each pixel (voxel column), surface connection links to its four direct neighbors are packed into one 32-bit machine word and stored in parallel to the surface depth values [S1].
*   **Special Link Values:** Two special integer values are reserved to indicate links to subsequent or preceding entries/exits within the same column [S1].

**Algorithm Steps:**
1.  **Identify Spans:** Iterate through each voxel column to identify all spans formed by pairs of successive entry and exit depths [S1].
2.  **Find Neighbor Spans:** For each `current_span` in a column, identify relevant overlapping spans in the four direct neighboring columns [S1].
    *   `front_nb`: The first neighbor span where the neighbor's exit depth is greater than or equal to the current span's entry depth ($d_{nb, out} \ge d_{curr, in}$) [S1].
    *   `back_nb`: The last neighbor span where the neighbor's entry depth is less than or equal to the current span's exit depth ($d_{nb, in} \le d_{curr, out}$) [S1].
3.  **Link Entries (Front Connection):** Check if `front_nb` overlaps with `current_span` ($d_{nb, in} \le d_{curr, out}$) [S1].
    *   If `front_nb` is not connected to the previous span in the neighbor column ($d_{nb, in} > d_{prev, out}$), link `current_span.entry` to `front_nb.entry` [S1].
    *   Otherwise (the neighbor span is the same as the previous one, forming a bubble wall), link `current_span.entry` to `previous_span.exit` of the neighbor column [S1].
4.  **Link Exits (Back Connection):** Check if `back_nb` is not connected to the next span in the neighbor column ($d_{next, in} > d_{nb, out}$) [S1].
    *   If so, link `current_span.exit` to `back_nb.exit` [S1].
    *   Otherwise (the neighbor recurs in the next span, forming a bubble wall), link `current_span.exit` to `next_span.entry` of the neighbor column [S1].
5.  **Handle Non-Overlapping Cases:** If there is no overlapping neighbor span, link the entry and exit depths of the current span to each other [S1].
6.  **Handle Internal Bubbles:** Inner exit depths are connected to the subsequent entry depth in their own column, and inner entry depths are connected to the preceding exit depth [S1]. This accounts for bubbles inside the volume [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def span_based_extraction(columns):
    # columns: list of lists of (entry, exit) tuples per voxel column
    # Returns: list of link words per span per column
    all_links = []
    for col in columns:
        col_links = []
        previous_exit = None
        for i, (current_entry, current_exit) in enumerate(col):
            next_entry = col[i+1][0] if i < len(col) - 1 else None
            
            # front_nb = first_neighbor_span(where .exit >= current_span.entry)
            front_nb = None
            for nb_span in get_neighbor_spans(col, i, 'front'):
                if nb_span.exit >= current_entry:
                    front_nb = nb_span
                    break
            
            # back_nb = last_neighbor_span(where .entry <= current_span.exit)
            back_nb = None
            for nb_span in get_neighbor_spans(col, i, 'back'):
                if nb_span.entry <= current_exit:
                    back_nb = nb_span
            
            link_word = 0
            
            if front_nb is not None:
                if front_nb.entry <= current_exit:
                    if previous_exit is None or front_nb.entry > previous_exit:
                        link_word |= pack_link(current_entry, front_nb.entry)
                    else:
                        link_word |= pack_link(current_entry, previous_exit)
            
            if back_nb is not None:
                if next_entry is None or next_entry > back_nb.exit:
                    link_word |= pack_link(current_exit, back_nb.exit)
                else:
                    link_word |= pack_link(current_exit, next_entry)
            else:
                link_word |= pack_link(current_entry, current_exit)
            
            col_links.append(link_word)
            previous_exit = current_exit
        all_links.append(col_links)
    return all_links

def get_neighbor_spans(current_col, current_idx, direction):
    # Returns list of spans in the specified neighbor column.
    # Returns empty list if neighbor is out of bounds.
    return []

def pack_link(entry, exit):
    # Packs two depth values into a link word.
    return (entry << 16) | exit
```

**Parameters and Edge Cases:**
*   **Bubbles:** The algorithm explicitly handles internal bubbles by linking inner exit to subsequent entry and inner entry to preceding exit within the same column [S1].
*   **Boundary Spans:** If a span has no overlapping neighbor, entry and exit are linked to each other, effectively closing the surface at boundaries [S1].
*   **State Change Detection:** Ray casting or state changes between inside and outside can be detected by checking the LSB of the layer index when entering a new column [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of spans and neighbors [S1].
*   **Memory Complexity:** Highly memory-efficient. Links are packed into 32-bit words per pixel, stored in parallel with depth values [S1].
*   **Strengths:** Suitable for large-scale particle data due to low memory footprint. Enables easy application of smoothing filters (e.g., blurring, curvature flow) via surface traversal [S1].
*   **Limitations:** Relies on compressed depth lists; performance depends on the efficiency of the preceding voxelization and compression steps [S1].

## Background
The method was introduced in the context of memory-efficient on-the-fly voxelization of particle data [S1]. It addresses the challenge of extracting connected surfaces from compressed voxel data without storing explicit mesh structures, enabling efficient filtering and rendering of complex particle systems [S1].

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
