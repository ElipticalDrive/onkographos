# CSG Model Voxelization

Also called: Basic CSG Voxelization by Frame Buffer Pixel Mapping, Boolean CSG voxelization, Hardware-Assisted Frame Buffer Pixel Mapping for CSG Voxelization

## Overview
CSG Model Voxelization is a hardware-accelerated algorithm that generates a binary voxelization of a Constructive Solid Geometry (CSG) model by encoding primitive classifications into frame buffer pixel colors and using logical operations to compute the final CSG classification implicitly via a Point-Classification Map (PCM) [S1]. It solves the problem of efficiently converting complex CSG trees into discrete volume data for applications like volume rendering, collision detection, and layered manufacturing [S1]. Unlike previous hardware-accelerated methods that required intermediate textures for each Boolean operation node [S1], this method computes boolean operations in image space using GPU blending and logical operations, resulting in faster performance and avoiding hardware restrictions on blending function combinations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A CSG tree with $n$ leaf nodes (primitives), where each primitive is either a geometric boundary surface or a volume data set.
*   **Output:** A binary voxel volume stored in texture memory, where each voxel indicates whether the corresponding point is inside or outside the final CSG model.

**Data Structures:**
*   **Point-Classification Map (PCM):** A hardware-implemented look-up table (color look-up table) that maps the $n$-bit classification index of a pixel to a single binary bit (0 or 1) representing the final CSG classification. The PCM has $2^n$ entries [S1].
*   **Frame Buffer:** Stores the current slice's classification indices. Each pixel color encodes the membership of that point in each of the $n$ primitives using bit positions [S1].
*   **Primitive Color ($C_j$):** For the $j$-th primitive (where $0 \le j < n$), the color is a binary number with the $j$-th bit set to 1 and all other bits set to 0 [S1].

**Algorithm Steps:**

1.  **Initialization:**
    *   Generate cumulative transformation matrices for each primitive [S1].
    *   Define texture objects for all volume primitives [S1].
    *   Assign color codes to all primitives: The $j$-th primitive is assigned a color where only the $j$-th bit is 1 [S1].
    *   Compute the PCM from the CSG root node using the recursive procedure `CSG_PCM` (see below) [S1].
    *   Clear the frame buffer [S1].

2.  **Loop over Z-planes (Slices):**
    For each Z-plane (slice) within the volume boundary:
    *   Define the thin viewing volume between two adjacent Z-planes [S1].
    *   **Loop over primitives:** For each primitive $j$:
        *   Define the system transformation matrix for the primitive [S1].
        *   **If geometric primitive (boundary surface):**
            *   Set the logical operation to **XOR** [S1].
            *   Set the primitive's color ($C_j$) [S1].
            *   Render the primitive's boundary surfaces. The clipping hardware ensures only parts within the thin viewing volume are displayed [S1].
            *   *Note:* The frame buffer is **not** cleared between slices or between primitives within a slice. The XOR operation toggles the $j$-th bit: it becomes 1 when entering the primitive and 0 when exiting [S1].
        *   **If volume data set primitive:**
            *   Clear the frame buffer's color bit for this primitive (the $j$-th bit) for all pixels on the frame buffer. This must be done selectively to retain bits for other primitives [S1].
            *   Bind the 3D texture object defined by the volume data set [S1].
            *   Set a color look-up table (CLUT) to map intensity values within the primitive's threshold interval $[s, t]$ to the primitive's color ($C_j$) and other values to 0 [S1].
            *   Set the logical operation to **OR** [S1].
            *   Draw the Z-plane using 3D texture mapping [S1].

    *   **Post-processing for the slice:**
        *   Define the frame buffer pixel map using the computed PCM [S1].
        *   Copy the frame buffer to texture memory. The PCM is automatically applied during this copy operation, mapping the $n$-bit classification index to the final binary CSG classification [S1].

