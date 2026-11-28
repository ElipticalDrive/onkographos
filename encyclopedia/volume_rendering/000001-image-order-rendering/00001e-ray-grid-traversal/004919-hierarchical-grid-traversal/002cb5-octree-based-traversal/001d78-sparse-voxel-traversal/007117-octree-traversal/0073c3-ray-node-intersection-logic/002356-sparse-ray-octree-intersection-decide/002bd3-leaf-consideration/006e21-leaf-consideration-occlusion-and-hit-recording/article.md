# Leaf Consideration (Occlusion and Hit Recording)

Also called: LeafNode.IsCompletelyOccluded, Sub-node Iteration and AABB Construction, Voxel Hit Detection in Leaf Nodes

## Overview
This method handles the final stage of ray traversal within a sparse voxel octree: processing a leaf node that has been determined to be intersected by the ray. It distinguishes between two structural cases: a leaf node containing no further subdivision (a pure voxel leaf) and a leaf node that contains sub-nodes (a "virtual" leaf or intermediate node acting as a leaf in the current traversal context). The method determines occlusion status, records the closest hit point and normal, and optionally counts all occluded voxels within the node's extent. It serves as the termination condition for the ray traversal, providing the precise intersection data required for shading or pathfinding logic.

## How it works

### Inputs and Outputs
**Inputs:**
- `LeafNode`: The current leaf node being processed. Contains bounds, address, and flags.
- `RayState`: Contains `RayOrigin`, `OriginalRayDirection`, and `RaySize`.
- `OutHit`: An output structure accumulating hit information. Fields include `Distance`, `ImpactPoint`, `ImpactNormal`, `NodeAddress`, `bBlockingHit`, and `OccludedVoxelCount`.
- `bCountAllOccludedVoxels`: A boolean flag determining whether to stop at the first hit or continue to count all occluded voxels in the leaf.

**Outputs:**
- `OutHit`: Updated with the closest hit information if a hit is found.
- Return value: `bool` indicating whether a hit was recorded (`true`) or not (`false`).

### Data Structures
- **HitCandidate**: Implicitly represented by `OutHit`.
- **AABB (Axis-Aligned Bounding Box)**: Defined by `Min` and `Max` vectors.
- **SubNodeOffset**: Precomputed offsets for the 64 sub-nodes of a leaf, derived from `LeafCenter` and `LeafExtent`.

### Steps

1. **Complete Occlusion Check**:
   Call `LeafNode.IsCompletelyOccluded()`. If this returns `true`:
   - Increment `OutHit.OccludedVoxelCount` by the number of voxels in the leaf node.
   - Return `true` immediately. No further processing is required as the node is fully blocked.

2. **Pure Leaf Node Handling (No Sub-nodes)**:
   If the leaf node does not contain sub-nodes:
   - Calculate the intersection distance $T_{min}$ of the ray with the leaf node's AABB.
   - If $T_{min} < OutHit.Distance$ (or if `OutHit.bBlockingHit` is false):
     - Record the impact point, normal, distance ($T_{min}$), and node address in `OutHit`.
     - Set `OutHit.bBlockingHit = true`.
   - If `bCountAllOccludedVoxels` is `false`, return `true`.
   - Otherwise, return `true` (traversal ends here as there are no sub-nodes to iterate).

