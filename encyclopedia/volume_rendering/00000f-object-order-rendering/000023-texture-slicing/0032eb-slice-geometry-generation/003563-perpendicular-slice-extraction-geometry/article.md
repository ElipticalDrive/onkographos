# Perpendicular Slice Extraction Geometry
Also called: Slice plane matrix calculation, extractPerpendicularPlane

## Overview
Perpendicular Slice Extraction Geometry computes the geometric properties of a slice perpendicular to a specific axis (x, y, or z) at a given RAS (Right-Anterior-Superior) index [S1, unofficial]. It returns the slice dimensions, a plane matrix defining the orientation and position in RAS space, and a `sliceAccess` function to map 2D slice coordinates to 3D volume data indices [S1, unofficial]. It sets itself apart from general plane slicing by enforcing strict axis-aligned perpendicularity and handling the coordinate transformation between RAS world space and IJK volume index space explicitly [S1, unofficial].

## How it works
**Inputs:**
- `RASIndex`: The integer index of the slice along the chosen axis in RAS space.
- `axis`: The axis of extraction ('x', 'y', or 'z').
- `RASDimensions`: The dimensions of the volume in RAS space (xLength, yLength, zLength).
- `spacing`: The physical spacing of the volume voxels (spacing[x], spacing[y], spacing[z]).
- `inverseMatrix`: The inverse of the transformation matrix from RAS to IJK space (used if not in segmentation mode) [S1, unofficial].
- `volume`: The volume data object with an `access(i, j, k)` method [S1, unofficial].

**Outputs:**
- `iLength`: The number of voxels in the first direction (i) of the slice.
- `jLength`: The number of voxels in the second direction (j) of the slice.
- `matrix`: A 4x4 transformation matrix representing the slice plane in RAS space.
- `planeWidth`: The physical width of the slice in the first direction.
- `planeHeight`: The physical height of the slice in the second direction.
- `sliceAccess(i, j)`: A function that returns the volume value at 2D slice coordinates $(i, j)$ [S1, unofficial].

**Data Structures:**
- `axisInIJK`: A 3D vector representing the extraction axis in IJK space.
- `firstDirection`, `secondDirection`: 3D vectors representing the axes of the slice plane in RAS space.
- `planeMatrix`: A 4x4 identity matrix initialized to store the plane's orientation and position.
- `dimensions`: A tuple `(xLength, yLength, zLength)` representing the volume dimensions [S1, unofficial].

**Steps:**
1.  **Initialize Vectors:** Set `axisInIJK`, `firstDirection`, `secondDirection`, `planeMatrix` (identity), and `dimensions` [S1, unofficial].
2.  **Set Directions and Spacings based on Axis:**
    -   **If axis is 'x':**
        -   `axisInIJK` = $(1, 0, 0)$
        -   `firstDirection` = $(0, 0, -1)$
        -   `secondDirection` = $(0, -1, 0)$
        -   `firstSpacing` = `spacing[z]`
        -   `secondSpacing` = `spacing[y]`
        -   `IJKIndex` = $(RASIndex, 0, 0)$
        -   Rotate `planeMatrix` by $\pi/2$ around the Y-axis.
        -   `positionOffset` = $(RASDimensions[0] - 1) / 2$
        -   Set `planeMatrix` position to $(RASIndex - positionOffset, 0, 0)$ [S1, unofficial].
    -   **If axis is 'y':**
        -   `axisInIJK` = $(0, 1, 0)$
        -   `firstDirection` = $(1, 0, 0)$
        -   `secondDirection` = $(0, 0, 1)$
        -   `firstSpacing` = `spacing[x]`
        -   `secondSpacing` = `spacing[z]`
        -   `IJKIndex` = $(0, RASIndex, 0)$
        -   Rotate `planeMatrix` by $-\pi/2$ around the X-axis.
        -   `positionOffset` = $(RASDimensions[1] - 1) / 2$
        -   Set `planeMatrix` position to $(0, RASIndex - positionOffset, 0)$ [S1, unofficial].
    -   **If axis is 'z' (default):**
        -   `axisInIJK` = $(0, 0, 1)$
        -   `firstDirection` = $(1, 0, 0)$
        -   `secondDirection` = $(0, -1, 0)$
        -   `firstSpacing` = `spacing[x]`
        -   `secondSpacing` = `spacing[y]`
        -   `IJKIndex` = $(0, 0, RASIndex)$
        -   `positionOffset` = $(RASDimensions[2] - 1) / 2$
        -   Set `planeMatrix` position to $(0, 0, RASIndex - positionOffset)$ [S1, unofficial].
3.  **Transform Directions (Non-Segmentation Mode):** If not in segmentation mode, apply `inverseMatrix` to `firstDirection`, `secondDirection`, and `axisInIJK`, then normalize them [S1, unofficial].
4.  **Assign Arglets and Calculate Lengths:**
    -   Set `firstDirection.arglet` = 'i' and `secondDirection.arglet` = 'j' [S1, unofficial].
    -   Calculate `iLength` = $\lfloor |firstDirection \cdot dimensions| \rfloor$
    -   Calculate `jLength` = $\lfloor |secondDirection \cdot dimensions| \rfloor$
    -   Calculate `planeWidth` = $|iLength \times firstSpacing|$
    -   Calculate `planeHeight` = $|jLength \times secondSpacing|$ [S1, unofficial].
