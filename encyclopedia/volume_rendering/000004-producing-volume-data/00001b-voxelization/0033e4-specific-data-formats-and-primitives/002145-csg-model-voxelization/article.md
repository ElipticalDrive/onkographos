# CSG Model Voxelization

Also called: Basic CSG Voxelization by Frame Buffer Pixel Mapping, Boolean CSG voxelization, Hardware-Assisted Frame Buffer Pixel Mapping for CSG Voxelization

## Overview
CSG Model Voxelization is a hardware-accelerated algorithm that generates a binary voxelization of a Constructive Solid Geometry (CSG) model by encoding primitive classifications into frame buffer pixel colors and using logical operations to compute the final CSG classification implicitly via a Point-Classification Map (PCM) [S1]. It solves the problem of efficiently converting complex CSG trees into discrete volumetric data for applications like volume rendering and layered manufacturing [S1]. Unlike previous hardware-accelerated methods that required intermediate textures for each Boolean operation node, this method computes boolean operations in image space using frame buffer pixel maps, resulting in faster performance and avoiding hardware restrictions on blending function combinations [S1].

## How it works
The method voxelize CSG nodes into worksheets and performs boolean operations in image space using GPU blending [S2, unofficial]. It operates slice-by-slice (Z-plane by Z-plane), rendering primitives into a frame buffer where each bit of the pixel color represents the inclusion of a specific primitive [S1].

### Inputs and Outputs
*   **Input**: A CSG tree with $n$ leaf nodes (primitives). Each primitive is either a geometric boundary surface or a volume data set.
*   **Output**: A binary volume representation stored in texture memory, where each voxel indicates whether it lies inside the final CSG model.

### Data Structures
1.  **Frame Buffer**: Stores the classification index for each pixel. The classification index is an $n$-bit binary number where the $j$-th bit is 1 if the point is inside the $j$-th primitive, and 0 otherwise [S1].
2.  **Point-Classification Map (PCM)**: A hardware-implemented look-up table (color look-up table) that maps the classification index (pixel color) to the final CSG classification (binary in/out) [S1]. The PCM has $2^n$ entries [S1].
3.  **Primitive Color ($C_j$)**: For the $j$-th primitive, the color is a binary number with the $j$-th bit set to 1 and all other bits set to 0 [S1].

### Algorithm Steps
1.  **Initialization**:
    *   Generate cumulative transformation matrices for each primitive [S1].
    *   Define texture objects for all volume primitives [S1].
    *   Assign color codes to all primitives: the $j$-th primitive's color is a binary number with the $j$-th bit set to 1 [S1].
    *   Compute the PCM from the CSG root node using `CSG_PCM` (see below) [S1].
    *   Clear the frame buffer [S1].

2.  **Loop over Z-planes (Slices)**:
    *   Define the thin viewing volume between two adjacent Z-planes inside the volume boundary [S1].
    *   **Loop over primitives**:
        *   Define the system transformation matrix for the current primitive [S1].
        *   **If geometric primitive**:
            *   Set the logical operation to **XOR** [S1].
            *   Set the primitive's color ($C_j$) [S1].
            *   Render the boundary surfaces of the primitive. The clipping hardware ensures only parts within the thin viewing volume are displayed [S1]. The XOR operation toggles the $j$-th bit: entering the primitive sets the bit to 1, exiting sets it to 0 [S1].
        *   **If volume data set primitive**:
            *   Clear the frame buffer's color bit for this primitive (the $j$-th bit) for all pixels. This cannot be done by clearing the entire frame buffer, as bits for other primitives must be retained [S1].
            *   Bind the 3D texture object defined by the volume data set [S1].
            *   Set a color look-up table (CLUT) to map intensity values within the threshold interval $[s, t]$ to the primitive's color ($C_j$) and other values to 0 [S1].
            *   Set the logical operation to **OR** [S1].
            *   Draw the Z-plane using 3D texture mapping [S1].
    *   **Post-processing for the slice**:
        *   Define the frame buffer pixel map using the computed PCM [S1].
        *   Copy the frame buffer to texture memory. The PCM is automatically applied during the copy operation, mapping the classification index to the final binary CSG classification [S1].

