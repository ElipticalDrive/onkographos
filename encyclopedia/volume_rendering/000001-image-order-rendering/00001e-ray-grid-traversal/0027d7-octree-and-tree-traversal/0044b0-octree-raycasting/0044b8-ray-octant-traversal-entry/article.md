# Ray-Octant Traversal (Entry)

Also called: findEntryOctant, findNextOctant

## Overview
Determines the index of the first octant a ray enters within a local 2x2x2 block of an octree, and the index of the subsequent octant the ray traverses. It solves the problem of efficiently mapping ray-parametric intersections to discrete octant indices without branching on every axis comparison. It differs from [Ray-Octree Boundary Intersection](11318-ray-octree-boundary-intersection.md) by operating on parametric t-values relative to a specific octant's bounds rather than global geometric coordinates.

## How it works
**Inputs:**
- `tx0, ty0, tz0`: Ray projection parameters at the start of the current segment.
- `tx1, ty1, tz1`: Ray projection parameters at the end of the current segment.
- `txm, tym, tzm`: Ray projection parameters at the midpoint (or opposite face) of the current octant along each axis. Defined as $txm = 0.5 \times (tx0 + tx1)$, $tym = 0.5 \times (ty0 + ty1)$, $tzm = 0.5 \times (tz0 + tz1)$ [S1, unofficial].

**Outputs:**
- `entryIndex`: Integer 0-7 representing the octant index the ray enters.
- `nextOctant`: Integer 0-7 representing the next octant to visit.

**Data Structures:**
- `octantTable`: A lookup table of size $8 \times 3$. `octantTable[currentOctant][exit]` returns the next octant index. `exit` is 0 for YZ-plane, 1 for XZ-plane, 2 for XY-plane [S1, unofficial].

**Steps:**

1. **Calculate Entry Index:**
   - Initialize `entry = 0`.
   - Determine which axis parameter is largest at the start:
     - If `tx0` is the largest (`tx0 > ty0 && tx0 > tz0`):
       - Ray enters through the YZ-plane (index 0).
       - If `tym < tx0`, set bit 2: `entry |= 4` (Note: Source [S1, unofficial] says `entry |= 2` for bit 2, but standard binary bit 2 is value 4. However, looking at the switch cases in [S1, unofficial], the logic maps specific comparisons to specific bits. Let's follow the source's explicit bit manipulation instructions strictly, even if the bit numbering seems non-standard or 1-based in description. Source says: "set bit 2 (entry |= 2)". This implies bit 1 is value 1, bit 2 is value 2, bit 3 is value 4? No, `entry |= 4` is used elsewhere. Let's look at the pattern.
       - Case 0 (YZ entry): `tx0` max. Checks: `tym < tx0` -> `entry |= 2`. `tzm < tx0` -> `entry |= 1`.
       - Case 1 (XZ entry): `ty0` max. Checks: `txm < ty0` -> `entry |= 4`. `tzm < ty0` -> `entry |= 1`.
       - Case 2 (XY entry): `tz0` max. Checks: `txm < tz0` -> `entry |= 4`. `tym < tz0` -> `entry |= 2`.
       - This implies a specific bit mapping: Bit 0 (val 1) corresponds to Z-axis crossing? Bit 1 (val 2) corresponds to Y-axis crossing? Bit 2 (val 4) corresponds to X-axis crossing?
       - Let's re-read carefully: "If tym < tx0, set bit 2 (entry |= 2)". This is contradictory to standard binary (bit 2 is 4). However, the code `entry |= 2` sets the second bit (value 2). The comment "bit 2" likely refers to the Y-axis (second axis). Similarly "bit 1" refers to Z-axis (third axis? or first?).
       - Let's stick to the exact bitwise operations provided in [S1, unofficial] to ensure implementation correctness.
     - Else if `ty0` is larger than `tz0`:
       - Ray enters through the XZ-plane (index 1).
       - If `txm < ty0`, set `entry |= 4`.
       - If `tzm < ty0`, set `entry |= 1`.
     - Else (`tz0` is largest):
       - Ray enters through the XY-plane (index 2).
       - If `txm < tz0`, set `entry |= 4`.
       - If `tym < tz0`, set `entry |= 2`.
   - Return `entry`.

2. **Calculate Next Octant:**
   - Compare `tx1` and `ty1`.
   - If `tx1 < ty1`: `min = tx1`, `exit = 0` (YZ-plane).
   - Else: `min = ty1`, `exit = 1` (XZ-plane).
   - If `tz1 < min`: `exit = 2` (XY-plane).
   - Return `octantTable[currentOctant][exit]`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_entry_octant(tx0, ty0, tz0, txm, tym, tzm):
    entry = 0
    
    if tx0 > ty0 and tx0 > tz0:
        # Enters through YZ-plane
        if tym < tx0:
            entry |= 2
        if tzm < tx0:
            entry |= 1
    elif ty0 > tz0:
        # Enters through XZ-plane
        if txm < ty0:
            entry |= 4
        if tzm < ty0:
            entry |= 1
    else:
        # Enters through XY-plane
        if txm < tz0:
            entry |= 4
        if tym < tz0:
            entry |= 2
            
    return entry

def find_next_octant(current_octant, tx1, ty1, tz1):
    # Determine exit plane based on exit t-values
    if tx1 < ty1:
        min_t = tx1
        exit_plane = 0 # YZ-plane
    else:
        min_t = ty1
        exit_plane = 1 # XZ-plane
        
    if tz1 < min_t:
        exit_plane = 2 # XY-plane
        
    # Lookup table must be provided externally
    # octant_table[current_octant][exit_plane]
    return octant_table[current_octant][exit_plane]
```

**Parameters:**
- `tx0, ty0, tz0`: Start t-values [S1, unofficial].
- `txm, tym, tzm`: Midpoint t-values [S1, unofficial].
- `tx1, ty1, tz1`: End t-values [S1, unofficial].
- `octantTable`: $8 \times 3$ integer array [S1, unofficial].

**Edge Cases:**
- Ray origin exactly on a plane boundary: The comparisons (`>`, `<`) determine the entry. Exact equality may lead to arbitrary but consistent selection depending on implementation.
- Degenerate ray (zero length): `tx0 == tx1`, etc. The logic still holds but `txm == tx0`.

## Operating characteristics
- **Time Complexity:** $O(1)$ for both entry and next octant calculation.
- **Memory Complexity:** $O(1)$ for the lookup table (24 integers).
- **Strengths:** Branch-minimized entry calculation; fast traversal step.
- **Limitations:** Requires pre-computed or maintained lookup table; specific to pointer-based octree traversal where children are accessed via flags.

## Background
Originated in the `sparse-octree` library by vanruesc [S1, unofficial]. It is used in efficient raycasting through sparse octrees where nodes are stored in arrays and accessed via bit flags.

## Sources
[S1] sparse-octree module, https://github.com/vanruesc/sparse-octree, fetched 2026-10-27 (unofficial)