5.  **Calculate IJKIndex:** Round the dot product of the transformed `IJKIndex` vector with `axisInIJK` to get the integer index [S1, unofficial].
6.  **Identify Directions:** Identify `iDirection`, `jDirection`, `kDirection` from `[firstDirection, secondDirection, axisInIJK]` by finding the vector with the highest absolute dot product with base vectors $[1,0,0]$, $[0,1,0]$, $[0,0,1]$ respectively (threshold > 0.9) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extractPerpendicularPlane(RASIndex, axis, RASDimensions, spacing, inverseMatrix, volume, isSegmentationMode=False):
    # Initialize vectors
    axisInIJK = [0, 0, 0]
    firstDirection = [0, 0, 0]
    secondDirection = [0, 0, 0]
    planeMatrix = identity_matrix() # 4x4
    dimensions = RASDimensions # (xLength, yLength, zLength)
    
    # Set directions and spacings based on axis
    if axis == 'x':
        axisInIJK = [1, 0, 0]
        firstDirection = [0, 0, -1]
        secondDirection = [0, -1, 0]
        firstSpacing = spacing[2] # spacing[z]
        secondSpacing = spacing[1] # spacing[y]
        IJKIndex = [RASIndex, 0, 0]
        rotate_matrix(planeMatrix, PI/2, [0, 1, 0]) # Rotate around Y
        positionOffset = (RASDimensions[0] - 1) / 2
        set_matrix_position(planeMatrix, [RASIndex - positionOffset, 0, 0])
    elif axis == 'y':
        axisInIJK = [0, 1, 0]
        firstDirection = [1, 0, 0]
        secondDirection = [0, 0, 1]
        firstSpacing = spacing[0] # spacing[x]
        secondSpacing = spacing[2] # spacing[z]
        IJKIndex = [0, RASIndex, 0]
        rotate_matrix(planeMatrix, -PI/2, [1, 0, 0]) # Rotate around X
        positionOffset = (RASDimensions[1] - 1) / 2
        set_matrix_position(planeMatrix, [0, RASIndex - positionOffset, 0])
    else: # axis == 'z'
        axisInIJK = [0, 0, 1]
        firstDirection = [1, 0, 0]
        secondDirection = [0, -1, 0]
        firstSpacing = spacing[0] # spacing[x]
        secondSpacing = spacing[1] # spacing[y]
        IJKIndex = [0, 0, RASIndex]
        positionOffset = (RASDimensions[2] - 1) / 2
        set_matrix_position(planeMatrix, [0, 0, RASIndex - positionOffset])
        
    # Transform directions if not in segmentation mode
    if not isSegmentationMode:
        firstDirection = normalize(transform_vector(inverseMatrix, firstDirection))
        secondDirection = normalize(transform_vector(inverseMatrix, secondDirection))
        axisInIJK = normalize(transform_vector(inverseMatrix, axisInIJK))
        
    # Assign arglets
    firstDirection['arglet'] = 'i'
    secondDirection['arglet'] = 'j'
    
    # Calculate lengths and dimensions
    iLength = floor(abs(dot(firstDirection, dimensions)))
    jLength = floor(abs(dot(secondDirection, dimensions)))
    planeWidth = abs(iLength * firstSpacing)
    planeHeight = abs(jLength * secondSpacing)
    
    # Calculate IJKIndex
    # Note: The source says "round the dot product of the transformed IJKIndex vector with axisInIJK"
    # Assuming IJKIndex is already in the correct space or needs transformation if not segmentation mode
    # If not segmentation mode, IJKIndex might need to be transformed or calculated differently.
    # Based on the source, it seems IJKIndex is used directly or transformed.
    # Let's assume IJKIndex is in the space of axisInIJK for the dot product.
    # If not segmentation mode, IJKIndex might need to be transformed by inverseMatrix?
    # The source says "Calculate IJKIndex by rounding the dot product of the transformed IJKIndex vector with axisInIJK."
    # This implies IJKIndex was transformed in the previous step if not segmentation mode.
    # However, IJKIndex was set as [RASIndex, 0, 0] etc. which is in RAS space.
    # If not segmentation mode, we transformed directions, but not IJKIndex explicitly in the steps.
    # Let's assume IJKIndex is also transformed if not segmentation mode.
    if not isSegmentationMode:
        IJKIndex = transform_vector(inverseMatrix, IJKIndex)
        
    IJKIndex_val = round(dot(IJKIndex, axisInIJK))
    
    # Identify directions
    base_vectors = [[1, 0, 0], [0, 1, 0], [0, 0, 1]]
    directions_list = [firstDirection, secondDirection, axisInIJK]
    
    iDirection = None
    jDirection = None
    kDirection = None
    
    for idx, direction in enumerate(directions_list):
        for base_idx, base in enumerate(base_vectors):
            if base_idx == 0 and iDirection is None and abs(dot(direction, base)) > 0.9:
                iDirection = direction
            elif base_idx == 1 and jDirection is None and abs(dot(direction, base)) > 0.9:
                jDirection = direction
            elif base_idx == 2 and kDirection is None and abs(dot(direction, base)) > 0.9:
                kDirection = direction
                
    # Define sliceAccess function
    def sliceAccess(i, j):
        # Determine si, sj, sk based on direction and arglets
        # This part is complex and depends on how arglets and directions map to IJK indices
        # The source says: "It determines si, sj, sk based on whether the direction matches axisInIJK or uses arglets."
        # And "It then inverts indices if the direction dot product with base vectors is negative."
        
        # Simplified logic based on source description:
        # We need to map 2D slice coordinates (i, j) to 3D volume indices (accessI, accessJ, accessK)
        
        # Let's assume i corresponds to firstDirection and j to secondDirection
        # And axisInIJK is the constant index
        
        # Calculate the offset in IJK space
        # This is a placeholder for the complex mapping logic described
        # The source does not give explicit formulas for si, sj, sk
        
        # Placeholder for the actual mapping logic
        accessI = 0
        accessJ = 0
        accessK = 0
        
        # Determine which direction corresponds to which axis
        # This requires checking the dot products of firstDirection and secondDirection with base vectors
        
        # If firstDirection is aligned with I axis
        if abs(dot(firstDirection, [1, 0, 0])) > 0.9:
            accessI = i
            if dot(firstDirection, [1, 0, 0]) < 0:
                accessI = -i # Invert if negative
        # If firstDirection is aligned with J axis
        elif abs(dot(firstDirection, [0, 1, 0])) > 0.9:
            accessJ = i
            if dot(firstDirection, [0, 1, 0]) < 0:
                accessJ = -i
        # If firstDirection is aligned with K axis
        elif abs(dot(firstDirection, [0, 0, 1])) > 0.9:
            accessK = i
            if dot(firstDirection, [0, 0, 1]) < 0:
                accessK = -i
                
        # Same for secondDirection
        if abs(dot(secondDirection, [1, 0, 0])) > 0.9:
            accessI += j
            if dot(secondDirection, [1, 0, 0]) < 0:
                accessI -= j
        elif abs(dot(secondDirection, [0, 1, 0])) > 0.9:
            accessJ += j
            if dot(secondDirection, [0, 1, 0]) < 0:
                accessJ -= j
        elif abs(dot(secondDirection, [0, 0, 1])) > 0.9:
            accessK += j
            if dot(secondDirection, [0, 0, 1]) < 0:
                accessK -= j
                
        # Add the constant IJKIndex
        accessI += IJKIndex[0]
        accessJ += IJKIndex[1]
        accessK += IJKIndex[2]
        
        return volume.access(accessI, accessJ, accessK)
        
    return {
        'iLength': iLength,
        'jLength': jLength,
        'sliceAccess': sliceAccess,
        'matrix': planeMatrix,
        'planeWidth': planeWidth,
        'planeHeight': planeHeight
    }
