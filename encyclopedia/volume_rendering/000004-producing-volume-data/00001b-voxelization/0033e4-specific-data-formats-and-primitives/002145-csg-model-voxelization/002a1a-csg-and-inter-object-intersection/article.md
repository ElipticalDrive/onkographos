# CSG and Inter-object Intersection

Also called: Solid Voxel CSG

## Overview
CSG and Inter-object Intersection performs general Constructive Solid Geometry (CSG) operations by voxelizing each object into a separate slicemap and rendering them over one another using GPU blending to achieve the desired boolean operation [S1, unofficial]. It differs from its parent [CSG Model Voxelization](2497-csg-model-voxelization.md) by explicitly focusing on the intersection and boolean combination of multiple distinct objects rather than just single-object voxelization, and from [SDF-based Voxelization](6595-sdf-based-voxelization.md) by operating in image space via rasterization rather than distance field evaluation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of polygonal meshes representing the objects to be combined.
*   **Outputs:** A voxel grid (slicemap) representing the boolean result of the CSG operation.

**Data Structures:**
*   **Slicemap:** A 2D texture or frame buffer region representing a single slice (or the full volume if rendered slice-by-slice) where pixel values encode the presence of geometry. The specific bit-depth or channel usage depends on the complexity of the CSG tree (e.g., 8 primitives per color channel/component as noted in [S2]).

**Steps:**
1.  **Voxelize Each Object:** Render each object in the CSG operation into a separate slicemap. This is typically done by drawing the geometry into a frame buffer or texture target dedicated to that object [S1, unofficial].
2.  **Blend for Boolean Operation:** Render the slicemap of the first object over the slicemap of the second object (or accumulate results for trees) using a specific blending function or logical operation corresponding to the desired CSG operator (Union, Intersection, Difference, XOR) [S1, unofficial].
    *   For simple binary operations, standard GPU blending or logical operations (AND, OR, XOR) are applied [S2].
    *   For complex CSG trees, intermediate results are stored, requiring $\log^* n$ storage where $n$ is the height of the tree [S1, unofficial].
3.  **Recover Volume (Optional):** If the result is needed for collision detection or haptic feedback, compute the density (volume) of the intersection rapidly from the resulting voxel grid [S1, unofficial].

**Parameters:**
*   **Voxel Resolution:** The method works at a precision of several billion ($10^9$) voxels in real-time [S1, unofficial].
*   **Blending Function:** Determined by the CSG operator (e.g., logical OR for Union, logical AND for Intersection) [S2].