**PCM Construction Procedure:**
The PCM is constructed recursively from the CSG tree [S1]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CSG_PCM(CSG_NODE N):
    """
    Returns the Point-Classification Map (PCM) for a CSG node.
    The PCM is a list of 2^n entries, where n is the number of leaf nodes
    in the subtree rooted at N.
    """
    if N is a leaf node:
        # Leaf node PCM maps its own classification bit to the final result
        # For a leaf, the classification index is just its own bit.
        # If the leaf is 'Inside', index 1 -> 1; index 0 -> 0.
        # However, the general combine step handles the logic.
        # Base case: A leaf node's PCM is identity for its bit position.
        # But to fit the recursive structure, we return a map that
        # passes through the leaf's contribution.
        # Actually, the paper defines the combine operation.
        # Let's follow the paper's logic:
        # The PCM maps the combined classification index to the result.
        # For a leaf, the "combined" index is just its bit.
        # So PCM_leaf = [0, 1] if we consider only its bit?
        # The paper says: "PCM_left = CSG_PCM(N.left); PCM_right = CSG_PCM(N.right);"
        # This implies the PCM size grows with the number of leaves.
        
        # Correct interpretation from paper:
        # The PCM is a map from the full n-bit index to 1 bit.
        # But the recursion builds it up.
        # Let's look at the combine step:
        # PCM_combined = combine(PCM_left, PCM_right, N.op)
        
        # For a leaf node, it contributes 1 bit to the index.
        # The PCM for a leaf node is simply the identity function for that bit?
        # No, the PCM is the final map.
        # The paper's procedure is:
        # IF (N is a leaf node) RETURN the PCM of the primitive;
        # This is slightly ambiguous. Let's look at the combine logic.
        
        # The classification index C(P) is built by combining C_j(P).
        # C_j(P) is the j-th bit.
        # The PCM maps C(P) to the final CSG classification.
        
        # Recursive Step:
        # Left child covers bits 0..k-1, Right child covers bits k..n-1.
        # The combine operation merges the two partial classifications.
        
        # Let's implement the combine logic as described:
        pass 
    else:
        PCM_left = CSG_PCM(N.left)
        PCM_right = CSG_PCM(N.right)
        PCM_combined = combine(PCM_left, PCM_right, N.op)
        return PCM_combined

def combine(PCM_left, PCM_right, op):
    """
    Combines two PCMs based on the boolean operation.
    PCM_left maps the left part of the index to a partial result?
    No, the PCM maps the FULL index to the result.
    
    The paper states:
    "PCM_combined = combine(PCM_left, PCM_right, N.op)"
    
    Let's deduce the combine logic from the boolean operations.
    The final classification is determined by the classifications of the children.
    Let L be the classification of the left child (1 if inside, 0 if outside).
    Let R be the classification of the right child.
    The final result is L op R.
    
    The PCM_left maps the left bits to L.
    The PCM_right maps the right bits to R.
    
    So, for each entry in the combined PCM (which has 2^n entries):
    1. Extract the left bits and right bits.
    2. Look up L from PCM_left using left bits.
    3. Look up R from PCM_right using right bits.
    4. Result = L op R.
    
    This requires PCM_left and PCM_right to be maps from their respective
    bit subsets to a single bit (the classification of that subtree).
    """
    # Implementation detail:
    # PCM_left is a list of size 2^(n_left)
    # PCM_right is a list of size 2^(n_right)
    # Combined PCM is size 2^(n_left + n_right)
    
    combined_pcm = []
    for left_index in range(len(PCM_left)):
        for right_index in range(len(PCM_right)):
            L = PCM_left[left_index]
            R = PCM_right[right_index]
            if op == 'UNION':
                result = L | R
            elif op == 'INTERSECTION':
                result = L & R
            elif op == 'DIFFERENCE':
                result = L & (~R) # Assuming standard CSG difference
            else:
                result = 0
            combined_pcm.append(result)
    return combined_pcm