3. **Sub-node Iteration (Leaf with Sub-nodes)**:
   If the leaf node contains sub-nodes:
   - Initialize local variables:
     - `bHit = false`
     - `ClosestHit = \infty` (MAX_float)
     - `ClosestSubNode = LeafNode.Address`
     - `OccludedSubNodes = 0`
     - `IntersectingSubNodes = 0`
   - Iterate through all 64 sub-nodes (`SubIdx` from 0 to 63):
     a. **Occlusion Skip**:
        - Call `IsSubNodeOccluded(SubIdx)`.
        - If the sub-node is **not** occluded, skip to the next sub-node.
     b. **AABB Construction**:
        - Calculate `SubNodePos`:
          $$ \text{SubNodePos} = \text{LeafCenter} + \text{GetSubNodeOffset}(\text{SubIdx}, \text{LeafExtent}) $$
        - Calculate `SubNodeExtent`:
          $$ \text{SubNodeExtent} = \text{LeafExtent} \times 0.25 $$
        - Build `SubNodeBox` AABB using `SubNodePos` and `SubNodeExtent`.
     c. **Ray-Box Intersection**:
        - Test intersection between `RayState` and `SubNodeBox`.
        - Compute `SubTMin` and `SubTMax`.
        - If intersecting:
          - Increment `IntersectingSubNodes`.
          - Increment `OutHit.OccludedVoxelCount`.
          - Set `bHit = true`.
          - If `SubTMin < ClosestHit`:
            - Update `ClosestHit = SubTMin`.
            - Set `ClosestSubNode.SubNodeIndex = SubIdx`.
            - Update `OutHit` fields (`ImpactPoint`, `ImpactNormal`, `Distance`, `NodeAddress`, `bBlockingHit`) with the new closest hit data.
            - If `bCountAllOccludedVoxels` is `false`, return `true` immediately.
   - After iterating all sub-nodes, return `bHit`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def LeafConsiderationOcclusionAndHitRecording(LeafNode, RayState, OutHit, bCountAllOccludedVoxels):
    # Step 1: Check for complete occlusion
    if LeafNode.IsCompletelyOccluded():
        OutHit.OccludedVoxelCount += LeafNode.VoxelCount
        return True

    # Step 2: Handle pure leaf nodes (no sub-nodes)
    if not LeafNode.HasSubNodes():
        # Calculate intersection with leaf AABB
        SubTMin, SubTMax = RayBoxIntersection(LeafNode.AABB, RayState.RayOrigin, RayState.OriginalRayDirection, RayState.RaySize)
        
        if SubTMin is not None:
            # Check if this is a better hit
            if not OutHit.bBlockingHit or SubTMin < OutHit.Distance:
                OutHit.Distance = SubTMin
                OutHit.ImpactPoint = RayState.RayOrigin + RayState.OriginalRayDirection * SubTMin
                OutHit.ImpactNormal = CalculateNormal(LeafNode.AABB, OutHit.ImpactPoint)
                OutHit.NodeAddress = LeafNode.Address
                OutHit.bBlockingHit = True
            
            if not bCountAllOccludedVoxels:
                return True
            else:
                # Count voxels in this leaf
                OutHit.OccludedVoxelCount += LeafNode.VoxelCount
                return True

    # Step 3: Handle leaf nodes with sub-nodes
    bHit = False
    ClosestHit = float('inf')
    ClosestSubNode = LeafNode.Address
    IntersectingSubNodes = 0
    
    LeafCenter = LeafNode.Center
    LeafExtent = LeafNode.Extent
    
    for SubIdx in range(64):
        # Skip if sub-node is not occluded (i.e., it is free/empty)
        if not IsSubNodeOccluded(LeafNode, SubIdx):
            continue
            
        # Calculate sub-node AABB
        SubNodePos = LeafCenter + GetSubNodeOffset(SubIdx, LeafExtent)
        SubNodeExtent = LeafExtent * 0.25
        SubNodeBox = BuildAABB(SubNodePos, SubNodeExtent)
        
        # Test ray-box intersection
        SubTMin, SubTMax = RayBoxIntersection(SubNodeBox, RayState.RayOrigin, RayState.OriginalRayDirection, RayState.RaySize)
        
        if SubTMin is not None:
            IntersectingSubNodes += 1
            OutHit.OccludedVoxelCount += 1 # Assuming 1 voxel per sub-node for counting purposes, or adjust based on leaf depth
            bHit = True
            
            if SubTMin < ClosestHit:
                ClosestHit = SubTMin
                ClosestSubNode.SubNodeIndex = SubIdx
                OutHit.Distance = SubTMin
                OutHit.ImpactPoint = RayState.RayOrigin + RayState.OriginalRayDirection * SubTMin
                OutHit.ImpactNormal = CalculateNormal(SubNodeBox, OutHit.ImpactPoint)
                OutHit.NodeAddress = LeafNode.Address # Or specific sub-node address if available
                OutHit.bBlockingHit = True
                
                if not bCountAllOccludedVoxels:
                    return True
    
    return bHit
```

### Parameters and Edge Cases
- **`bCountAllOccludedVoxels`**: If `false`, the method returns early upon finding the first hit, optimizing for performance. If `true`, it continues to count all occluded voxels, useful for statistics or volumetric analysis.
- **Precision**: Ray-box intersection tests are sensitive to floating-point precision. Use robust intersection algorithms to avoid missed hits due to epsilon errors.
- **Degenerate Input**: If `RayState.OriginalRayDirection` is zero or `LeafExtent` is zero, handle gracefully (return no hit or error).

## Operating characteristics

### Time and Memory Complexity
- **Time Complexity**: $O(1)$ for pure leaf nodes. $O(64)$ for leaf nodes with sub-nodes, as it iterates through all 64 sub-nodes. In practice, many sub-nodes may be skipped if occluded, but the worst case is constant time relative to the leaf size.
- **Memory Complexity**: $O(1)$ auxiliary space, as it only uses local variables for tracking the closest hit and counters.

### Performance and Scaling
- **Performance**: Fast for sparse scenes where many leaf nodes are occluded or empty. The early exit on `bCountAllOccludedVoxels = false` significantly reduces computation time for pathfinding where only the first hit matters.
- **Scaling**: Scales linearly with the number of sub-nodes in the leaf, but since the number of sub-nodes is fixed at 64, it is effectively constant time. Performance is more dependent on the density of occluded voxels within the leaf.

### Strengths and Limitations
- **Strengths**: Handles both pure and subdivided leaf nodes uniformly. Efficiently skips non-occluded sub-nodes. Provides accurate hit recording for shading or pathfinding.
- **Limitations**: Iterating through all 64 sub-nodes can be costly if many are occluded and need to be checked. The method assumes a fixed octree structure with 64 sub-nodes per leaf.

### When to Use
- Use this method when traversing a sparse voxel octree for ray casting, particularly in navigation or collision detection where precise hit information is required.
- Choose `bCountAllOccludedVoxels = false` for real-time applications like pathfinding where only the first obstacle matters.
- Choose `bCountAllOccludedVoxels = true` for offline analysis or volumetric rendering where total occlusion counts are needed.

## Background

### Origin and History
This method is part of the Nav3D plugin for Unreal Engine, developed by Darby Costello [S1, unofficial]. It is designed to handle 3D navigation data using sparse voxel octrees, enabling efficient pathfinding and tactical reasoning in complex 3D environments. The method leverages the hierarchical structure of the octree to quickly determine occlusion and hit points, optimizing for both memory usage and traversal speed.

### Usage
The method is used in the Nav3D plugin for processing navigation meshes and volumes, allowing for dynamic and efficient pathfinding in 3D spaces. It is particularly useful in games and simulations where agents need to navigate complex environments with varying levels of detail and occlusion.

## Sources

[S1] darbycostello/Nav3D, https://github.com/darbycostello/Nav3D, fetched 2026-10-26 (unofficial)
