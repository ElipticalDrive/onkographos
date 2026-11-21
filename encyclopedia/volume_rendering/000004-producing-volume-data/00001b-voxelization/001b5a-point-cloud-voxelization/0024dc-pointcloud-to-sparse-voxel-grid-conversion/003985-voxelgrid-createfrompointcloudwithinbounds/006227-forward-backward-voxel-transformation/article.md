# Forward-Backward Voxel Transformation

Also called: Inverse mapping with forward projection, Two-pass voxel rotation

## Overview
The Forward-Backward Voxel Transformation is a two-pass algorithm designed to transform a voxelized point cloud from a source orientation and position to a destination orientation and position without introducing visual holes [S1, unofficial]. It solves the coverage problem inherent in single-pass voxel transformations by combining a forward projection of source points with an inverse sampling of the destination grid [S1, unofficial]. Unlike single-pass methods that either project points to a grid or sample the grid from source space, this method ensures full coverage by validating every destination voxel against the source data [S1, unofficial]. It is distinct from sibling methods like Vertex Cluster Assignment, which typically handle spatial clustering rather than rigid body transformation of existing voxel data [10352-vertex-cluster-assignment.md].

## How it works
The method operates on a source point cloud represented as a dictionary mapping integer 3D coordinates to colors. It computes a local transformation matrix that aligns the source cloud with a target object's transform. The process consists of two passes: a forward pass to establish the bounding volume and populate initial voxels, and an inverse pass to fill any gaps by sampling the source cloud for every voxel in the destination bounding box.

### Inputs and Outputs
- **Input**: 
  - `sourceCloud`: A dictionary mapping `Vector3Int` (voxel coordinates) to `Color` (or associated data).
  - `objectRootT`: The transform of the target object (game object or scene node).
  - `pCT`: The transform of the source point cloud.
- **Output**: 
  - `alignedCloud`: A dictionary mapping `Vector3Int` to `Color`, representing the transformed voxel grid.

### Data Structures
- **`alignedCloud`**: `Dictionary<Vector3Int, Color>`. Stores the resulting voxel grid. Keys are integer 3D coordinates; values are colors.
- **`sourceCloud`**: `Dictionary<Vector3Int, Color>`. The input voxel data.
- **`Bounds`**: A bounding box structure tracking the minimum and maximum integer coordinates of the transformed points.

### Transformation Matrix Computation
Before the passes, a local transformation matrix is computed to map source coordinates to destination space. This matrix combines the inverse of the object root's rotation and position with the point cloud's rotation and position [S1, unofficial].

1. Compute local rotation:
   $$ R_{local} = R_{object}^{-1} \times R_{cloud} $$
   Where $R_{object}$ is the rotation of the target object and $R_{cloud}$ is the rotation of the source cloud.

2. Compute local position:
   $$ P_{local} = T_{object}^{-1}(P_{cloud}) $$
   Where $T_{object}^{-1}$ is the inverse transform of the target object applied to the cloud's position.

3. Construct the transformation matrix $M$:
   $$ M = \text{TRS}(P_{local}, R_{local}, \mathbf{1}) $$
   Where $\text{TRS}$ constructs a $4 \times 4$ matrix from translation $P_{local}$, rotation $R_{local}$, and uniform scale $\mathbf{1}$.

### Numbered Steps

1. **Initialize**: Create an empty `alignedCloud` dictionary. Initialize a `Bounds` object and a boolean flag `boundsSet` to false.

2. **Pass 1: Forward Projection**:
   - Iterate through each entry `(pos, color)` in `sourceCloud`.
   - Transform the position to destination space:
     $$ P_{transformed} = M \times \begin{bmatrix} pos.x \\ pos.y \\ pos.z \\ 1 \end{bmatrix} $$
     Note: Only the upper $3 \times 4$ part of the matrix is used for point transformation (ignoring perspective).
   - Round the transformed floating-point coordinates to integer voxel coordinates:
     $$ pos_{dest} = \text{Round}(P_{transformed}) $$
     Where $\text{Round}$ rounds each component to the nearest integer.
   - Update the bounding box:
     - If `boundsSet` is false, set `bounds.min` and `bounds.max` to $pos_{dest}$ and set `boundsSet` to true.
     - Otherwise, encapsulate $pos_{dest}$ into `bounds`.
   - Store the color in the output:
     $$ alignedCloud[pos_{dest}] = color $$

3. **Compute Inverse Matrix**:
   - Calculate the inverse of the transformation matrix:
     $$ M^{-1} = M^{-1} $$

4. **Determine Destination Bounding Box**:
   - Round the bounds of the transformed cloud to integer coordinates:
     $$ b_{min} = \text{Round}(bounds.min) $$
     $$ b_{max} = \text{Round}(bounds.max) $$