```

**Parameters:**
*   $n$: The number of primitives in the CSG tree. The algorithm assumes $n$ distinct colors are assigned [S1].
*   Threshold interval $[s, t]$: Used for volume data set primitives to map intensity to the primitive's color [S1].

**Edge Cases and Pitfalls:**
*   **Frame Buffer Bit Depth Limitation:** Standard RGBA frame buffers have 8 bits per component (32 bits total). The basic algorithm is limited to 8 primitives if using only one color component (e.g., Red) because the frame buffer pixel map only has separate look-up tables for individual RGBA components, not cross combinations [S1].
*   **Large CSG Trees:** For CSG trees with more than 8 primitives, a multi-pass approach is required, though the specific details of this modification are not provided in the source text [S1].
*   **Volume Primitive Bit Clearing:** When rendering volume primitives, the specific bit for that primitive must be cleared for all pixels before rendering. This cannot be done by simply clearing the entire frame buffer, as bits for other geometric primitives must be retained [S1].

## Subtopics

### [CSG and Inter-object Intersection](4703-csg-and-inter-object-intersection.md)
Performs general CSG operations by voxelizing each object into a separate slicemap and rendering them sequentially with blending to compute the desired boolean operation. It differs by explicitly leveraging density computation for collision detection and haptic feedback, maintaining real-time performance on billions of voxels with low dependence on input geometry complexity.

### [SDF-based Voxelization](6595-sdf-based-voxelization.md)
Visualizes CSG meshes or analytic primitives as voxel chunks by sampling a 3D grid and querying Signed Distance Fields (SDF). It differs by performing voxelization on the CPU and storing results in a packed bitmap, rendered as analytic box primitives without shader changes.

### [Sequential CSG](7629-sequential-csg.md)
A CPU-based algorithm for boolean operations on two voxel grids using word-level parallelism. It differs by iterating linearly through grid data and applying bitwise operations to chunks of voxels, serving as a sequential baseline.

## Operating characteristics
*   **Time Complexity:** The algorithm is hardware-accelerated and interactive. Performance is faster than previous methods that used intermediate textures for each Boolean operation node [S1].
*   **Memory Complexity:** Requires frame buffer storage for the current slice and texture memory for the final volume. The PCM size is $2^n$ entries, where $n$ is the number of primitives [S1].
*   **Strengths:** No pre-processing is required for changes to the CSG tree, making it effective for volumetric CSG modeling environments [S1]. It supports multi-resolution rendering through dynamic voxelization of regions of interest [S1].
*   **Limitations:** Limited by frame buffer bit depth (typically 8 primitives per pass) and the inability of the frame buffer pixel map to handle cross-component lookups [S1].
*   **When to use:** Interactive CSG modeling, volume rendering, and applications requiring rapid voxelization of complex CSG models [S1].
*   **When not to use:** When the CSG tree has significantly more than 8 primitives and a multi-pass approach is not feasible or desired, or when explicit surface boundary representation is required for standard graphics systems that do not support the necessary hardware features (polygon engine, frame buffer pixel map, logical operations, 3D texture mapping) [S1].

## Background
The method was introduced in the paper "Fast CSG voxelization by frame buffer pixel mapping" [S1]. It builds on previous work [6] which used blending functions and intermediate textures, but improves performance by using the Point-Classification Map (PCM) and frame buffer pixel maps to avoid hardware restrictions on blending function combinations [S1]. The algorithm relies on specific graphics system features: polygon-based graphics engine, frame buffer pixel map, logical operations, and 3D texture mapping [S1].

## Sources
[S1] Fast CSG voxelization by frame buffer pixel mapping, https://doi.org/10.1145/353888.353896, fetched 2026-11-04
[S2] PDF Real-time Voxelization for Complex Models - GitHub Pages, https://wo1fsea.github.io/files/About_NavMesh/P2_Real-time_Voxelization_for_Complex_Models.pdf, fetched 2026-10-29 (unofficial)
[S3] (PDF) Real-time Voxelization for Complex Models, https://www.researchgate.net/publication/252533918_Real-time_Voxelization_for_Complex_Models, fetched 2026-11-03 (unofficial)