**Edge Cases and Pitfalls:**
*   **CSG Tree Depth:** Standard single-pass methods may be limited by frame buffer bit-depth (e.g., handling only 8 primitives per component) [S2].
*   **Precision:** The cost is directly linked to voxelization; artifacts may occur if the resolution is insufficient for complex intersections [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def perform_csg_intersection(meshes, boolean_op, resolution):
    """
    Performs CSG boolean operation on a list of meshes.
    :param meshes: List of polygonal meshes.
    :param boolean_op: String indicating the operation ('union', 'intersect', 'xor', etc.)
    :param resolution: Resolution of the voxel grid.
    :return: Voxel grid representing the result.
    """
    # Step 1: Voxelize each object into a separate slicemap
    slicemaps = []
    for mesh in meshes:
        # Render mesh to a temporary texture/framebuffer (slicemap)
        # This step depends on the specific GPU rasterization pipeline
        slicemap = render_to_slicemap(mesh, resolution)
        slicemaps.append(slicemap)
    
    # Step 2: Blend slicemaps according to boolean operation
    result_slicemap = slicemaps[0]
    
    for i in range(1, len(slicemaps)):
        next_slicemap = slicemaps[i]
        
        if boolean_op == 'union':
            # Logical OR or additive blending
            result_slicemap = blend_logical_or(result_slicemap, next_slicemap)
        elif boolean_op == 'intersect':
            # Logical AND
            result_slicemap = blend_logical_and(result_slicemap, next_slicemap)
        elif boolean_op == 'xor':
            # Logical XOR
            result_slicemap = blend_logical_xor(result_slicemap, next_slicemap)
        elif boolean_op == 'difference':
            # Subtract: A - B = A AND (NOT B)
            # Note: Implementation of NOT depends on bit representation
            result_slicemap = blend_difference(result_slicemap, next_slicemap)
        else:
            raise ValueError(f"Unsupported boolean operation: {boolean_op}")
            
    return result_slicemap

def blend_logical_or(a, b):
    # Returns a voxel grid where a voxel is 1 if it is 1 in a OR b
    return a | b

def blend_logical_and(a, b):
    # Returns a voxel grid where a voxel is 1 if it is 1 in a AND b
    return a & b

def blend_logical_xor(a, b):
    # Returns a voxel grid where a voxel is 1 if it is 1 in a XOR b
    return a ^ b

def blend_difference(a, b):
    # Returns a voxel grid where a voxel is 1 if it is 1 in a AND 0 in b
    # Assuming binary representation where 1 is inside, 0 is outside
    return a & (~b)
```

## Subtopics

### Multi-Pass Voxelization for Large CSG Trees
An iterative approach to voxelize CSG trees larger than the frame buffer bit-depth limit (e.g., >8 primitives) by repeatedly voxelizing subtrees of at most 8 primitives and replacing the subtree with the resulting volume primitive [S2]. It differs from the basic single-pass method by allowing arbitrary tree depth through iterative reduction [S2]. Choose this when the CSG tree exceeds the hardware's single-pass bit-depth capacity [S2]. Pitfall: Texture memory pressure; multiple intermediate volumes may need to be stored simultaneously, causing swapping. Mitigation: Choose the subtree with the most volume primitives to replace old volumes with new ones [S2]. Complexity: Extra traffic between frame buffer and texture memory, but overhead is small as it is a fast direct cache operation [S2].

### Region-Based Dynamic Voxelization
A technique to apply voxelization on-the-fly to only a region of interest (e.g., viewing region) to avoid low resolution caused by large volume spaces, enabling multi-resolution rendering [S2]. It differs from global voxelization by focusing on a sub-region [S2]. Choose this when the scene volume is large but only a small part is visible or relevant [S2]. Pitfall: Parity error in XOR operations; if slicing starts from the front face of a subregion rather than the outside, parity is not guaranteed. Mitigation: Generate a "0th slice" by drawing all primitives with XOR to establish initial parity, then carry this state over to subsequent slices [S2].

### Blended CSG Volume Rendering
A method to generate blended images for CSG models involving volume data sets by using the binary voxelization result as a 3D mask to extract the volumetric portion of the model [S2]. It differs from binary CSG by incorporating intensity values from volume data [S2]. Choose this when visualizing CSG models that contain internal volume data (e.g., CT scans) [S2]. Steps: Render volumetric primitives individually and combine with logical OR; then extract the region within the CSG model using the binary mask with logical AND [S2]. Pitfall: Background color; a white background must be used when drawing slices for volumetric primitives to avoid parts of the binary volume being removed by the AND operation [S2].

## Operating characteristics
*   **Time Complexity:** Real-time performance is achieved, with costs directly linked to voxelization resolution rather than input geometry complexity [S1, unofficial]. Multi-pass approaches add overhead for texture memory transfers, but this is described as fairly small [S2].
*   **Memory Complexity:** Depends on the number of intermediate volumes stored. For large CSG trees, multi-pass methods may require significant texture memory, potentially causing swapping [S2]. Single-pass methods are limited by frame buffer bit-depth (e.g., 8 primitives per channel) [S2].
*   **Performance:** Works at a precision of several billion ($10^9$) voxels in real-time [S1, unofficial].
*   **Strengths:** Simple implementation, low dependence on input geometry complexity, rapid volume recovery for collision/haptics [S1, unofficial].
*   **Limitations:** Limited by frame buffer bit-depth for single-pass; texture memory pressure for multi-pass; potential artifacts at low resolutions [S1, unofficial][S2].
*   **When to Use:** Real-time CSG modeling, collision detection, haptic feedback, visualization of large CT scans [S1, unofficial][S2].
*   **When Not to Use:** When exact arithmetic is required without grid quantization (use [Constructive Solid Geometry (CSG) Boolean Operations](11752-constructive-solid-geometry-csg-boolean-operations.md) instead) [S1, unofficial].

## Background
The method was introduced in the context of single-pass GPU solid voxelization and its applications, specifically for testing object penetration and performing CSG operations [S1, unofficial]. It leverages the compactness of slicemaps and the ability to estimate normals to compensate for binary information loss, enabling real-time visualization of large volume data sets (e.g., $1024^3$ CT scans) that would otherwise stall the pipeline with texture transfers [S1, unofficial]. The multi-pass and region-based extensions were developed to overcome frame buffer bit-depth limitations and resolution issues in large spatial regions [S2].

## Sources
[S1] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
[S2] Fast CSG voxelization by frame buffer pixel mapping, https://doi.org/10.1145/353888.353896, fetched 2026-11-04
