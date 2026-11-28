# Point Generation from Voxel Grid

Also called: PointGenerator, Voxel to Point Conversion

## Overview
This method converts a sparse voxel grid into a discrete set of 3D points for rendering or simulation [S1, unofficial]. It differs from full active-voxel iteration by implementing a subsampling strategy (first, last, and strided intermediate points) when the number of active voxels in a leaf node exceeds a specified limit, ensuring bounded output size per leaf [S1, unofficial].

## How it works
**Inputs:**
*   A sparse voxel tree containing active voxels [S1, unofficial].
*   A coordinate transform object mapping voxel indices to world coordinates [S1, unofficial].
*   `voxelsPerLeaf`: The maximum number of points to generate per leaf node, defaulting to `TreeType::LeafNodeType::NUM_VOXELS` [S1, unofficial].

**Outputs:**
*   `mPoints`: A flat array storing interleaved XYZ coordinates, with size $N \times 3$ where $N$ is the total number of generated points [S1, unofficial].
*   `mIndices`: An array storing point indices, with size $N$ [S1, unofficial].

**Data Structures:**
*   `std::vector<GLfloat> mPoints`: Stores point positions [S1, unofficial].
*   `std::vector<GLuint> mIndices`: Stores point indices [S1, unofficial].
*   `std::vector<openvdb::Coord> coords`: Temporary buffer for collecting voxel coordinates within a leaf [S1, unofficial].

**Algorithm:**
1. Iterate over leaf nodes in parallel [S1, unofficial].
2. For each leaf node, retrieve the leaf and its active voxel iterator, then obtain the active voxel count (`activeVoxels`) [S1, unofficial].
3. If `activeVoxels <= voxelsPerLeaf`, iterate through all active voxels, transform their indices to world space, and insert each as a point [S1, unofficial].
4. If `activeVoxels > voxelsPerLeaf` and `voxelsPerLeaf == 1`, insert only the position of the first active voxel [S1, unofficial].
5. If `activeVoxels > voxelsPerLeaf` and `voxelsPerLeaf > 1`, collect all active voxel coordinates into a vector [S1, unofficial]. Insert the first and last coordinates [S1, unofficial]. Calculate stride $r = \lfloor \frac{\text{voxelsPerLeaf}}{\text{activeVoxels}} \rfloor$ [S1, unofficial]. Iterate $i$ from $1$ to $\text{voxelsPerLeaf}-2$, inserting coordinates at index $i \times r$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_points(tree, transform, voxels_per_leaf):
    """
    Generate points from a sparse voxel tree.
    
    Args:
        tree: Sparse voxel tree with leaf nodes.
        transform: Object with method indexToWorld(coord) returning Vec3d.
        voxels_per_leaf: Max points per leaf.
        
    Returns:
        mPoints: List of floats [x, y, z, ...]
        mIndices: List of integers [0, 1, ...]
    """
    mPoints = []
    mIndices = []
    index = 0
    
    # Iterate over leaf nodes (parallelized in implementation)
    for leaf in tree.leaf_nodes():
        active_count = leaf.on_voxel_count()
        
        if active_count <= voxels_per_leaf:
            # Case 1: Include all active voxels
            for voxel in leaf.active_voxels():
                pos = transform.indexToWorld(voxel.coord)
                mPoints.extend(pos)
                mIndices.append(index)
                index += 1
                
        elif voxels_per_leaf == 1:
            # Case 2: Only first voxel
            voxel = next(leaf.active_voxels())
            pos = transform.indexToWorld(voxel.coord)
            mPoints.extend(pos)
            mIndices.append(index)
            index += 1
            
        else:
            # Case 3: Subsampled
            coords = [voxel.coord for voxel in leaf.active_voxels()]
            
            # First point
            pos = transform.indexToWorld(coords[0])
            mPoints.extend(pos)
            mIndices.append(index)
            index += 1
            
            # Last point
            pos = transform.indexToWorld(coords[active_count - 1])
            mPoints.extend(pos)
            mIndices.append(index)
            index += 1
            
            # Intermediate points
            stride = voxels_per_leaf // active_count
            for i in range(1, voxels_per_leaf - 2):
                idx = i * stride
                pos = transform.indexToWorld(coords[idx])
                mPoints.extend(pos)
                mIndices.append(index)
                index += 1
                
    return mPoints, mIndices
```

**Parameters:**
*   `voxelsPerLeaf`: Default value is `TreeType::LeafNodeType::NUM_VOXELS` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Stride Zero/One:** When `activeVoxels > voxelsPerLeaf`, the sampling uses the first, last, and regularly spaced intermediate points [S1, unofficial]. If `activeVoxels` is small relative to `voxelsPerLeaf`, the stride $r$ might be 0 or 1, potentially causing duplicate points or insufficient coverage, though the loop bounds mitigate some issues [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of active voxels, as each active voxel is visited at least once [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the output points, with $O(L)$ temporary storage per leaf where $L$ is the number of active voxels in the leaf [S1, unofficial].
*   **Performance:** Parallelized using `tbb::parallel_for` for leaf iteration [S1, unofficial].
*   **Strengths:** Bounded output size per leaf; simple implementation; leverages existing sparse tree structures [S1, unofficial].
*   **Limitations:** Subsampling may lose detail if `voxelsPerLeaf` is too low; potential for duplicate points if stride is small [S1, unofficial].

## Background
This method is part of the OpenVDB library's point generation utilities, specifically designed for converting sparse voxel data into point clouds for rendering or other applications [S1, unofficial].

## Sources
[S1] AcademySoftwareFoundation/openvdb, https://github.com/AcademySoftwareFoundation/openvdb, fetched 2026-10-27 (unofficial)
