# Morton Order Indexing for Out-of-Core Rendering

Also called: Morton indexing, Z-order curve traversal

## Overview
Morton Order Indexing for Out-of-Core Rendering is a data organization scheme that sorts voxel indices based on spatial locality using a recursive Z-shaped curve traversal, reducing disk seek operations by storing similar color/function values contiguously [S1, unofficial]. It differs from standard Morton Code Encoding by integrating the spatial index with a histogram-based look-up table to manage variable-sized voxelsets on disk, enabling efficient out-of-core retrieval [S1, unofficial].

## How it works
**Inputs:**
- A 3D grid of voxels with dimensions $2^n \times 2^n \times 2^n$ [S1, unofficial].
- Voxel data including position (3×4-byte values) and normalized gradient (3 bytes) [S1, unofficial].
- Background images used to identify noise colors [S1, unofficial].

**Outputs:**
- A disk file containing voxelsets stored in Morton order [S1, unofficial].
- A look-up table $l$ mapping Morton indices to file offsets [S1, unofficial].

**Data Structures:**
- **Histogram $h$:** An array where $h(r,g,b)$ stores the number of occurrences (voxels) for a specific color/value $(r,g,b)$ [S1, unofficial].
- **Look-up Table $l$:** Stores starting offsets $s_V$ for each voxelset $V$ [S1, unofficial].

**Steps:**

1. **Morton Index Calculation:** Compute the Morton index $M(r,g,b)$ for each voxel's color/value using bit-interleaving of its components [S1, unofficial].
2. **Histogram Generation (Pass 1):** Iterate through all voxels and increment the histogram entry for each color/value: $h(r,g,b) \leftarrow h(r,g,b) + 1$ [S1, unofficial].
3. **Noise Cleaning:** Identify colors present in the background images and set $h(r,g,b) = 0$ for all such colors to exclude background noise from storage [S1, unofficial].
4. **Look-up Table Construction:** Calculate the starting offset $s_V$ for each unique Morton index $M$ using the cumulative sum of the histogram:
   $$ s_V = \sum_{i=0}^{M(r,g,b)-1} h(M^{-1}(i)) $$
   where $M^{-1}(i)$ maps the integer index $i$ back to its corresponding color/value tuple, and $h$ is the cleaned histogram [S1, unofficial]. Store these offsets in the look-up table $l$ [S1, unofficial].
5. **Data Encoding (Pass 2):** Iterate through the voxels in Morton order (sorted by $M(r,g,b)$) and write each voxel's data to the disk file at the offset $s_V$ retrieved from the look-up table $l$ [S1, unofficial].
6. **Data Retrieval:** To render, calculate the Morton index for selected colors, sort these indices in ascending order, and access the file consecutively using the offsets in $l$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def morton_index(x, y, z):
    """Interleave bits of x, y, z to form a Morton code."""
    code = 0
    for i in range(16):
        code |= (x & (1 << i)) << (2 * i)
        code |= (y & (1 << i)) << (2 * i + 1)
        code |= (z & (1 << i)) << (2 * i + 2)
    return code

def preprocess(voxels, background_colors):
    histogram = {}
    for v in voxels:
        c = v.color
        histogram[c] = histogram.get(c, 0) + 1
    for bg in background_colors:
        histogram.pop(bg, None)
    sorted_colors = sorted(histogram.keys(), key=lambda c: morton_index(*c))
    lut = {}
    offset = 0
    for c in sorted_colors:
        lut[morton_index(*c)] = offset
        offset += histogram[c] * 15
    return lut, histogram, sorted_colors

def encode_to_disk(voxels, lut, filepath):
    with open(filepath, 'wb') as f:
        sorted_voxels = sorted(voxels, key=lambda v: morton_index(*v.color))
        current_offset = 0
        for v in sorted_voxels:
            m = morton_index(*v.color)
            if lut[m] != current_offset:
                current_offset = lut[m]
            f.seek(current_offset)
            f.write(v.encode())
            current_offset += 15

def retrieve(selected_colors, lut, histogram, filepath):
    indexed_colors = sorted([(morton_index(c), c) for c in selected_colors if c in histogram])
    results = []
    with open(filepath, 'rb') as f:
        for m, c in indexed_colors:
            offset = lut[m]
            count = histogram[c]
            f.seek(offset)
            results.append(f.read(count * 15))
    return results
```

**Parameters:**
- Voxel data size: 15 bytes per voxel (3×4 bytes position + 3 bytes normal) [S1, unofficial].
- Grid dimensions: $2^n \times 2^n \times 2^n$ [S1, unofficial].

**Edge Cases:**
- Background noise: Must be cleaned from the histogram to avoid storing unnecessary data [S1, unofficial].
- Degenerate input: If a color has zero occurrences after cleaning, it is excluded from the look-up table and file [S1, unofficial].

## Subtopics

### [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md)
The foundational bit-interleaving method used to compute the indices.

### [Morton Index Calculation via Byte-Lookup Table](2384-morton-index-calculation-via-byte-lookup-table.md)
An optimization for computing Morton codes, potentially used in Step 1.

### [Morton Code Generation for Octrees](4673-morton-code-generation-for-octrees.md)
Related to spatial indexing but for sparse structures.

### [Morton Code Spatial Sorting](4818-morton-code-spatial-sorting.md)
The sorting step used to organize voxels before writing.

## Operating characteristics
- **Time Complexity:** Pre-processing requires $O(N)$ to build the histogram and $O(N \log N)$ or $O(N)$ to sort voxels by Morton index, where $N$ is the number of voxels [S1, unofficial]. Retrieval is $O(K \log K)$ for sorting $K$ selected colors, followed by $O(1)$ seek per voxelset [S1, unofficial].
- **Memory Complexity:** Requires storing the histogram and look-up table in memory [S1, unofficial].
- **Performance:** Reduces disk seek operations by accessing datasets consecutively, allowing real-time reading from disk [S1, unofficial]. If $n$ datasets are located next to each other in a file, a single disk-read operation can fetch them, reducing unnecessary disk rotations [S1, unofficial].
- **Strengths:** Exploits spatial locality for similar colors; reduces unnecessary disk rotations [S1, unofficial].
- **Limitations:** Requires two-pass pre-processing; variable-sized voxelsets require careful offset management [S1, unofficial].

## Background
This method was proposed in "Interactive Visualization of Very Large Datasets Using an Out-of-Core Point-based Approach" [S1, unofficial]. It addresses the challenge of rendering large volumetric datasets that do not fit in main memory by organizing data on disk to minimize I/O latency [S1, unofficial].

## Sources
[S1] Interactive Visualization of Very Large Datasets Using an Out-of-Core Point-based Approach, https://core.ac.uk/download/484268453.pdf, fetched 2026-10-27 (unofficial)
