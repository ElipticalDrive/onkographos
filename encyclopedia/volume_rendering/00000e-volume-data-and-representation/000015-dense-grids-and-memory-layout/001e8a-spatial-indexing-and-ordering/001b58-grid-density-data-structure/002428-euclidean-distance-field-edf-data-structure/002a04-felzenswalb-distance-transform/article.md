# Felzenswalb Distance Transform

Also called: Euclidean Distance Transform (Felzenswalb et al.), Exact Euclidean distance field, Felzenswalb et al. distance transform

## Overview
The Felzenswalb distance transform computes the exact Euclidean distance field from a binary volume in three passes, eliminating propagation and numerical errors inherent in approximate methods [S1, unofficial]. It is distinguished from the Satherley and Jones method by its exactness, parallelizability, and significantly higher computational efficiency, though it requires greater memory consumption [S1, unofficial].

## How it works
The method computes the minimum Euclidean distance from every voxel in a volume to the nearest foreground voxel (value 1) in a binary input volume [S1, unofficial].

**Inputs and Outputs:**
- Input: A binary volume $V$ of dimensions $D_x \times D_y \times D_z$, where $V(x,y,z) \in \{0, 1\}$ [S1, unofficial].
- Output: A scalar distance field $D$ of the same dimensions, where $D(x,y,z)$ is the Euclidean distance to the nearest voxel with value 1 [S1, unofficial].

**Data Structures:**
- The input volume may be padded or subdivided into meta-cells for processing; for example, an $891 \times 951 \times 601$ binary volume is embedded in a $1025 \times 1025 \times 1025$ zero-filled volume and subdivided into 8 meta-cells of size $513 \times 513 \times 513$ [S1, unofficial].
- The distance field $D$ stores floating-point values [G].

**Algorithm Steps:**
The algorithm proceeds in three separable passes along orthogonal axes [S1, unofficial]. Each pass computes a 1D distance transform along one dimension, using the results of the previous pass as input [G].
1. **Initialization:** Initialize the distance field $D$ with infinity for background voxels and 0 for foreground voxels [G].
2. **Pass 1 (X-axis):** For each voxel $(x, y, z)$, compute the 1D Euclidean distance transform along the x-axis using the current values in $D$ [G].
3. **Pass 2 (Y-axis):** For each voxel $(x, y, z)$, compute the 1D Euclidean distance transform along the y-axis using the updated values from Pass 1 [G].
4. **Pass 3 (Z-axis):** For each voxel $(x, y, z)$, compute the 1D Euclidean distance transform along the z-axis using the updated values from Pass 2 [G].

**1D Transform Formula:**
For a 1D array of values $f[i]$, the distance transform $g[i]$ is computed as:
$$ g[i] = \min_{j} \left( \sqrt{(i-j)^2 + f[j]^2} \right) $$
This is computed in linear time using a stack-based lower envelope of parabolas [G].

