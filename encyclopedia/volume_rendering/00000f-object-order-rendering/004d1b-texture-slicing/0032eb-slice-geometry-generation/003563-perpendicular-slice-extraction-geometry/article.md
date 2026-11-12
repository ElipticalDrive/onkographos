# Perpendicular Slice Extraction Geometry

Also called: Slice plane matrix calculation, extractPerpendicularPlane

## Overview
Perpendicular Slice Extraction Geometry computes the geometric properties of a slice perpendicular to a specified anatomical axis (x, y, or z) at a specific RAS index. It returns the slice dimensions, a plane matrix defining orientation and position in RAS space, and a `sliceAccess` function to map 2D slice coordinates to 3D volume data indices. It differs from [Plane Slicing](845-plane-slicing.md) by enforcing strict alignment with the primary axes and handling the coordinate transformation between RAS (Right-Anterior-Superior) and IJK (voxel) spaces explicitly through rotation and offset calculations.

## How it works
**Inputs:**
- `RASIndex`: The index along the chosen axis in RAS space.
- `axis`: The axis of extraction ('x', 'y', or 'z').
- `RASDimensions`: The dimensions of the volume in RAS space $(D_x, D_y, D_z)$.
- `spacing`: The voxel spacing $(s_x, s_y, s_z)$.
- `inverseMatrix`: The transformation matrix from IJK to RAS (or vice versa, depending on convention; here used to transform directions).
- `volume`: The volume object providing an `access(i, j, k)` method.
- `segmentationMode`: Boolean flag indicating if the volume is a segmentation mask.

**Outputs:**
- A dictionary/object containing:
  - `iLength`: Width of the slice in voxels.
  - `jLength`: Height of the slice in voxels.
  - `matrix`: The $4 \times 4$ plane matrix (orientation/position).
  - `planeWidth`: Physical width of the slice.
  - `planeHeight`: Physical height of the slice.
  - `sliceAccess`: A function mapping $(i, j)$ to volume data.

**Data Structures:**
- `axisInIJK`: Vector $(1,0,0)$, $(0,1,0)$, or $(0,0,1)$ corresponding to the axis.
- `firstDirection`, `secondDirection`: Vectors defining the local 2D axes of the slice.
- `planeMatrix`: $4 \times 4$ identity matrix initially, modified by rotation and translation.
- `dimensions`: Vector $(xLength, yLength, zLength)$ representing the volume extent.

**Steps:**

1.  **Initialize Vectors and Matrix:**
    Set `axisInIJK`, `firstDirection`, `secondDirection`, `planeMatrix` (identity), and `dimensions`.

2.  **Configure Axis-Specific Parameters:**
    Based on `axis`:
    -   **If 'x':**
        -   `axisInIJK` = $(1, 0, 0)$
        -   `firstDirection` = $(0, 0, -1)$
        -   `secondDirection` = $(0, -1, 0)$
        -   `firstSpacing` = $s_z$
        -   `secondSpacing` = $s_y$
        -   `IJKIndex` = $(RASIndex, 0, 0)$
        -   Rotate `planeMatrix` by $\pi/2$ around the Y-axis.
        -   `positionOffset` = $(D_x - 1) / 2$
        -   Set `planeMatrix` position to $(RASIndex - positionOffset, 0, 0)$.
    -   **If 'y':**
        -   `axisInIJK` = $(0, 1, 0)$
        -   `firstDirection` = $(1, 0, 0)$
        -   `secondDirection` = $(0, 0, 1)$
        -   `firstSpacing` = $s_x$
        -   `secondSpacing` = $s_z$
        -   `IJKIndex` = $(0, RASIndex, 0)$
        -   Rotate `planeMatrix` by $-\pi/2$ around the X-axis.
        -   `positionOffset` = $(D_y - 1) / 2$
        -   Set `planeMatrix` position to $(0, RASIndex - positionOffset, 0)$.
    -   **If 'z' (default):**
        -   `axisInIJK` = $(0, 0, 1)$
        -   `firstDirection` = $(1, 0, 0)$
        -   `secondDirection` = $(0, -1, 0)$
        -   `firstSpacing` = $s_x$
        -   `secondSpacing` = $s_y$
        -   `IJKIndex` = $(0, 0, RASIndex)$
        -   `positionOffset` = $(D_z - 1) / 2$
        -   Set `planeMatrix` position to $(0, 0, RASIndex - positionOffset)$.