### Point-Classification Map Construction
The PCM is constructed recursively from the CSG tree [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CSG_PCM(CSG_NODE N):
    """
    Computes the Point-Classification Map for a CSG node.
    Returns a look-up table mapping n-bit classification indices to binary CSG classification.
    """
    if N is a leaf node:
        # For a leaf node (primitive), the PCM maps the primitive's specific bit
        # to the final classification. Since it's a leaf, the classification
        # is simply the value of that bit (assuming no negation at leaf level
        # in the basic description, though CSG trees can have NOT).
        # The source implies returning the PCM of the primitive.
        # In the context of the bit encoding, if the node is a simple leaf,
        # the map is identity for that bit's contribution relative to the tree structure.
        # However, the source describes combining PCM_left and PCM_right.
        # For a leaf, it effectively passes through the bit.
        return the PCM of the primitive; 
    else:
        PCM_left = CSG_PCM(N.left)
        PCM_right = CSG_PCM(N.right)
        PCM_combined = combine(PCM_left, PCM_right, N.op)
        return PCM_combined
```
*Note: The source [S1] provides the pseudocode structure but does not detail the `combine` function's internal logic for arbitrary boolean operators (AND, OR, XOR) on the look-up tables, other than stating it combines them based on the node's operation [S1].*

### Parameters
*   **$n$**: The number of primitives in the CSG tree. The algorithm assumes $n$ distinct colors are assigned [S1].
*   **Primitive Color ($C_j$)**: The color assigned to the $j$-th primitive is the $j$-th color bit if the point is inside the primitive, and 0 otherwise [S1].

### Edge Cases and Pitfalls
*   **Frame Buffer Bit Depth Limitation**: Standard RGBA frame buffers have 8 bits per component (32 bits total). The basic algorithm is limited to 8 primitives if using only one color component (e.g., Red) because the pixel map only has separate look-up tables for individual RGBA components, not cross combinations [S1].
*   **Frame Buffer Depth Limitation**: The basic algorithm assumes each pixel has as many bits as the number of primitives. If $n > 8$ (or the available bit depth), the algorithm must be modified, e.g., using a multi-pass approach [S1].
*   **Volume Primitive Bit Clearing**: For volume primitives, the specific bit must be cleared before rendering to ensure correct classification. Simply clearing the frame buffer is incorrect as it destroys bits for other primitives [S1].

## Subtopics

### [CSG and Inter-object Intersection](4703-csg-and-inter-object-intersection.md)
Performs general CSG operations by voxelizing each object into a separate slicemap and rendering them sequentially with blending to compute the desired boolean operation. It differs by explicitly leveraging density computation capabilities for collision detection and haptic feedback.

### [SDF-based Voxelization](6595-sdf-based-voxelization.md)
Visualizes CSG meshes or analytic primitives as voxel chunks by sampling a 3D grid and querying Signed Distance Fields. It differs by relying on CPU-side SDF queries and storing results in a packed bitmap, enabling A/B comparison with the original mesh.

### [Sequential CSG](7629-sequential-csg.md)
A CPU-based algorithm for boolean operations on two voxel grids using word-level parallelism. It differs by operating entirely on the host, serving as a baseline or fallback when hardware acceleration is unavailable.

## Operating characteristics
*   **Time Complexity**: The algorithm is hardware-accelerated and interactive. It avoids the generation of intermediate textures for each Boolean operation node, which was a bottleneck in previous methods [S1].
*   **Memory Complexity**: Requires frame buffer storage for classification indices and texture memory for the final volume. The PCM size is $2^n$ entries [S1].
*   **Performance**: Faster than previous hardware-accelerated CSG voxelization methods that used blending functions and intermediate textures [S1].
*   **Limitations**: Limited by frame buffer bit depth (typically 8 primitives per pass using one color component) [S1]. Requires graphics hardware support for frame buffer pixel maps, logical operations, and 3D texture mapping [S1].
*   **When to use**: When interactive voxelization of CSG models is required for volume rendering or layered manufacturing, and the number of primitives is small enough to fit within frame buffer bit depth constraints, or a multi-pass approach is acceptable [S1].

## Background
The method was presented in a paper titled "Fast CSG voxelization by frame buffer pixel mapping" [S1]. It builds upon previous work [6] which used blending functions and intermediate textures, aiming to overcome performance limitations and hardware restrictions on blending function combinations [S1]. The algorithm is applicable to any regions of interest, providing a multi-resolution rendering solution through dynamic voxelization [S1].

## Sources
[S1] Fast CSG voxelization by frame buffer pixel mapping, https://doi.org/10.1145/353888.353896, fetched 2026-11-04
[S2] PDF Real-time Voxelization for Complex Models - GitHub Pages, https://wo1fsea.github.io/files/About_NavMesh/P2_Real-time_Voxelization_for_Complex_Models.pdf, fetched 2026-10-29 (unofficial)
[S3] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
