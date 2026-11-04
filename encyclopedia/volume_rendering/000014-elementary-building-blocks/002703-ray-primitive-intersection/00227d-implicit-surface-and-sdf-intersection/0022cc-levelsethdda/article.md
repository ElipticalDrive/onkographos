# LevelSetHDDA

Also called: Hierarchical DDA for Level Sets, Hierarchical Digital Differential Analyzer for Level Set trees, LevelSet Hierarchical DDA

## Overview
LevelSetHDDA is a specialized Hierarchical Digital Differential Analyzer (HDDA) designed for static intersection testing against level set trees, such as those used in signed distance field (SDF) representations [S1, S2, unofficial]. It recursively traverses the VDB tree hierarchy from a specified node level down to the leaf level (-1) to determine if a ray intersects the implicit surface [S1, S2, unofficial]. It differs from generic volume traversal methods by specializing in the binary occupancy test of level set nodes and terminating early upon finding the first valid voxel intersection [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A `TesterT` object that encapsulates the ray, the tree structure, and the logic for node/voxel testing. The tester must provide:
    *   `tester.ray()`: Returns the ray being tested.
    *   `tester.template hasNode<NodeT>(voxel)`: Returns `true` if the node at `voxel` is active/occupied.
    *   `tester.setRange(t_min, t_max)`: Sets the valid time interval for the current traversal step.
    *   `tester.init(t)`: Initializes the tester state at time `t`.
    *   `tester(voxel, t)`: Returns `true` if the voxel at `voxel` is intersected at time `t`.
*   **Output:** `bool` indicating whether an intersection was found.

**Data Structures:**
The method relies on the OpenVDB tree hierarchy.
*   `TreeT`: The VDB tree type.
*   `ChainT`: Defined as `typename TreeT::RootNodeType::NodeChainType` [S1, S2, unofficial].
*   `NodeT`: Defined as `typename ChainT::template Get<NodeLevel>` [S1, S2, unofficial]. This resolves to the specific node type at the current `NodeLevel` in the hierarchy.
*   `math::DDA`: A Digital Differential Analyzer instance. It requires a ray and the total number of voxels along the node's side (`NodeT::TOTAL` for internal nodes, `0` for the leaf specialization) [S1, S2, unofficial].

**Algorithm Steps:**
The method is implemented as a template struct `LevelSetHDDA<TreeT, int NodeLevel>` with two specializations: the general recursive case and the base case for voxels (`NodeLevel = -1`) [S1, S2, unofficial].

1.  **General Case (`NodeLevel > -1`):**
    1.  Create a DDA instance for the current node type: `math::DDA<typename TesterT::RayT, NodeT::TOTAL> dda(tester.ray())` [S1, S2, unofficial].
    2.  Enter a loop controlled by `dda.step()`.
    3.  Check if the current voxel position `dda.voxel()` corresponds to an active node using `tester.template hasNode<NodeT>(dda.voxel())` [S1, S2, unofficial].
    4.  If the node is active:
        *   Update the tester's valid time range to the current DDA interval: `tester.setRange(dda.time(), dda.next())` [S1, S2, unofficial].
        *   Recursively call `LevelSetHDDA<TreeT, NodeLevel-1>::test(tester)` [S1, S2, unofficial].
        *   If the recursive call returns `true`, propagate `true` immediately [S1, S2, unofficial].
    5.  Advance the DDA: `dda.step()` [S1, S2, unofficial]. If it returns `false`, exit the loop.
    6.  If the loop completes without finding an intersection, return `false` [S1, S2, unofficial].

2.  **Base Case (`NodeLevel = -1`):**
    1.  Create a DDA instance for voxels: `math::DDA<typename TesterT::RayT, 0> dda(tester.ray())` [S1, S2, unofficial].
    2.  Initialize the tester with the starting time: `tester.init(dda.time())` [S1, S2, unofficial].
    3.  Enter a loop controlled by `dda.step()`.
    4.  Test the current voxel: `if (tester(dda.voxel(), dda.next()))` [S1, S2, unofficial].
    5.  If the test returns `true`, return `true` immediately [S1, S2, unofficial].
    6.  Advance the DDA: `dda.step()` [S1, S2, unofficial]. If it returns `false`, exit the loop.
    7.  If the loop completes without finding an intersection, return `false` [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Template struct LevelSetHDDA<TreeT, NodeLevel>
# NodeLevel is an integer > -1

def test(tester):
    # Determine node type and its total voxel count along one dimension
    ChainT = TreeT.RootNodeType.NodeChainType
    NodeT = ChainT.Get[NodeLevel]
    total_voxels = NodeT.TOTAL
    
    # Initialize DDA for the current node level
    # Ray is obtained from tester
    dda = math.DDA(tester.ray(), total_voxels)
    
    while True:
        # Check if the current voxel position has an active node
        if tester.hasNode[NodeT](dda.voxel()):
            # Set the time range for the recursive step
            tester.setRange(dda.time(), dda.next())
            
            # Recurse to the next lower level
            if LevelSetHDDA[TreeT, NodeLevel-1].test(tester):
                return True
        
        # Step to the next voxel
        if not dda.step():
            break
            
    return False

# Specialization for NodeLevel = -1 (Voxels)
def test(tester):
    # Initialize DDA for voxels (total_voxels = 0 in this specialization context)
    dda = math.DDA(tester.ray(), 0)
    
    # Initialize tester state
    tester.init(dda.time())
    
    while True:
        # Test the specific voxel at the current time
        if tester(dda.voxel(), dda.next()):
            return True
            
        # Step to the next voxel
        if not dda.step():
            break
            
    return False
```

**Parameters:**
*   `TreeT`: The VDB tree type [S1, S2, unofficial].
*   `NodeLevel`: The current node level in the hierarchy being tested [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The DDA implementation relies on floating-point arithmetic for ray traversal. Precision issues may arise with very long rays or very small features, potentially causing missed nodes or voxels [G].
*   **Boundary Conditions:** The DDA loop terminates when `dda.step()` returns false, which typically happens when the ray exits the bounding volume of the current node or voxel grid [S1, S2, unofficial].
*   **Empty Trees:** If the tree has no active nodes or voxels, the `hasNode` or `tester` calls will consistently return false, and the function will return false after traversing the entire ray extent within the bounds [S1, S2, unofficial].

## Subtopics

### VolumeHDDA
A generic hierarchical DDA for ray intersections against a generic volume tree. Unlike LevelSetHDDA, it traverses down to the leaf node level (not the voxel level) and is designed for volumetric data rather than binary level set occupancy. [VolumeHDDA](3592-volumehdda.md)

## Operating characteristics
*   **Time Complexity:** The complexity depends on the tree depth and the number of active nodes/voxels intersected by the ray. In the worst case, it traverses all active nodes down to the leaf level. The base case iterates through all voxels along the ray segment within the leaf node.
*   **Memory Complexity:** Minimal additional memory is required beyond the recursion stack, which is proportional to the tree depth.
*   **Strengths:** Efficient for sparse level set trees as it skips inactive nodes quickly. Early termination upon finding the first intersection makes it suitable for collision detection and surface finding.
*   **Limitations:** Specialized for level sets; not directly applicable to dense volumetric data without modification. Performance degrades if the ray intersects many active nodes/voxels.
*   **When to use:** Use for ray-casting against signed distance fields or level set representations where only the first intersection or occupancy is needed.
*   **When not to use:** Use VolumeHDDA for dense volumetric data traversal where sampling values is required, not just occupancy.

## Background
LevelSetHDDA is part of the OpenVDB library, developed by DreamWorks Animation and later maintained by the Academy Software Foundation. It is defined in `openvdb/math/DDA.h` [S1, S2, unofficial]. The method leverages the hierarchical structure of VDB trees to optimize ray traversal for level set data, which is commonly used in visual effects for fluid simulation, collision detection, and surface reconstruction.

## Sources
[S1] OpenVDB: openvdb/math/DDA.h Source File, https://academysoftwarefoundation.github.io/openvdb/DDA_8h_source.html, fetched 2026-10-24 (unofficial)
[S2] OpenVDB: openvdb/math/DDA.h Source File, https://www.openvdb.org/documentation/doxygen/DDA_8h_source.html, fetched 2026-10-26 (unofficial)
