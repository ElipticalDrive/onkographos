# 3-D Thinning Skeleton

Also called: Thinning algorithm, Topology-preserving data compression

## Overview
3-D Thinning Skeleton is a topology-preserving data compression technique that reduces a binary voxel object to a compact skeleton lying along its medial surface or axis [S1, unofficial]. It solves the problem of excessive sub-branches found in Medial Axis Transform (MAT) skeletons, making it more suitable for applications like diecastability evaluation [S1, unofficial]. Unlike MAT, which computes exact geometric centers, thinning iteratively deletes non-critical boundary points, preserving the object's topological structure while simplifying its geometry.

## How it works
**Inputs and Outputs:**
*   **Input:** A binary voxel grid where "black" voxels represent the object and "white" voxels represent the background.
*   **Output:** A binary voxel grid representing the skeleton, where only the remaining object voxels are marked as black.

**Data Structures:**
*   **Voxel Grid:** A 3D array of boolean or integer values.
*   **Connectivity Definitions:**
    *   **Object Connectivity:** 26-connectivity (vertex-adjacent). Two voxels are 26-connected if they share a face, edge, or vertex [S1, unofficial].
    *   **Background Connectivity:** 6-connectivity (face-adjacent). Two voxels are 6-connected if they share a face [S1, unofficial].

**Parameters:**
*   **Object Connectivity:** 26 [S1, unofficial].
*   **Background Connectivity:** 6 [S1, unofficial].
*   **Termination Condition:** No more non-critical boundary points can be deleted.