```

**Edge Cases and Pitfalls:**
-   **Precision:** Rounding errors in `IJKIndex` calculation can lead to sampling the wrong voxel.
-   **Boundaries:** The slice dimensions are calculated based on the volume dimensions, but the actual slice might extend beyond the volume if the plane matrix is not clamped.
-   **Degenerate Input:** If the volume has zero spacing in any dimension, the plane width or height will be zero, leading to degenerate geometry.
-   **Segmentation Mode:** In segmentation mode, the `inverseMatrix` is not applied, which may lead to incorrect orientation if the volume is not axis-aligned in RAS space [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** The computation of the plane matrix and dimensions is $O(1)$ as it involves a fixed number of arithmetic operations and vector transformations. The `sliceAccess` function is also $O(1)$ per call.
-   **Memory Complexity:** The memory usage is $O(1)$ as it only stores a few vectors and matrices.
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** The method scales linearly with the number of slices if called repeatedly for different `RASIndex` values.
-   **Strengths:** Simple and efficient for axis-aligned slices. Provides a direct mapping from 2D slice coordinates to 3D volume indices.
-   **Limitations:** Only supports axis-aligned slices. Requires the volume to be in RAS space or a known transformation matrix.
-   **When to use:** When rendering axis-aligned slices of a volume in RAS space.
-   **When not to use:** When rendering arbitrary oriented slices or when the volume is not in RAS space.

## Background
The method is described in the context of a WebGL volume rendering demo [S1, unofficial]. It is part of a larger system for slice geometry generation in volume rendering.

## Sources
[S1] PeterC-DLS/webgl_volume_demo, https://github.com/PeterC-DLS/webgl_volume_demo, fetched 2026-10-27 (unofficial)