**Parameters:**
- Grid spacing: Assumed to be 1 unit per voxel unless specified otherwise [G].
- Meta-cell size: $513 \times 513 \times 513$ was used in the reference implementation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Memory Consumption:** The method requires more memory than the Satherley and Jones method [S1, unofficial].
- **Propagation Errors:** Unlike approximate methods, this method does not suffer from propagation errors [S1, unofficial].
- **Numerical Errors:** The method is designed to avoid numerical errors [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def felzenswalb_distance_transform(binary_volume):
    """
    Computes the exact Euclidean distance field from a binary volume.
    binary_volume: 3D list/array of 0s and 1s.
    Returns: 3D array of distances.
    """
    height = len(binary_volume)
    width = len(binary_volume[0])
    depth = len(binary_volume[0][0])
    
    # Initialize distance field: 0 for foreground, inf for background
    dist = [[[float('inf') if binary_volume[z][y][x] == 0 else 0.0
              for x in range(width)] for y in range(height)] for z in range(depth)]
    
    # Pass 1: X-axis
    for z in range(depth):
        for y in range(height):
            line = [dist[z][y][x] for x in range(width)]
            transformed = _compute_1d_transform(line)
            for x in range(width):
                dist[z][y][x] = transformed[x]
                
    # Pass 2: Y-axis
    for z in range(depth):
        for x in range(width):
            line = [dist[z][y][x] for y in range(height)]
            transformed = _compute_1d_transform(line)
            for y in range(height):
                dist[z][y][x] = transformed[y]
                
    # Pass 3: Z-axis
    for y in range(height):
        for x in range(width):
            line = [dist[z][y][x] for z in range(depth)]
            transformed = _compute_1d_transform(line)
            for z in range(depth):
                dist[z][y][x] = transformed[z]
                
    return dist

def _compute_1d_transform(line):
    """
    Computes the 1D Euclidean distance transform for a line of values.
    line: 1D array where background is infinity and foreground is 0.
    Returns: 1D array of Euclidean distances.
    """
    n = len(line)
    v = [0] * n
    z = [0.0] * (n + 1)
    k = 0
    v[0] = 0
    z[0] = -float('inf')
    z[1] = float('inf')
    
    for q in range(1, n):
        if line[q] == 0:
            k += 1
            v[k] = q
            s = (q * q + line[q] * line[q] - v[k-1] * v[k-1] - line[v[k-1]] * line[v[k-1]]) / (2.0 * (q - v[k-1]))
            while s <= z[k-1]:
                k -= 1
                s = (q * q + line[q] * line[q] - v[k-1] * v[k-1] - line[v[k-1]] * line[v[k-1]]) / (2.0 * (q - v[k-1]))
            k += 1
            v[k] = q
            z[k-1] = s
            z[k] = float('inf')
            
    result = [0.0] * n
    k = 0
    for q in range(n):
        while z[k+1] < q:
            k += 1
        result[q] = (q - v[k]) ** 2 + line[v[k]] ** 2
        
    return [math.sqrt(d) for d in result]
```

## Subtopics

### Safety Margin Halos
A technique to depict safety margins around structures at risk (e.g., pathologic lymph nodes) as special halos on the current slice to prevent damage during surgery [S2, book]. It involves performing an Euclidean distance transform for all structures at risk, overlaying the distance information on the slice image, and rejecting color-coding to avoid irrelevant information [S2, book]. Instead, important distance thresholds (2mm and 5mm) are depicted as halos by drawing isolines [S2, book]. A pitfall is that in the slice view, left and right are swapped because the viewing direction is from bottom to top [S2, book].

## Operating characteristics
- **Time Complexity:** The algorithm runs in three passes, each of which is linear in the size of the volume [S1, G, unofficial]. The 1D transform is $O(N)$ for a line of length $N$ [G]. Thus, the total time complexity is $O(D_x D_y D_z)$ [G].
- **Memory Complexity:** The method requires more memory than the Satherley and Jones method [S1, unofficial].
- **Measured Performance:** For a binary volume of dimensions $891 \times 951 \times 601$ (embedded in $1025 \times 1025 \times 1025$), the processing time was 722 seconds with Satherley and Jones (2001), 270 seconds with Felzenswalb et al. (2012) serial, and 47 seconds with Felzenswalb et al. (2012) parallelized (8 threads) [S1, unofficial]. This represents a ~93% reduction in processing time compared to Satherley and Jones [S1, unofficial].
- **Strengths:** Exact distances, no propagation errors, parallelizable, significantly faster than approximate methods [S1, unofficial].
- **Limitations:** Higher memory consumption [S1, unofficial].
- **When to use:** When exact Euclidean distances are required and computational efficiency is important, especially in parallel environments [S1, unofficial].

## Background
The Felzenswalb distance transform was introduced by Felzenswalb et al. in 2012 [S1, unofficial]. It is used to generate the scalar distance field required for isosurface extraction [S1, unofficial]. The method is an improvement over the Satherley and Jones method (2001), which uses 8 passes, is not parallelizable, and produces approximations with potential errors [S1, unofficial].

## Sources
[S1] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
[S2] Illustrative Focus+Context Approaches in Interactive Volume Visualization, https://doi.org/10.4230/dfu.sciviz.2010.136, fetched 2026-10-26 (book)