**Steps:**
1.  **Classify Boundary Voxels:** Identify boundary voxels using 6-neighborhood connectivity. A boundary voxel is a black voxel that has at least one white 6-neighbor [S1, unofficial].
2.  **Identify Non-Critical Points:** Determine which boundary voxels are "simple points." A point is simple if its deletion preserves the topology of the object. This check relies on the parent method, [Simple Point Detection via Euler Characteristic](20385-simple-point-detection-via-euler-characteristic.md), which verifies if the Euler characteristic remains unchanged upon removal [S1, unofficial].
3.  **Protect End Points:** Identify geometric end points (termini) of the skeleton. These points must be protected from deletion to preserve the geometric features and length of the skeleton [S1, unofficial].
4.  **Iterative Deletion:** Iteratively delete identified non-critical boundary points (that are not end points) until no such points remain. The process repeats until the skeleton is stable [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def thin_3d_skeleton(volume):
    """
    volume: 3D numpy array of booleans (True for object/black, False for background/white)
    Returns: 3D numpy array representing the skeleton
    """
    # Copy volume to avoid modifying input
    current_volume = volume.copy()
    
    # Define connectivity masks
    # 6-neighborhood for background check (face-adjacent)
    neighbors_6 = [
        (1, 0, 0), (-1, 0, 0),
        (0, 1, 0), (0, -1, 0),
        (0, 0, 1), (0, 0, -1)
    ]
    
    # 26-neighborhood for object connectivity (vertex-adjacent)
    neighbors_26 = []
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                neighbors_26.append((dx, dy, dz))

    changed = True
    while changed:
        changed = False
        # Identify boundary voxels: Black voxels with at least one White 6-neighbor
        boundary_voxels = []
        for x in range(volume.shape[0]):
            for y in range(volume.shape[1]):
                for z in range(volume.shape[2]):
                    if not current_volume[x, y, z]:
                        continue
                    
                    is_boundary = False
                    for dx, dy, dz in neighbors_6:
                        nx, ny, nz = x + dx, y + dy, z + dz
                        # Check bounds
                        if 0 <= nx < volume.shape[0] and 0 <= ny < volume.shape[1] and 0 <= nz < volume.shape[2]:
                            if not current_volume[nx, ny, nz]:
                                is_boundary = True
                                break
                        else:
                            # Out of bounds is considered background
                            is_boundary = True
                            break
                    
                    if is_boundary:
                        boundary_voxels.append((x, y, z))
        
        # Process boundary voxels
        # Note: Parallel thinning variants (see Subtopics) handle simultaneous deletion.
        # This basic implementation assumes sequential or single-pass deletion per iteration.
        # To preserve topology strictly, one must check if deletion is safe.
        
        voxels_to_delete = []
        for (x, y, z) in boundary_voxels:
            # Check if it is an end point (protected)
            # An end point is a boundary voxel with only one 26-connected object neighbor
            # that is also a boundary voxel? Or simply degree 1 in the skeleton graph?
            # The source mentions protecting end points to preserve geometric features [S1, unofficial].
            # A common definition: A voxel is an end point if it has exactly one 26-connected neighbor in the current volume.
            object_neighbors = 0
            for dx, dy, dz in neighbors_26:
                nx, ny, nz = x + dx, y + dy, z + dz
                if 0 <= nx < volume.shape[0] and 0 <= ny < volume.shape[1] and 0 <= nz < volume.shape[2]:
                    if current_volume[nx, ny, nz]:
                        object_neighbors += 1
            
            if object_neighbors == 1:
                continue # Protect end points
            
            # Check if simple point (topology preserving)
            # This requires the Euler Characteristic check from the parent article
            # [Simple Point Detection via Euler Characteristic](20385-simple-point-detection-via-euler-characteristic.md)
            # For implementation completeness, we assume a function exists that checks this.
            if is_simple_point(current_volume, x, y, z, neighbors_26, neighbors_6):
                voxels_to_delete.append((x, y, z))
        
        # Delete voxels
        for (x, y, z) in voxels_to_delete:
            current_volume[x, y, z] = False
            changed = True
            
    return current_volume

def is_simple_point(vol, x, y, z, n26, n6):
    """
    Placeholder for the logic defined in [Simple Point Detection via Euler Characteristic].
    In a full implementation, this would calculate the Euler characteristic of the local neighborhood
    before and after hypothetical deletion.
    """
    # [G] Standard implementation involves checking if the number of connected components
    # and holes in the neighborhood remains invariant.
    # For this article, we rely on the parent article's definition.
    return True # Placeholder: Actual logic depends on parent article implementation
```

**Edge Cases and Pitfalls:**
*   **Connectivity Paradox:** Using identical connectivity for object and background causes topological paradoxes. It is essential to use 26-connectivity for the object (part) and 6-connectivity for the background to ensure continuity and avoid ambiguity [S1, unofficial].
*   **Annihilation:** True parallel thinning can lead to the complete deletion of an object ("annihilation") or connectivity loss. Sequential processing or hybrid orders (as in Directional Parallel Thinning) mitigate this [S1, unofficial].
*   **End Point Protection:** Failure to protect end points may result in the erosion of geometric features, shortening the skeleton incorrectly [S1, unofficial].

## Subtopics

### [Directional Parallel Thinning](23079-directional-parallel-thinning.md)
A parallel variant that processes boundary voxels in specific spatial directions across six sub-iterations to prevent connectivity loss and annihilation. It restricts simultaneous deletions to voxels of a single border type per sub-cycle and includes over-shrinking prevention.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Generally depends on the number of boundary voxels and the number of iterations required for convergence.
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels in the input volume, as it operates in-place or with a single copy.
*   **Strengths:** Produces a skeleton along the medial surface/axis without the excessive sub-branches typical of Medial Axis Transform (MAT) skeletons [S1, unofficial]. Suitable for applications like diecastability evaluation where topological correctness and medial axis approximation are prioritized over exact geometric distance [S1, unofficial].
*   **Limitations:** The resulting skeleton may require post-pruning operations to remove noise or spurious branches [S1, unofficial]. Performance can be sensitive to the order of voxel processing in sequential implementations.
*   **When to use:** When a topologically correct, compact representation of a 3D object's shape is needed, and the exact geometric medial axis is not required.
*   **When not to use:** When exact geometric distances to the boundary are needed (use MAT instead) or when the object has complex internal structures that might be collapsed incorrectly by simple thinning.

## Background
Thinning is a topological data compression technique commonly used for digital images [S1, unofficial]. While many algorithms were proposed for 2-D images (e.g., Lam et al., 1992; Pavlidis, 1982), attention has recently been devoted to 3-D thinning skeletons (e.g., Ma, 1994; Saha and Chaudhuri, 1994; Lee et al., 1994; Kong and Rosenfeld, 1989) [S1, unofficial]. The specific application discussed in the source is for die casting part and die design visualization [S1, unofficial].

## Sources
[S1] Simple visualization techniques for die casting part and die design. Final report, July 1, 1995--September 30, 1997, https://digital.library.unt.edu/ark:/67531/metadc690873/?q=voxelization triangle mesh, fetched 2026-10-28 (unofficial)
