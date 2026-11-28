# Point-in-Box Inclusion Test via Sorted Indices and Bisect

Also called: Efficient Voxel Query via Sorted Indices and Bisect, Range Query via Binary Search on Sorted Coordinates

## Overview
This method performs point-in-box inclusion tests by pre-sorting point cloud indices along each axis and using binary search to determine occupancy [S1, unofficial]. It solves the problem of efficiently querying which points lie within an orthogonal bounding box by reducing the search space to $O(\log N)$ per axis [S1, unofficial]. It differs from its parent, [Point-in-Box Inclusion Test](1400-point-in-box-inclusion-test.md), by explicitly leveraging sorted index arrays and bisect operations to handle range queries, rather than generic partitioning logic [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of $N$ 3D points $P = \{p_1, \dots, p_N\}$ where $p_i = (x_i, y_i, z_i)$, and a query box defined by $[x_{min}, x_{max}] \times [y_{min}, y_{max}] \times [z_{min}, z_{max}]$ [S1, unofficial].
- Output: A subset of points $P_{in} \subseteq P$ such that $p_i \in P_{in}$ if and only if $x_{min} \le x_i \le x_{max}$, $y_{min} \le y_i \le y_{max}$, and $z_{min} \le z_i \le z_{max}$ [S1, unofficial].

**Data Structures:**
Three arrays of indices sorted by coordinate value are maintained:
- `idx_x`: Indices of $P$ sorted by $x$-coordinate.
- `idx_y`: Indices of $P$ sorted by $y$-coordinate.
- `idx_z`: Indices of $P$ sorted by $z$-coordinate.
The corresponding sorted coordinate values are stored in `sorted_x`, `sorted_y`, and `sorted_z` [S1, unofficial]. *Note: The provided code snippet in the source does not contain this method; the description relies on the source notes [S1, unofficial].*

**Steps:**
1. **Initialization**: Extract $x$, $y$, and $z$ coordinates from $P$. Create index arrays $[0, \dots, N-1]$. Sort each index array based on its corresponding coordinate to form `idx_x`, `idx_y`, `idx_z` [S1, unofficial]. Extract sorted coordinate values into `sorted_x`, `sorted_y`, `sorted_z` [S1, unofficial].
2. **Query Processing**: For a query box, use `bisect.bisect_left` to find the first index in each sorted array where the coordinate is $\ge$ the lower bound, and `bisect.bisect_right` to find the first index where the coordinate is $>$ the upper bound [S1, unofficial].
3. **Intersection**: Map the sorted indices back to original point indices for each axis using the index arrays. Compute the set intersection of the three index sets to identify points satisfying all three bounds [G].
4. **Return**: Return the points corresponding to the intersected indices [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import bisect

class VoxelQuery:
    def __init__(self, points):
        """points: list of (x, y, z) tuples or 2D array-like"""
        self.points = points
        self.N = len(points)
        # Pre-sort indices by each coordinate
        self.idx_x = sorted(range(self.N), key=lambda i: points[i][0])
        self.idx_y = sorted(range(self.N), key=lambda i: points[i][1])
        self.idx_z = sorted(range(self.N), key=lambda i: points[i][2])
        # Extract sorted coordinate values for binary search
        self.sorted_x = [points[i][0] for i in self.idx_x]
        self.sorted_y = [points[i][1] for i in self.idx_y]
        self.sorted_z = [points[i][2] for i in self.idx_z]

    def get_points_in_range(self, x_min, x_max, y_min, y_max, z_min, z_max):
        # Find ranges in sorted arrays
        x_lo = bisect.bisect_left(self.sorted_x, x_min)
        x_hi = bisect.bisect_right(self.sorted_x, x_max)
        y_lo = bisect.bisect_left(self.sorted_y, y_min)
        y_hi = bisect.bisect_right(self.sorted_y, y_max)
        z_lo = bisect.bisect_left(self.sorted_z, z_min)
        z_hi = bisect.bisect_right(self.sorted_z, z_max)
        
        # Map back to original indices
        orig_x = set(self.idx_x[x_lo:x_hi])
        orig_y = set(self.idx_y[y_lo:y_hi])
        orig_z = set(self.idx_z[z_lo:z_hi])
        
        # Intersect sets to find points inside the 3D box
        final_indices = orig_x & orig_y & orig_z
        
        return [self.points[i] for i in final_indices]
```
*Note: The source notes describe returning "the slice of points between these indices," which implies a 1D query or a simplified description [S1, unofficial]. The pseudocode implements the standard 3D intersection logic [G].*

**Parameters:**
- `x_min`, `x_max`, `y_min`, `y_max`, `z_min`, `z_max`: Float values defining the box boundaries [S1, unofficial].

**Edge Cases:**
- Empty point cloud: Return empty set [G].
- Box outside point cloud bounds: `bisect` returns equal bounds, yielding empty slices [G].
- Duplicate coordinates: `bisect_left` and `bisect_right` correctly define inclusive ranges for duplicates [G].

## Subtopics

### [Searchsorted](7938-searchsorted.md)
Searchsorted finds the indices where query points `v` should be inserted into a sorted reference array `a` to maintain order. It serves as the core lookup mechanism for the parent method, enabling range queries on pre-sorted coordinate arrays. Unlike standard binary search which returns a single insertion point, this vectorized implementation explicitly computes both lower (`idx_lo`) and upper (`idx_hi`) bounds to handle edge cases and range queries robustly.

## Operating characteristics
- **Time Complexity**: $O(N \log N)$ for preprocessing (sorting). $O(\log N)$ for each axis query. Total query time is $O(\log N + K)$ where $K$ is the number of candidate points before intersection [G].
- **Memory Complexity**: $O(N)$ to store the sorted index and coordinate arrays [S1, unofficial].
- **Strengths**: Fast query times for static point clouds. Simple to implement using standard library bisect functions [S1, unofficial].
- **Limitations**: Preprocessing time is significant for large datasets. Dynamic updates (insertions/deletions) are expensive ($O(N)$) [G]. The source notes do not specify how 3D intersection is handled efficiently beyond the index mapping [S1, unofficial].
- **When to use**: Static point clouds with frequent range queries [S1, unofficial].
- **When not to use**: Dynamic point clouds or when memory is constrained [G].

## Background
This method is derived from standard computational geometry techniques for range searching [G]. The specific implementation is referenced in the [MasonAndrewHarrison/Point-Cloud-Voxelizer](https://github.com/MasonAndrewHarrison/Point-Cloud-Voxelizer) repository [S1, unofficial]. It is used in voxelization pipelines to efficiently determine which points contribute to a voxel grid [S1, unofficial].

## Sources
[S1] MasonAndrewHarrison/Point-Cloud-Voxelizer, https://github.com/MasonAndrewHarrison/Point-Cloud-Voxelizer, fetched 2026-10-27 (unofficial)
