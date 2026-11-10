# Voxel Child Address Calculation
Also called: Voxel address_of

## Overview
Voxel Child Address Calculation computes the linear index of a specific child octant within a compact payload array, given its logical octant index and a bitmask of active children. It solves the problem of mapping sparse octree children to a dense, contiguous memory block, allowing traversal to skip empty octants without allocating storage for them. Unlike [Octree Child Index Computation](1807-octree-child-index-computation.md), which yields the 0-7 octant offset, this method translates that offset into the actual storage position based on the validity mask.

## How it works
**Inputs:**
- `octant`: The logical index of the target child (0–7).
- `valid`: A bitmask where the $i$-th bit is set if child $i$ exists.

**Outputs:**
- `address`: The linear index in the compact child array.

**Data Structures:**
- `valid`: Integer bitmask (8 bits).

**Steps:**
1. Create a mask that isolates all bits preceding the target `octant`.
2. Count the number of set bits in the intersection of the mask and `valid`.
3. Return the count as the address.

**Formula:**
$$ \text{address} = \text{octant} + \text{popcount}(\sim(0xFF \ll \text{octant}) \ \& \ \text{valid}) $$

Where:
- `popcount` counts the number of set bits (population count).
- `~` is bitwise NOT.
- `<<` is bitwise left shift.
- `&` is bitwise AND.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_child_address_calculation(octant: int, valid: int) -> int:
    """
    Calculate the linear address of a child in a compact array.
    
    Args:
        octant: Logical child index (0-7).
        valid: Bitmask of active children.
        
    Returns:
        Linear index in the compact payload array.
    """
    # Create mask: isolate bits preceding the target octant
    # ~0xFF is 0xFFFFFF00 (assuming 32-bit int), shifting left by octant
    # ensures bits 0 to octant-1 are 1, and octant onwards are 0.
    mask = ~(0xFF << octant)
    
    # Count valid children preceding the target
    preceding_valid = popcount(mask & valid)
    
    # The address is the number of preceding valid children
    return preceding_valid

def popcount(n: int) -> int:
    """Count the number of set bits in an integer."""
    count = 0
    while n:
        n &= n - 1
        count += 1
    return count
```

**Edge Cases:**
- If `octant` is not set in `valid`, the result is the insertion point where that child would be if added.
- If `valid` is 0, the result is 0 (though accessing child 0 is invalid).

## Subtopics

### [Prefix Rank for Compact Child Indexing](5129-prefix-rank-for-compact-child-indexing.md)
Maps a child's logical index to its compact position using a rank query on a bit vector. Chosen when storage must be minimized by skipping empty octants.

## Operating characteristics
- **Time Complexity:** $O(1)$ assuming hardware popcount or $O(\log N)$ where $N$ is the number of bits (8 for octants).
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Extremely fast, enables dense packing of sparse data.
- **Limitations:** Requires the children to be stored in octant order within the compact array.

## Background
Originated in implementations of sparse voxel octrees for real-time rendering, such as in the Zazen renderer [S1, unofficial]. It is a fundamental component of hierarchical grid traversal methods that use compact child arrays.

## Sources
[S1] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
