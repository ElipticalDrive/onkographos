# Occupancy Grid Filling
## Overview
Occupancy Grid Filling is a post-processing step that fills interior regions of an occupancy grid by identifying the "outside" label and setting all other background labels to occupied (1) [S1, unofficial]. It ensures that enclosed voids are treated as solid while preserving original non-zero data (e.g., colors) attached to occupied voxels [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: An HDF5 file containing an occupancy tensor (numpy array) [S1, unofficial].
- Output: An HDF5 file containing the filled occupancy tensor [S1, unofficial].

**Data Structures:**
- **HDF5 Dataset**: The input and output files use an HDF5 dataset with the key `'tensor'` (default) containing a numpy array [S1, unofficial]. Chunks are created based on the tensor shape, with dimensions > 2 set to chunk size 1 [S1, unofficial].

**Parameters:**
- `background`: The background label value for `skimage.morphology.label`, set to 1 [S1, unofficial].
- `connectivity`: The connectivity for `skimage.morphology.label`, set to 1 (6-connectivity in 3D) [S1, unofficial].

**Steps:**
1. Read the input HDF5 file containing the occupancy tensor [S1, unofficial].
2. If the occupancy tensor has fewer than 4 dimensions, expand the dimensions to add a batch dimension [S1, unofficial].
3. Initialize a filled tensor of zeros with the same shape as occupancy [S1, unofficial].
4. Loop through each sample $n$ in the batch (range of `occupancy.shape[0]`) [S1, unofficial]:
    a. Label connected components in the occupancy slice using `skimage.morphology.label` with `background=1` and `connectivity=1` [S1, unofficial].
    b. Identify the outside label as the label value at the corner of the grid (`labels[0][0][0]`) [S1, unofficial].
    c. Set all voxels in the filled slice that do not have the `outside_label` to 1 [S1, unofficial].
    d. Preserve original non-zero values (e.g., colors) in the filled slice by copying them from the original occupancy slice to the filled slice where `occupancy[n] != 0` [S1, unofficial].
5. Write the filled tensor to the output HDF5 file [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import h5py
import numpy as np
from skimage.morphology import label

def fill_occupancy(input_path: str, output_path: str):
    """
    Fill interior regions of an occupancy grid.
    
    :param input_path: Path to input HDF5 file.
    :param output_path: Path to output HDF5 file.
    """
    with h5py.File(input_path, 'r') as f_in:
        occupancy = f_in['tensor'][:]
        
        # Expand dimensions if necessary to add batch dimension
        if occupancy.ndim < 4:
            occupancy = np.expand_dims(occupancy, axis=0)
            
        # Initialize filled tensor with zeros
        filled = np.zeros_like(occupancy, dtype=occupancy.dtype)
        
        # Loop through batch
        for n in range(occupancy.shape[0]):
            # Label connected components
            labels = label(occupancy[n], background=1, connectivity=1)
            
            # Identify outside label from corner voxel
            outside_label = labels[0, 0, 0]
            
            # Set all non-outside voxels to 1
            filled[n, labels != outside_label] = 1
            
            # Preserve original non-zero values (e.g., colors)
            filled[n, occupancy[n] != 0] = occupancy[n, occupancy[n] != 0]
            
    with h5py.File(output_path, 'w') as f_out:
        # Chunks: dimensions > 2 set to chunk size 1
        chunks = tuple(1 if i > 2 else occupancy.shape[i] for i in range(occupancy.ndim))
        f_out.create_dataset('tensor', data=filled, chunks=chunks)
```

**Edge Cases and Pitfalls:**
- The outside label is determined by the corner voxel `labels[0][0][0]` [S1, unofficial]. If the corner voxel belongs to an internal object rather than the true exterior, the outside label will be misidentified and the filling will be incorrect [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Not given in the sources.

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Not given in the sources.

**Strengths and Limitations:**
- Not given in the sources.

**When to Use:**
- Not given in the sources.

**When Not to Use:**
- Not given in the sources.

## Background
**Origin:**
- The method is described in the `mesh-voxelization` GitHub repository by David Stutz [S1, unofficial].

**History and Development:**
- Not given in the sources.

**Where it is Used:**
- Not given in the sources.

## Sources
[S1] Mesh Voxelization - GitHub, https://github.com/davidstutz/mesh-voxelization, fetched 2026-10-26 (unofficial)
