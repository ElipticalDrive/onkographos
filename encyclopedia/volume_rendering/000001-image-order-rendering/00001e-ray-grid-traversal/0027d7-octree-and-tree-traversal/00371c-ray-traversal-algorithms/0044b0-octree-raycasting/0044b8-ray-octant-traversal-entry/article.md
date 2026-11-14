# Ray-Octant Traversal (Entry)

Also called: findEntryOctant, findNextOctant

## Overview
Determines the index of the first octant a ray enters within a local 2×2×2 block of an octree, and the index of the subsequent octant the ray traverses [S1, unofficial]. It maps ray-parametric intersections to discrete octant indices using axis-aligned comparisons [S1, unofficial].

## How it works
**Inputs:**
- `tx0, ty0, tz0`: Ray projection parameters at the start of the current segment [S1, unofficial].
- `tx1, ty1, tz1`: Ray projection parameters at the end of the current segment [S1, unofficial].
- `txm, tym, tzm`: Ray projection parameters at the midpoint of the current octant along each axis [S1, unofficial].

**Outputs:**
- `entryIndex`: Integer 0–7 representing the octant index the ray enters [S1, unofficial].
- `nextOctant`: Integer 0–7 representing the next octant to visit [S1, unofficial].

**Data Structures:**
- `octantTable`: An $8 \times 3$ lookup table where `octantTable[currentOctant][exit]` returns the next octant index [S1, unofficial]. `exit` is 0 for the YZ-plane, 1 for the XZ-plane, and 2 for the XY-plane [S1, unofficial].

**Formulas:**
- Midpoint parameters: $txm = 0.5 \times (tx0 + tx1)$, $tym = 0.5 \times (ty0 + ty1)$, $tzm = 0.5 \times (tz0 + tz1)$ [S1, unofficial].

**Steps:**
1. **Calculate Entry Index:**
   - Initialize `entry = 0`.
   - Compare start parameters to find the largest:
     - If `tx0` is the largest (`tx0 > ty0` and `tx0 > tz0`): the ray enters through the YZ-plane (index 0). If `tym < tx0`, set bit 1: `entry |= 2`. If `tzm < tx0`, set bit 0: `entry |= 1` [S1, unofficial].
     - Else if `ty0` is larger than `tz0`: the ray enters through the XZ-plane (index 1). If `txm < ty0`, set bit 2: `entry |= 4`. If `tzm < ty0`, set bit 0: `entry |= 1` [S1, unofficial].
     - Else (`tz0` is largest): the ray enters through the XY-plane (index 2). If `txm < tz0`, set bit 2: `entry |= 4`. If `tym < tz0`, set bit 1: `entry |= 2` [S1, unofficial].
   - Return `entry` [S1, unofficial].

2. **Calculate Next Octant:**
   - Compare exit parameters `tx1` and `ty1` [S1, unofficial].
   - If `tx1 < ty1`, set `min_t = tx1` and `exit = 0` (YZ-plane) [S1, unofficial].
   - Else, set `min_t = ty1` and `exit = 1` (XZ-plane) [S1, unofficial].
   - If `tz1 < min_t`, set `exit = 2` (XY-plane) [S1, unofficial].
   - Return `octantTable[currentOctant][exit]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_entry_octant(tx0, ty0, tz0, txm, tym, tzm):
    entry = 0
    if tx0 > ty0 and tx0 > tz0:
        if tym < tx0:
            entry |= 2
        if tzm < tx0:
            entry |= 1
    elif ty0 > tz0:
        if txm < ty0:
            entry |= 4
        if tzm < ty0:
            entry |= 1
    else:
        if txm < tz0:
            entry |= 4
        if tym < tz0:
            entry |= 2
    return entry

def find_next_octant(current_octant, tx1, ty1, tz1, octant_table):
    if tx1 < ty1:
        min_t = tx1
        exit_plane = 0
    else:
        min_t = ty1
        exit_plane = 1
    if tz1 < min_t:
        exit_plane = 2
    return octant_table[current_octant][exit_plane]

# Lookup table mapping (current_octant, exit_plane) -> next_octant
OCTANT_TABLE = [
    [1, 2, 4], [3, 6, 5], [9, 10, 12], [11, 14, 13],
    [17, 18, 20], [19, 22, 21], [25, 26, 28], [27, 30, 29]
]
```

**Parameters:**
- `tx0, ty0, tz0`: Start t-values [S1, unofficial].
- `txm, tym, tzm`: Midpoint t-values [S1, unofficial].
- `tx1, ty1, tz1`: End t-values [S1, unofficial].
- `octantTable`: $8 \times 3$ integer array [S1, unofficial].

**Edge Cases:**
- Ray origin exactly on a plane boundary: The strict comparisons (`>`, `<`) determine the entry [S1, unofficial]. Exact equality may lead to arbitrary but consistent selection depending on implementation [G].
- Degenerate ray (zero length): `tx0 == tx1`, etc. The logic still holds but `txm == tx0` [G].
- Outside-in rays: The algorithm assumes `tx0, ty0, tz0` represent parametric distances to the current node's faces; negative directions are handled correctly by the midpoint comparisons [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ for both entry and next octant calculation [S1, unofficial].
- **Memory Complexity:** $O(1)$ for the lookup table (24 integers) [S1, unofficial].
- **Strengths:** Branch-minimized entry calculation; fast traversal step [S1, unofficial].
- **Limitations:** Requires pre-computed or maintained lookup table; specific to pointer-based octree traversal where children are accessed via flags [S1, unofficial].

## Background
Used in efficient raycasting through sparse octrees where nodes are stored in arrays and accessed via bit flags [S1, unofficial].

## Sources
[S1] sparse-octree module, https://github.com/vanruesc/sparse-octree, fetched 2026-10-27 (unofficial)