3.  **Transform Directions (Non-Segmentation Mode):**
    If `segmentationMode` is false:
    -   Apply `inverseMatrix` to `firstDirection`, `secondDirection`, and `axisInIJK`.
    -   Normalize the resulting vectors.

4.  **Calculate Dimensions and Physical Size:**
    -   Assign `firstDirection.arglet = 'i'` and `secondDirection.arglet = 'j'`.
    -   Calculate `iLength` = $\lfloor | \text{firstDirection} \cdot \text{dimensions} | \rfloor$.
    -   Calculate `jLength` = $\lfloor | \text{secondDirection} \cdot \text{dimensions} | \rfloor$.
    -   Calculate `planeWidth` = $| \text{iLength} \times \text{firstSpacing} |$.
    -   Calculate `planeHeight` = $| \text{jLength} \times \text{secondSpacing} |$.

5.  **Refine IJK Index:**
    -   Calculate the new `IJKIndex` by rounding the dot product of the transformed `IJKIndex` vector with `axisInIJK`.

6.  **Identify Directional Components:**
    -   Identify `iDirection`, `jDirection`, `kDirection` from the list `[firstDirection, secondDirection, axisInIJK]`.
    -   Match them to base vectors $[1,0,0]$, $[0,1,0]$, $[0,0,1]$ by finding the vector with the highest absolute dot product (threshold $> 0.9$).

