# Morton Code Sorting for Brick Layout

Also called: Brick Building (Morton Code Sorting), Morton Code Sorting

## Overview
Morton Code Sorting for Brick Layout is a method for organizing sub-bricks into a global buffer to maximize spatial coherence and cache efficiency during volume rendering. It solves the problem of scattered memory access patterns by sorting sub-bricks according to their Z-order (Morton) codes within hierarchical groups. It sets itself apart from its siblings by operating iteratively at each level of the brick hierarchy, using enumeration sort to guarantee that the global buffer remains compact and spatially coherent as construction proceeds [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of sub-bricks belonging to a parent brick, each with a known 3D index within the parent.
- Output: A sorted list of sub-bricks placed into a global output buffer, ordered by their Morton codes.

**Data Structures:**
- **Sub-brick:** Contains its local 3D index $(x, y, z)$ within the parent brick.
- **Global Buffer:** A contiguous array of brick pointers or data blocks.
- **Thread Mapping:** A 1:1 mapping between threads and sub-bricks within a group.

**Parameters:**
- Maximum number of sub-bricks per parent: 64 [S1, unofficial].

**Steps:**
1. **Calculate Morton Codes:** For each sub-brick, compute its Morton code based on its 3D index $(x, y, z)$ within the parent brick. This involves interleaving the bits of the coordinates.
2. **Enumerate Sort:** Perform an enumeration sort within the group of sub-bricks. Since the maximum number of sub-bricks is 64, this is efficient. Each thread calculates the sorted index of one brick by counting how many other bricks in the group have a smaller Morton code.
3. **Place in Buffer:** Place each sorted sub-brick into the global buffer at the calculated sorted index, relative to the group's local order. This ensures the global buffer remains compact and spatially coherent.
4. **Iterate:** Repeat steps 1-3 for each level of the brick hierarchy, ensuring that the entire buffer of bricks is sorted as construction proceeds one level at a time [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def morton_sort_bricks(sub_bricks, global_buffer, group_offset):
    """
    Sorts sub-bricks within a group by Morton code and places them in the global buffer.
    
    Args:
        sub_bricks: List of sub-brick objects, each with a 3D index (x, y, z).
        global_buffer: The global output buffer for bricks.
        group_offset: The starting index in the global buffer for this group.
        
    Returns:
        None (modifies global_buffer in place)
    """
    n = len(sub_bricks)
    # Calculate Morton codes for all sub-bricks in the group
    morton_codes = []
    for brick in sub_bricks:
        # Interleave bits of x, y, z to form Morton code
        code = interleave_bits(brick.x, brick.y, brick.z)
        morton_codes.append(code)
    
    # Enumeration sort: calculate rank for each brick
    ranks = [0] * n
    for i in range(n):
        rank = 0
        for j in range(n):
            if morton_codes[j] < morton_codes[i]:
                rank += 1
            # Handle ties if necessary (e.g., stable sort or index comparison)
            elif morton_codes[j] == morton_codes[i] and j < i:
                rank += 1
        ranks[i] = rank
    
    # Place sorted sub-bricks into the global buffer
    for i in range(n):
        sorted_index = group_offset + ranks[i]
        global_buffer[sorted_index] = sub_bricks[i]

def interleave_bits(x, y, z):
    """
    Interleaves the bits of x, y, and z to form a Morton code.
    Assumes 32-bit integers for simplicity.
    """
    # This is a standard bit-interleaving function.
    # Implementation details omitted for brevity, but must be self-contained.
    # Example: 
    # x = 0b101, y = 0b011, z = 0b110
    # Result: 0b100111101011 (binary representation of interleaved bits)
    # A common implementation uses masks and shifts.
    m = 0
    for i in range(10): # Assuming 10 bits per coordinate for 1024^3 space
        m |= ((x >> i) & 1) << (3 * i)
        m |= ((y >> i) & 1) << (3 * i + 1)
        m |= ((z >> i) & 1) << (3 * i + 2)
    return m
```

**Edge Cases and Pitfalls:**
- **Ties in Morton Codes:** If two sub-bricks have the same Morton code (e.g., same position), a stable sort or secondary key (like original index) is needed to ensure deterministic placement [G].
- **Bit Width:** Ensure the Morton code calculation handles the maximum coordinate range without overflow [G].
- **Group Size:** The method assumes a maximum of 64 sub-bricks per group for efficient enumeration sort [S1, unofficial].

## Subtopics

### Bricking and Z-Order Layout
Bricking and Z-Order Layout is a variant that orders vertices into a fixed number of small cubes (bricking) or arranges them using a Lebesgue space-filling curve (Z-order) for better sequential access [S2]. It is generated by creating an out-of-core octree of vertices and traversing them in-order [S2]. Compared to spatial sorting, it provides more contiguous mesh portions on average but has a much larger front width and span [S2]. It is suitable for meshes with a dominant principal direction but less effective for shapes like spheres [S2].

## Operating characteristics
**Time Complexity:**
- Sorting within each group: $O(N^2)$ for enumeration sort, where $N$ is the number of sub-bricks per group (max 64) [S1, unofficial].
- Overall: Linear with respect to the number of bricks, as sorting is performed locally per group [S1, unofficial].

**Memory Complexity:**
- $O(B)$, where $B$ is the number of bricks, for the global buffer [S1, unofficial].

**Performance:**
- Improves cache coherency during rendering by ensuring nearby bricks are adjacent in memory [S1, unofficial].
- Effective for making use of cache during rendering, as nearby threads access adjacent bricks [S1, unofficial].

**Strengths:**
- Guarantees spatial coherence in the global buffer [S1, unofficial].
- Simple to implement with enumeration sort for small group sizes [S1, unofficial].

**Limitations:**
- Limited to a maximum of 64 sub-bricks per group for efficient sorting [S1, unofficial].
- May not be optimal for meshes without a dominant principal direction [S2].

**When to Use:**
- When spatial coherence and cache efficiency are critical for volume rendering performance [S1, unofficial].
- When the brick hierarchy has a manageable number of sub-bricks per parent [S1, unofficial].

**When Not to Use:**
- For meshes with complex shapes like spheres, where other layouts may be more effective [S2].
- When the number of sub-bricks per group exceeds 64, making enumeration sort inefficient [S1, unofficial].

## Background
Morton Code Sorting for Brick Layout was introduced in the context of real-time rendering and construction of signed distance fields [S1, unofficial]. It builds on the concept of Z-order encoding for spatial data organization [S1, unofficial]. The method is designed to optimize memory access patterns in volume rendering by leveraging the spatial locality of bricks [S1, unofficial].

## Sources
[S1] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
[S2] Streaming Simplification of Tetrahedral Meshes, https://doi.org/10.1109/tvcg.2007.21, fetched 2026-11-04
