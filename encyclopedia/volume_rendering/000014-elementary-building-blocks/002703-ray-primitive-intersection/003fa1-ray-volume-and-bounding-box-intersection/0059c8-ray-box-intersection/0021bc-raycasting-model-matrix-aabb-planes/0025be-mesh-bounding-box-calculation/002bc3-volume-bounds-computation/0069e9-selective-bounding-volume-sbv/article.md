# Selective Bounding Volume (SBV)
## Overview
Selective Bounding Volume (SBV) is a 3D extension of the 2D bounding box method used to identify and extract a Volume of Interest (VOI) from a 3D medical image by ignoring blank slices and confining coordinates to active portions [S1]. It sets itself apart from shape-adaptive methods by refusing to discard any active portions in any slices, thereby preserving clinically significant background details that other methods might remove [S1].
## How it works
**Inputs and Outputs:**
- Input: A 3D medical image $X$ with dimensions $[x, y, z]$ (width, height, depth) [S1].
- Output: An extracted sub-volume $VOI$ containing only the active region [S1].

**Data Structures:**
- `temp_coordinates`: An array of 4 integers $[x_1, x_2, x_3, x_4]$ initialized to zero. These store the bounding box coordinates $(min\_row, min\_col, max\_row, max\_col)$ for each non-blank slice [S1].
- `RowMin`, `ColumnMin`, `RowMax`, `ColumnMax`: Integer variables storing the global minimum and maximum coordinates across all active slices [S1].
- `z1`, `z2`: Integer indices marking the first and last non-zero slices [S1].

**Steps:**
1.  **Get Dimensions:** Determine the size of the input image $X$ as $[x, y, z]$ [S1].
2.  **Initialize:** Declare `RowMin`, `ColumnMin`, `RowMax`, `ColumnMax` as integers. Initialize `temp_coordinates` $[x_1, x_2, x_3, x_4]$ to $[0, 0, 0, 0]$ [S1].
3.  **Identify Active Z-Range:** Iterate through all $z$-slices to search for zero-intensity (blank) slices. Identify $z_1$ as the index of the first non-zero slice and $z_2$ as the index of the last non-zero slice [S1]. Blank slices at the foremost and end of the volume are ignored to prevent them from affecting the bounding volume calculation [S1].
4.  **Compute Slice Bounding Boxes:** For each slice $z$ from $z_1$ to $z_2$:
    - Apply 2D bounding box operations on the slice to find its local active region [S1].
    - Update `temp_coordinates` with the local bounding box coordinates $[x_1, x_2, x_3, x_4]$ for that slice [S1].
5.  **Compute Global Bounds:** Determine the global bounding box by taking the minimum and maximum of the accumulated coordinates:
    - $RowMin = \min(x_1)$
    - $ColumnMin = \min(x_2)$
    - $RowMax = \max(x_3)$
    - $ColumnMax = \max(x_4)$
    where the min/max functions operate over all slices from $z_1$ to $z_2$ [S1].
6.  **Extract VOI:** Iterate through the volume using the global bounds:
    - For $k$ from $z_1$ to $z_2$:
        - For $i$ from $RowMin$ to $RowMax$:
            - For $j$ from $ColumnMin$ to $ColumnMax$:
                - Assign $VOI(i - RowMin, j - ColumnMin, k - z_1) \leftarrow X(i, j, k)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def selective_bounding_volume(X):
    # Step 1: Get dimensions
    x, y, z = X.shape
    
    # Step 2: Initialize variables
    temp_coordinates = [0, 0, 0, 0] # [x1, x2, x3, x4]
    z1 = None
    z2 = None
    
    # Step 3: Identify active Z-range
    # Search for zero intensity slices
    for k in range(z):
        # Check if slice k is blank (all zeros)
        if not is_blank_slice(X[:, :, k]):
            if z1 is None:
                z1 = k
            z2 = k
            
    # If no active slices found, return empty or handle error
    if z1 is None:
        return None

    # Step 4: Compute slice bounding boxes
    for k in range(z1, z2 + 1):
        # Apply Bounding Box operations on slice k
        # This returns [min_row, min_col, max_row, max_col] for the active region
        local_coords = compute_2d_bounding_box(X[:, :, k])
        
        # Update temp_coordinates with the current slice's coordinates
        # Note: The source implies accumulating all coordinates to find global min/max
        # We store them in a list to process in Step 5
        temp_coordinates.append(local_coords)
        
    # Step 5: Assign global bounds
    # Extract all x1, x2, x3, x4 from the collected coordinates
    all_x1 = [c[0] for c in temp_coordinates[1:]] # Skip initial zero
    all_x2 = [c[1] for c in temp_coordinates[1:]]
    all_x3 = [c[2] for c in temp_coordinates[1:]]
    all_x4 = [c[3] for c in temp_coordinates[1:]]
    
    RowMin = min(all_x1)
    ColumnMin = min(all_x2)
    RowMax = max(all_x3)
    ColumnMax = max(all_x4)
    
    # Step 6: Extract VOI
    height = RowMax - RowMin + 1
    width = ColumnMax - ColumnMin + 1
    depth = z2 - z1 + 1
    
    VOI = np.zeros((height, width, depth), dtype=X.dtype)
    
    for k in range(z1, z2 + 1):
        for i in range(RowMin, RowMax + 1):
            for j in range(ColumnMin, ColumnMax + 1):
                VOI[i - RowMin, j - ColumnMin, k - z1] = X[i, j, k]
                
    return VOI

def is_blank_slice(slice_2d):
    # Check if all values are zero
    return np.all(slice_2d == 0)

def compute_2d_bounding_box(slice_2d):
    # Find coordinates of non-zero elements
    rows, cols = np.where(slice_2d != 0)
    if len(rows) == 0:
        return [0, 0, 0, 0]
    min_row = np.min(rows)
    max_row = np.max(rows)
    min_col = np.min(cols)
    max_col = np.max(cols)
    return [min_row, min_col, max_row, max_col]
```

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- **Blank Slices:** The method explicitly ignores blank slices (null image slices) at the foremost and end of the volume. If the entire volume is blank, the algorithm should handle the case where $z_1$ is never set [S1].
- **Active Portions:** Unlike shape-adaptive methods, SBV does not discard any active portions in any slices. This preserves clinically significant background details but may result in a larger bounding volume if the object is sparse within the active slices [S1].

## Operating characteristics
**Time and Memory Complexity:**
- Not given in the sources.

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Not given in the sources.

**Strengths and Limitations:**
- **Strengths:** Preserves all active portions in any slices, including clinically significant background details, which shape-adaptive methods might discard [S1]. It is an automated morphological operation-based object detection technique [S1].
- **Limitations:** Not given in the sources.

**When to Use:**
- Use when extracting a VOI from 3D medical images where preserving all active background details is critical, and blank slices exist at the boundaries of the volume [S1].

**When Not to Use:**
- Not given in the sources.

## Background
**Origin:**
- The Selective Bounding Volume (SBV) method was proposed in the paper "On a hybrid lossless compression technique for three‐dimensional medical images" [S1].

**History and Development:**
- The method is described as a 3D extension of the Bounding Box method [S1]. It was developed to overcome issues with existing techniques that had high algorithmic overhead or discarded non-object details [S1].

**Where it is Used:**
- 3D medical imaging, specifically for lossless compression and object detection [S1].

## Sources
[S1] On a hybrid lossless compression technique for three‐dimensional medical images, https://doi.org/10.1002/acm2.12960, fetched 2026-11-08