7.  **Construct Slice Access Function:**
    -   Define `sliceAccess(i, j)` to map 2D coordinates to 3D volume indices.
    -   Determine `si`, `sj`, `sk` based on whether the direction matches `axisInIJK` or uses arglets.
    -   Invert indices if the direction dot product with base vectors is negative.
    -   Call `volume.access(accessI, accessJ, accessK)`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extractPerpendicularPlane(RASIndex, axis, RASDimensions, spacing, inverseMatrix, volume, segmentationMode):
    # 1. Initialize
    axisInIJK = [0, 0, 0]
    firstDirection = [0, 0, 0]
    secondDirection = [0, 0, 0]
    planeMatrix = identity_4x4()
    dimensions = RASDimensions # [xLength, yLength, zLength]
    
    # 2. Switch on axis
    if axis == 'x':
        axisInIJK = [1, 0, 0]
        firstDirection = [0, 0, -1]
        secondDirection = [0, -1, 0]
        firstSpacing = spacing[2] # spacing[z]
        secondSpacing = spacing[1] # spacing[y]
        IJKIndex = [RASIndex, 0, 0]
        # Rotate planeMatrix by PI/2 around Y
        planeMatrix = rotate_y(planeMatrix, PI / 2)
        positionOffset = (RASDimensions[0] - 1) / 2
        planeMatrix.set_position(RASIndex - positionOffset, 0, 0)
        
    elif axis == 'y':
        axisInIJK = [0, 1, 0]
        firstDirection = [1, 0, 0]
        secondDirection = [0, 0, 1]
        firstSpacing = spacing[0] # spacing[x]
        secondSpacing = spacing[2] # spacing[z]
        IJKIndex = [0, RASIndex, 0]
        # Rotate planeMatrix by -PI/2 around X
        planeMatrix = rotate_x(planeMatrix, -PI / 2)
        positionOffset = (RASDimensions[1] - 1) / 2
        planeMatrix.set_position(0, RASIndex - positionOffset, 0)
        
    else: # axis == 'z'
        axisInIJK = [0, 0, 1]
        firstDirection = [1, 0, 0]
        secondDirection = [0, -1, 0]
        firstSpacing = spacing[0] # spacing[x]
        secondSpacing = spacing[1] # spacing[y]
        IJKIndex = [0, 0, RASIndex]
        positionOffset = (RASDimensions[2] - 1) / 2
        planeMatrix.set_position(0, 0, RASIndex - positionOffset)

    # 3. Transform Directions if not segmentation mode
    if not segmentationMode:
        firstDirection = normalize(multiply_vector_matrix(firstDirection, inverseMatrix))
        secondDirection = normalize(multiply_vector_matrix(secondDirection, inverseMatrix))
        axisInIJK = normalize(multiply_vector_matrix(axisInIJK, inverseMatrix))

    # 4. Calculate Dimensions
    firstDirection.arglet = 'i'
    secondDirection.arglet = 'j'
    
    iLength = floor(abs(dot(firstDirection, dimensions)))
    jLength = floor(abs(dot(secondDirection, dimensions)))
    
    planeWidth = abs(iLength * firstSpacing)
    planeHeight = abs(jLength * secondSpacing)

    # 5. Refine IJKIndex
    # "Calculate IJKIndex by rounding the dot product of the transformed IJKIndex vector with axisInIJK"
    # Note: The source implies IJKIndex might have been transformed in step 3 if not segmentation mode.
    # Assuming IJKIndex is in the same space as axisInIJK after step 3.
    IJKIndex = round(dot(IJKIndex, axisInIJK)) # This results in a scalar, but IJKIndex is a vector.
    # Correction based on source logic: The source says "Calculate IJKIndex...". 
    # Given the context of "IJKIndex = (RASIndex, 0, 0)" initially, and axisInIJK being a unit vector,
    # the dot product extracts the component along the axis.
    # However, the source says "IJKIndex" is a vector in the return/access. 
    # Let's assume the source meant to clamp or set the specific index component.
    # Re-reading: "Calculate IJKIndex by rounding the dot product...". 
    # This likely sets the scalar index value for the slice position.
    sliceIndex = round(dot(IJKIndex, axisInIJK))

    # 6. Identify Directions
    baseVectors = [[1,0,0], [0,1,0], [0,0,1]]
    directionsList = [firstDirection, secondDirection, axisInIJK]
    
    iDirection = None
    jDirection = None
    kDirection = None
    
    # Find iDirection: vector with highest abs dot with [1,0,0]
    max_dot_i = -1
    for d in directionsList:
        val = abs(dot(d, baseVectors[0]))
        if val > max_dot_i:
            max_dot_i = val
            iDirection = d
            
    # Find jDirection: vector with highest abs dot with [0,1,0]
    max_dot_j = -1
    for d in directionsList:
        val = abs(dot(d, baseVectors[1]))
        if val > max_dot_j:
            max_dot_j = val
            jDirection = d
            
    # Find kDirection: vector with highest abs dot with [0,0,1]
    max_dot_k = -1
    for d in directionsList:
        val = abs(dot(d, baseVectors[2]))
        if val > max_dot_k:
            max_dot_k = val
            kDirection = d

    # 7. Define sliceAccess
    def sliceAccess(i, j):
        # Determine si, sj, sk
        # This part is complex in the source description. 
        # "It determines si, sj, sk based on whether the direction matches axisInIJK or uses arglets."
        # "It then inverts indices if the direction dot product with base vectors is negative."
        
        # Helper to get component from direction
        def get_component(vec, base_idx):
            return vec[base_idx]

        # Map i (firstDirection) to IJK
        # If firstDirection is axisInIJK, it's the slice index.
        # Otherwise, it contributes to one of the other axes.
        
        # Simplified logic based on "arglet" and "inverts indices":
        accessI = 0
        accessJ = 0
        accessK = 0
        
        # Set the slice index
        # The source says "IJKIndex" is calculated. Let's assume sliceIndex is the coordinate along axisInIJK.
        # We need to map i, j to the other two axes.
        
        # Identify which axis corresponds to 'i' and 'j' in IJK space
        # iDirection corresponds to IJK axis 0, 1, or 2?
        # jDirection corresponds to IJK axis 0, 1, or 2?
        
        # Let's find the IJK index for iDirection
        i_axis = -1
        for k, base in enumerate(baseVectors):
            if abs(dot(iDirection, base)) > 0.9:
                i_axis = k
                break
        
        j_axis = -1
        for k, base in enumerate(baseVectors):
            if abs(dot(jDirection, base)) > 0.9:
                j_axis = k
                break
                
        axis_idx = -1
        for k, base in enumerate(baseVectors):
            if abs(dot(axisInIJK, base)) > 0.9:
                axis_idx = k
                break
                
        # Initialize access indices
        access = [0, 0, 0]
        access[axis_idx] = sliceIndex
        
        # Assign i and j
        # If iDirection dot base is negative, invert index?
        # "Inverts indices if the direction dot product with base vectors is negative"
        
        # Determine sign for i
        i_sign = 1
        for k, base in enumerate(baseVectors):
            if abs(dot(iDirection, base)) > 0.9:
                if dot(iDirection, base) < 0:
                    i_sign = -1
                break
        access[i_axis] = i_sign * i
        
        # Determine sign for j
        j_sign = 1
        for k, base in enumerate(baseVectors):
            if abs(dot(jDirection, base)) > 0.9:
                if dot(jDirection, base) < 0:
                    j_sign = -1
                break
        access[j_axis] = j_sign * j
        
        return volume.access(access[0], access[1], access[2])

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
-   **Precision:** Rounding errors in `IJKIndex` calculation may lead to sampling the wrong slice if the RAS index is not perfectly aligned with voxel centers.
-   **Boundaries:** `iLength` and `jLength` are floored. If the volume is small or the orientation is oblique (though this method is for perpendicular slices, the transformation might stretch the effective sampling area), the slice might extend beyond the volume bounds. The `volume.access` function should handle out-of-bounds checks.
-   **Degenerate Input:** If `spacing` contains zeros, `planeWidth` or `planeHeight` will be zero, leading to degenerate slices.

