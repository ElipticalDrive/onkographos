# Voxel Child Address Calculation

Also called: Voxel address_of

## Overview
Calculates the linear index (address) of a specific child octant within a compact block of valid children, based on a validity bitmask [S1, unofficial]. It maps a logical octant index (0–7) to its physical position in a dense array that only stores active children [S1, unofficial].

## How it works
**Inputs:**
- `valid`: Bitmask where bit $i$ is 1 if child $i$ is active, 0 otherwise.
- `octant`: Target child index (0–7).

**Outputs:**
- `address`: 0-based index of the target child within the compact array of valid children.

**Data Structures:**
- `valid`: Integer type (e.g., `uint8_t`), 8 bits.

**Steps:**
1. Create a mask that isolates all bits lower than the target `octant`:
   $$ \text{mask} = \sim(0xFF \ll \text{octant}) $$
   This isolates the bits corresponding to children with indices lower than `octant` [S1, unofficial].
2. Compute the bitwise AND of `valid` and `mask` to isolate valid children preceding the target [S1, unofficial].
3. Count the number of set bits (population count) in the result to determine the offset [S1, unofficial].
   $$ \text{address} = \text{popcount}(\text{valid} \ \& \ \text{mask}) $$

**Parameters:**
- `octant`: Integer, 0–7.

**Edge Cases and Pitfalls:**
- The calculation assumes children are stored in order of their octant index, but only valid children are present in the block; the popcount determines the offset [S1, unofficial].
- If `octant` is not valid (bit is 0 in `valid`), the result is the index where it *would* be if it were valid [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_child_address(valid: int, octant: int) -> int:
    # Create mask: isolate bits lower than octant
    # ~0xFF inverts all bits; shifting left moves the zero-bits to the right
    # Masking with 0xFF ensures 8-bit wrap-around for languages with infinite precision integers
    mask = (~0xFF << octant) & 0xFF
    # Count valid children preceding the target
    return bin(valid & mask).count('1')
```

## Subtopics

### Prefix Rank for Compact Child Indexing
Counts the number of active children (bits set to 1) in a bitmask that precede a given `child_index` [S2, unofficial]. Used to map a child index to a compact payload index in sparse octrees [S2, unofficial]. Supports wider nodes (e.g., 0–63 for 64-way) [S2, unofficial].
- **Definition:** Maps a logical child index to its rank among valid children [S2, unofficial].
- **Steps:** If `child_index <= 0`, return 0. Create `lower_mask = (1 << child_index) - 1`. Return `popcount(mask & lower_mask)` [S2, unofficial].
- **Parameters:** `mask`: Bitmask representing occupied children. `child_index`: Target child index (0–7 for 8-way, 0–63 for wide) [S2, unofficial].
- **Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prefix_rank(mask: int, child_index: int) -> int:
    if child_index <= 0:
        return 0
    lower_mask = (1 << child_index) - 1
    return bin(mask & lower_mask).count('1')
```

## Operating characteristics
- **Time Complexity:** $O(1)$ assuming fixed-width integers and hardware popcount [G].
- **Memory Complexity:** $O(1)$ [G].
- **Strengths:** Extremely fast, branchless (if popcount is hardware-supported), essential for compact memory layouts [G].
- **Limitations:** Only works for direct child indexing within a single node [G].
- **When to use:** When storing children in a compact array and needing to find the index of a specific child [G].
- **When not to use:** When traversing the tree or locating a node in the tree structure [G].

## Background
Originated in implementations of sparse voxel octrees for efficient memory storage and ray traversal [G]. Popularized by libraries like [zazen](https://github.com/gabrielmuller/zazen) and [sparse-voxels](https://github.com/hectorpiteau/sparse-voxels) for GPU-based volume rendering [S1, S2, unofficial].

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S2] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
