# Connected Component Labeling

Also called: CCL

## Overview
Connected Component Labeling (CCL) identifies and assigns unique identifiers to contiguous groups of occupied voxels in a 3D grid. It solves the problem of distinguishing distinct geometric entities from noise or isolated artifacts by grouping voxels that share a defined connectivity. Unlike its parent method, which filters based on component size, CCL provides the raw labeling data structure required for such filtering or for identifying separate objects in a scene.

## How it works
**Inputs:** A set of occupied voxel coordinates (typically as a list of tuples or a sparse key set).
**Outputs:** A mapping from original voxel indices to integer component labels.

**Data Structures:**
- `voxels`: Input array of shape $(N, 3)$ containing integer coordinates.
- `keys`: A deduplicated, sorted array of unique voxel coordinates with an added margin.
- `labels`: Output array of shape $(N,)$ containing integer labels.

**Parameters:**
- `connectivity`: Integer, either 6 or 26.
  - 6: Voxels are connected if they share a face (Von Neumann neighborhood).
  - 26: Voxels are connected if they share a face, edge, or corner (Moore neighborhood).
  - Note: Connectivity 18 is not supported by the underlying routine [S1, unofficial].

**Steps:**
1. **Validate Connectivity:** Ensure `connectivity` is 6 or 26. Raise an error otherwise [S1, unofficial].
2. **Key Generation:** Convert the input voxel coordinates to a canonical key format with a margin of 1. This step deduplicates the input and prepares the data for efficient neighbor lookup.
   $$ \text{keys}, \text{shift} = \text{to\_keys}(\text{voxels}, \text{margin}=1) $$
3. **Labeling:** Call the core component labeling routine on the unique keys. This returns the total number of components and a label for each unique key.
   $$ \text{num\_components}, \text{key\_labels} = \text{\_component\_labels}(\text{keys}, \text{connectivity}) $$
4. **Mapping:** Map the labels from the unique keys back to the original input row order. This is achieved by searching for the original voxels within the sorted keys array and indexing into the key labels.
   $$ \text{labels} = \text{key\_labels}[\text{np.searchsorted}(\text{keys}, \text{voxels})] $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def connected_component_labeling(voxels, connectivity=6):
    """
    Labels connected components of a voxel set.
    
    Args:
        voxels: (N, 3) array of integer voxel coordinates.
        connectivity: 6 or 26.
        
    Returns:
        labels: (N,) array of integer component labels.
    """
    # Step 1: Validate connectivity
    if connectivity not in (6, 26):
        raise ValueError(f"Connectivity must be 6 or 26, got {connectivity}")
    
    # Step 2: Convert voxels to keys with margin 1
    # This deduplicates and sorts the voxels for efficient processing.
    # The 'to_keys' function is assumed to handle the margin expansion 
    # and canonicalization required for neighbor checks.
    keys, shift = to_keys(voxels, margin=1)
    
    # Step 3: Compute labels for unique keys
    # The underlying routine uses a union-find or flood-fill approach 
    # on the sparse grid defined by 'keys'.
    num_components, key_labels = _component_labels(keys, connectivity)
    
    # Step 4: Map labels back to original input order
    # searchsorted finds the index of each original voxel in the sorted keys array.
    # This index is used to retrieve the corresponding label.
    indices = np.searchsorted(keys, voxels)
    labels = key_labels[indices]
    
    return labels
```

**Edge Cases and Pitfalls:**
- **Connectivity 18:** The underlying routine does not support 18-connectivity (face and edge sharing, but not corner). Using it will raise an error [S1, unofficial].
- **Precision:** Coordinates must be integers. Floating point coordinates should be rounded or quantized before labeling.
- **Boundaries:** The margin of 1 in `to_keys` ensures that boundary voxels are correctly identified as connected to their neighbors without index out-of-bounds errors during neighbor lookup.

## Subtopics

### Grid Cell Labeling Algorithm
A variant used in fluid simulation contexts where cells are labeled as Full, Unknown, Empty, or Surface. It initializes cells with surface markers as Full and others as Unknown. It identifies a guaranteed empty region, labels it Empty, and propagates this label to neighboring Unknown cells. Then, Unknown cells adjacent to Full cells are labeled Full. Finally, cells containing original surface markers are explicitly labeled Surface. This effectively fills the fluid body and atmosphere while preserving the surface boundary [S2, unofficial].

## Operating characteristics
**Time Complexity:** Depends on the underlying `_component_labels` implementation. Typically $O(N \alpha(N))$ where $N$ is the number of unique voxels and $\alpha$ is the inverse Ackermann function (for Union-Find), or $O(N)$ for flood-fill on sparse grids. The mapping step is $O(N \log N)$ due to sorting/searching.
**Memory Complexity:** $O(N)$ for storing keys, labels, and intermediate structures.
**Performance:** Efficient for sparse voxel sets. The deduplication and key-based approach avoids iterating over empty space.
**Strengths:** Robust handling of sparse data; clear separation of distinct objects.
**Limitations:** Does not inherently filter by size; requires post-processing for noise removal.

## Background
Connected Component Labeling is a fundamental image processing and computer vision technique. In the context of volume rendering and voxel operations, it is adapted for 3D sparse grids. The specific implementation described here is part of the `sparse-cubes` library, designed for efficient processing of large-scale voxel data [S1, unofficial]. The Grid Cell Labeling variant originates from fluid simulation research [S2, unofficial].

## Sources
[S1] navis-org/sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
[S2] Efficient computation and rendering of smoke animation, http://hdl.handle.net/2346/16985, fetched 2026-10-26 (unofficial)