5. **Pass 2: Inverse Sampling**:
   - Iterate through every voxel coordinate $dest$ in the bounding box defined by $b_{min}$ and $b_{max}$ for x, y, and z axes:
     $$ \text{for } x \text{ from } b_{min}.x \text{ to } b_{max}.x $$
     $$ \text{for } y \text{ from } b_{min}.y \text{ to } b_{max}.y $$
     $$ \text{for } z \text{ from } b_{min}.z \text{ to } b_{max}.z $$
       - Transform the destination voxel back to source space:
         $$ P_{test} = M^{-1} \times \begin{bmatrix} dest.x \\ dest.y \\ dest.z \\ 1 \end{bmatrix} $$
       - Round to integer source coordinates:
         $$ test_{src} = \text{Round}(P_{test}) $$
       - Check if this coordinate exists in the `sourceCloud`:
         - If `sourceCloud` contains `test_{src}`, retrieve its color and assign it to the destination voxel:
           $$ alignedCloud[dest] = sourceCloud[test_{src}] $$
         - If not, the voxel remains empty (or retains its previous value if it was set in Pass 1, though typically Pass 1 sets the "core" and Pass 2 fills gaps; in the reference implementation, Pass 2 overwrites or fills, ensuring no holes).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ForwardBackwardVoxelTransformation(sourceCloud, objectRootT, pCT):
    """
    Transforms a voxel point cloud to align with a target object's transform.
    
    Args:
        sourceCloud: Dictionary mapping Vector3Int to Color.
        objectRootT: Transform of the target object.
        pCT: Transform of the source point cloud.
        
    Returns:
        alignedCloud: Dictionary mapping Vector3Int to Color.
    """
    alignedCloud = {}
    
    # Compute local transformation matrix
    # localRot = Inverse(objectRootT.rotation) * pCT.rotation
    localRot = Quaternion.Inverse(objectRootT.rotation) * pCT.rotation
    # localPos = objectRootT.InverseTransformPoint(pCT.position)
    localPos = objectRootT.InverseTransformPoint(pCT.position)
    
    # Matrix4x4.TRS(localPos, localRot, Vector3.one)
    matrix = Matrix4x4.TRS(localPos, localRot, Vector3.one)
    
    bounds = Bounds()
    boundsSet = False
    
    # Pass 1: Forward Projection
    for pos, color in sourceCloud.items():
        # transformedPos = matrix.MultiplyPoint3x4(pos)
        transformedPos = matrix.MultiplyPoint3x4(pos)
        
        # pos_dest = RoundVector3(transformedPos)
        pos_dest = RoundVector3(transformedPos)
        
        if not boundsSet:
            bounds.min = pos_dest
            bounds.max = pos_dest
            boundsSet = True
        else:
            bounds.Encapsulate(pos_dest)
            
        alignedCloud[pos_dest] = color
        
    # Pass 2: Inverse Sampling
    invMatrix = matrix.inverse
    
    # bMin = RoundVector3(bounds.min)
    # bMax = RoundVector3(bounds.max)
    bMin = RoundVector3(bounds.min)
    bMax = RoundVector3(bounds.max)
    
    for x in range(bMin.x, bMax.x + 1):
        for y in range(bMin.y, bMax.y + 1):
            for z in range(bMin.z, bMax.z + 1):
                dest = Vector3Int(x, y, z)
                
                # testPosFloat = invMatrix.MultiplyPoint3x4(dest)
                testPosFloat = invMatrix.MultiplyPoint3x4(dest)
                
                # testPos = RoundVector3(testPosFloat)
                testPos = RoundVector3(testPosFloat)
                
                if testPos in sourceCloud:
                    alignedCloud[dest] = sourceCloud[testPos]
                    
    return alignedCloud

def RoundVector3(vec):
    """Rounds each component of a Vector3 to the nearest integer."""
    return Vector3Int(
        round(vec.x),
        round(vec.y),
        round(vec.z)
    )
```

### Parameters
- **`VoxelUtil.RoundVector3`**: A utility function that converts floating-point coordinates to integer voxel grid coordinates by rounding each component to the nearest integer [S1, unofficial].

### Edge Cases and Pitfalls
- **Holes**: Using only the forward pass (projecting source points) or only the inverse pass (sampling source for destination) results in visual holes in the rendered cloud [S1, unofficial]. The two-pass approach is necessary to ensure full coverage [S1, unofficial].
- **Precision**: Rounding errors in the inverse transformation may map a destination voxel to a source coordinate that is not exactly represented in the source cloud, leading to missing data if the source cloud is sparse [S1, unofficial].
- **Bounding Box**: The bounding box must be computed from the transformed positions in the forward pass to ensure the inverse pass covers all relevant destination voxels [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N + V)$, where $N$ is the number of points in the source cloud and $V$ is the number of voxels in the destination bounding box. The forward pass iterates over $N$ points, and the inverse pass iterates over $V$ voxels.
- **Memory Complexity**: $O(N + V)$ for storing the `alignedCloud` dictionary. The source cloud is read-only.
- **Performance**: The method is computationally more expensive than single-pass methods due to the two iterations and the overhead of dictionary lookups. However, it provides higher quality results compared to alternative methods like "Rotate by shearing," which may be more efficient but are sensitive to rotation order and yield lower quality (~80%) [S1, unofficial].
- **Strengths**: Ensures no holes in the transformed voxel cloud; robust to rotation and translation.
- **Limitations**: Higher memory and computational cost; sensitive to the precision of the transformation matrix and rounding operations.

## Background
The Forward-Backward Voxel Transformation was described in a 2023 discussion on Game Development Stack Exchange by a developer seeking a cleaner way to rotate and align a voxel point cloud with a game object's transform [S1, unofficial]. The method was presented as a solution to the problem of visual holes that arise when using single-pass voxel rotation techniques. It is inspired by 2D bitmap rotation techniques where inverse mapping is used to ensure full coverage of the destination grid [S1, unofficial]. The method is compared against "Rotate by shearing," a technique suggested by another user (DMGregory) which eliminates hash table overhead but is sensitive to rotation order and yields lower quality [S1, unofficial].

## Sources
[S1] Voxel Point Cloud Transformation (Rotation & Translation), https://gamedev.stackexchange.com/questions/206463/voxel-point-cloud-transformation-rotation-translation, fetched 2026-10-27 (unofficial)
