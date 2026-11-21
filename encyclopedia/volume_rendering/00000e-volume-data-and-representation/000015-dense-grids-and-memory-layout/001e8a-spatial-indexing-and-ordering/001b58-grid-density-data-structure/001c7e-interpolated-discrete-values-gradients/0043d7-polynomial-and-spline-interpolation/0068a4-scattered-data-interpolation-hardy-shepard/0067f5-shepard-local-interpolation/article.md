# Shepard Local Interpolation
## Overview
Shepard Local Interpolation is a scattered data interpolation method that estimates volume values at regular grid points by weighting nearby sample points using inverse distance squared, utilizing a K-D tree to efficiently find the $k$ nearest neighbors [S1, unofficial]. It differs from its sibling [Hardy Global Interpolation](19995-hardy-global-interpolation.md) by limiting the influence to a local neighborhood rather than all sample points, which reduces computational cost and memory usage for large datasets [S1, unofficial].
## How it works
**Inputs:**
- A set of scattered sample points, each with 3D coordinates $(x, y, z)$ and a scalar value $d$.
- A target volume grid defined by dimensions $(dim\_x, dim\_y, dim\_z)$.
- An integer $k$, the number of nearest neighbors to consider.

**Outputs:**
- A volume grid `ret` of size $(dim\_x, dim\_y, dim\_z)$ containing interpolated values.

**Data Structures:**
- A K-D tree constructed from the input sample points to support efficient nearest-neighbor queries.
- A 3D array `ret` initialized to zeros.

**Steps:**
1. Initialize `ret` with zeros.
2. Construct a K-D tree from the input sample points.
3. For each grid point $P$ in the volume grid:
    a. Query the K-D tree for the $k$ nearest neighbors of $P$. Let the indices of these neighbors be $inds$.
    b. Retrieve the data values $d_i$ and coordinates $(x_i, y_i, z_i)$ for each of the $k$ neighbors.
    c. Initialize `numerator` and `denominator` to $0.0$.
    d. For each neighbor $i$ from $1$ to $k$:
        i. Calculate the squared Euclidean distance $dist_i$ between $P$ and the neighbor's coordinates:
           $$dist_i = (P_x - x_i)^2 + (P_y - y_i)^2 + (P_z - z_i)^2$$
        ii. If $dist_i$ is zero (the grid point coincides with a sample point), the weight is effectively infinite. In this case, the interpolated value is simply $d_i$. [G]
        iii. Otherwise, calculate the weight $w_i = \frac{1}{dist_i}$.
        iv. Accumulate `numerator` by adding $d_i \times w_i$.
        v. Accumulate `denominator` by adding $w_i$.
    e. If `denominator` is not $0.0$, set `ret[P]` to $\frac{numerator}{denominator}$.
    f. Otherwise, set `ret[P]` to $0.0$ (or `numerator` if it was initialized to a non-zero value, but since numerator starts at 0, it remains 0). [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shepard_local_interpolation(samples, grid_dims, k):
    """
    samples: list of tuples (x, y, z, d)
    grid_dims: tuple (dim_x, dim_y, dim_z)
    k: int, number of nearest neighbors
    """
    # Build K-D tree from samples
    # Note: Standard library KDTree or similar is assumed.
    # We define a simple wrapper for clarity.
    kdtree = KDTree(samples)
    
    ret = np.zeros(grid_dims)
    
    for x in range(grid_dims[0]):
        for y in range(grid_dims[1]):
            for z in range(grid_dims[2]):
                point = np.array([x, y, z])
                
                # Query k nearest neighbors
                # distances: squared Euclidean distances
                # indices: indices into the samples array
                distances, indices = kdtree.query(point, k=k)
                
                numerator = 0.0
                denominator = 0.0
                
                for i in range(k):
                    idx = indices[i]
                    sample = samples[idx]
                    sample_coords = np.array([sample[0], sample[1], sample[2]])
                    sample_val = sample[3]
                    
                    # Calculate squared Euclidean distance
                    dist_sq = np.sum(np.square(point - sample_coords))
                    
                    # Handle exact match
                    if dist_sq == 0.0:
                        ret[x, y, z] = sample_val
                        break # Found exact match, no need to continue
                    
                    # Weight is inverse squared distance
                    weight = 1.0 / dist_sq
                    
                    numerator += sample_val * weight
                    denominator += weight
                
                if denominator != 0.0:
                    ret[x, y, z] = numerator / denominator
                else:
                    ret[x, y, z] = 0.0
                    
    return ret
```

**Parameters:**
- $k$: Number of nearest neighbors. Typical values depend on data density; higher $k$ provides smoother results but increases computation [S1, unofficial].

**Edge Cases and Pitfalls:**
- If a grid point coincides exactly with a sample point, the distance is zero, leading to division by zero. The implementation must handle this by directly assigning the sample value [S1, unofficial].
- If $k$ is larger than the number of available sample points, the query should return all available points [G].
- Precision issues may arise if distances are very small or very large. Using squared distances avoids square roots but can lead to overflow/underflow if not managed [G].

## Subtopics

### Shepard Global Interpolation
A variant that estimates volume values by weighting all sample points using inverse distance squared, without using a K-D tree or limiting the number of neighbors [S1, unofficial]. It is simpler to implement but computationally expensive for large datasets as it requires iterating over all sample points for each grid point [S1, unofficial]. The provided implementation appears to have a specific indexing assumption (`self.random_Points[point[0]]`) which may be a bug or specific to a particular data layout, so general implementation should iterate over all samples [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot M \cdot \log S)$ for local, where $N$ is the number of grid points, $M$ is $k$, and $S$ is the number of sample points. For global, it is $O(N \cdot S)$.
- **Memory Complexity:** $O(S)$ for the K-D tree and sample storage.
- **Strengths:** Local version is efficient for large datasets due to K-D tree acceleration. Simple to understand and implement.
- **Limitations:** Quality depends heavily on the choice of $k$. Global version is slow for large $S$.
- **When to use:** Use local for large datasets where performance is critical. Use global for small datasets or when global smoothness is desired.

## Background
Originated from the work of David Shepard on scattered data interpolation [S1, unofficial]. The local variant with K-D tree acceleration is a common implementation in graphics applications for volume rendering, as seen in the [WebGL-Volume-rendering](https://github.com/jarusified/WebGL-Volume-rendering) repository [S1, unofficial].

## Sources
[S1] jarusified/WebGL-Volume-rendering, https://github.com/jarusified/WebGL-Volume-rendering, fetched 2026-10-27 (unofficial)