## Subtopics

### Plane Slicing
-   **What it is:** A general method for extracting slices along arbitrary planes defined by a normal vector and a point.
-   **Difference:** Unlike Perpendicular Slice Extraction, which is restricted to axes-aligned cuts, Plane Slicing allows for oblique angles.
-   **When to choose:** When oblique views are required.
-   **Link:** [Plane Slicing](845-plane-slicing.md)

### GPU-based Cut-off Slice Generation
-   **What it is:** A method that generates slices on the GPU, often using cut-off planes to clip the volume.
-   **Difference:** Focuses on GPU acceleration and clipping logic rather than the specific geometric calculation of perpendicular axes in CPU-side data structures.
-   **When to choose:** For real-time rendering where slice generation is a bottleneck.
-   **Link:** [GPU-based Cut-off Slice Generation](948-gpu-based-cut-off-slice-generation.md)

### Sliced Volume Rendering
-   **What it is:** A rendering technique that renders volumes by compositing multiple slices.
-   **Difference:** This is a rendering strategy, whereas Perpendicular Slice Extraction Geometry is a geometric utility for generating the slice data.
-   **When to choose:** When using slice-based rendering (e.g., ray casting via slices) instead of ray marching.
-   **Link:** [Sliced Volume Rendering](3568-sliced-volume-rendering.md)

## Operating characteristics
-   **Time Complexity:** $O(1)$ for the geometric calculation. The `sliceAccess` function is $O(1)$ per call.
-   **Memory Complexity:** $O(1)$ for the returned structure.
-   **Strengths:** Simple, fast, and robust for standard orthogonal views (axial, coronal, sagittal).
-   **Limitations:** Only supports axis-aligned slices. Not suitable for oblique reformatting.
-   **When to use:** Standard medical imaging viewers, orthogonal cross-sections.
-   **When not:** Oblique views, curved planar reformatting.

## Background
The method originates from the `webgl_volume_demo` project by PeterC-DLS. It is designed to facilitate the extraction of orthogonal slices from 3D volumes in web-based volume rendering applications, handling the complex coordinate transformations between RAS and IJK spaces.

## Sources
[S1] PeterC-DLS/webgl_volume_demo, https://github.com/PeterC-DLS/webgl_volume_demo, fetched 2026-10-27 (unofficial)
