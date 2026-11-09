# Offline Sequential Reconstruction

Also called: NeuralVDB Decoding, NeuralVDB [Hash, NN(4), NN(3)] Decoding, Offline VDB reconstruction, Sequential decoding

## Overview
Offline Sequential Reconstruction is a method for decoding a NeuralVDB representation into a standard VDB tree in a single sequential pass, prioritizing memory efficiency over reconstruction time [S1][S2, unofficial]. It is designed for scenarios where the data is stored offline (e.g., on disk) or transmitted over low-bandwidth networks, and the entire volume must be reconstructed into memory [S1][S2, unofficial]. Unlike online random access methods, it does not support immediate point queries without full reconstruction, but it achieves a significantly smaller in-memory footprint by leveraging the compressed neural structure for the lower levels of the VDB hierarchy [S1][S2, unofficial].

## How it works
**Inputs:**
- Compressed NeuralVDB data using the `[Hash, 5, NN(4), NN(3)]` configuration [S1][S2, unofficial].
- A list of false positives explicitly encoded during training [S1][S2, unofficial].
- Disjoint blocked ranges of the volume domain [S1][S2, unofficial].

**Outputs:**
- A standard VDB tree in memory [S1][S2, unofficial].

**Data Structures:**
- **Standard VDB Nodes (Levels 3, 2, 1 masks):** Encoded identically to standard VDB structures, including bit-masks that compactly encode coordinates [S1][S2, unofficial].
- **Level 1 Nodes:** Size $16^3$ [S1][S2, unofficial]. Each node has associated classifiers (for child and active masks) and a value regressor (for tile values) [S1][S2, unofficial].
- **Level 0 Nodes (Leaves):** Size $8^3$ [S1][S2, unofficial]. Each leaf has an associated binary classifier (for active state) and a value regressor (for voxel values) [S1][S2, unofficial].
- **False Positive List:** A list of coordinates or identifiers for incorrect active voxels/tiles identified during training, used to correct masks [S1][S2, unofficial].

**Steps:**
1. **Reconstruct Upper Levels:** Process root and child nodes (levels 3, 2, and 1 masks) using standard VDB decoding methods, as they are encoded identically to standard VDB [S1][S2, unofficial].
2. **Sequential Pass Setup:** Distribute the volume into disjoint blocked ranges, assigning each range to a GPU for concurrent processing [S1][S2, unofficial]. Each blocked range has dedicated classifiers and regressors but accesses all networks to ensure continuity [S1][S2, unofficial].
3. **Level 1 Reconstruction:**
   - Sequentially access level 1 nodes. Their coordinates are derived trivially from the child masks at level 3 [S1][S2, unofficial].
   - For each level 1 node (size $16^3$):
     - Use standard inference with the associated classifiers to reconstruct the child mask and active mask [S1][S2, unofficial].
     - Use the value regressor to reconstruct tile values [S1][S2, unofficial].
     - Correct the reconstructed masks using the explicitly encoded list of false positives [S1][S2, unofficial].
4. **Level 0 Reconstruction:**
   - Using the corrected child masks from level 1, visit all leaf nodes (size $8^3$) [S1][S2, unofficial].
   - Sequentially infer voxel values and active states for each leaf node using the level 0 value regressor and binary classifier [S1][S2, unofficial].
5. **Merge Results:** Merge the reconstructed blocked ranges from multiple GPUs into a single output VDB [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def offline_sequential_reconstruction(neural_vdb_data, false_positives, blocked_ranges):
    """
    Reconstructs a standard VDB from NeuralVDB data in a single sequential pass.
    
    Args:
        neural_vdb_data: Compressed NeuralVDB data structure.
        false_positives: List of false positives encoded during training.
        blocked_ranges: List of disjoint volume ranges to process.
        
    Returns:
        standard_vdb: Reconstructed standard VDB tree.
    """
    standard_vdb = StandardVDB()
    
    # Step 1: Reconstruct upper levels (Root, Level 3, 2, 1 masks)
    # These are encoded identically to standard VDB
    standard_vdb.root = decode_standard_vdb_node(neural_vdb_data.root)
    standard_vdb.level_3 = decode_standard_vdb_nodes(neural_vdb_data.level_3)
    standard_vdb.level_2 = decode_standard_vdb_nodes(neural_vdb_data.level_2)
    standard_vdb.level_1_masks = decode_standard_vdb_nodes(neural_vdb_data.level_1_masks)
    
    # Step 2: Process blocked ranges concurrently (simulated here sequentially)
    reconstructed_ranges = []
    for range_id, range_data in enumerate(blocked_ranges):
        range_vdb = StandardVDB()
        
        # Step 3: Reconstruct Level 1 nodes (size 16^3)
        for level_1_node_id in range_data.level_1_nodes:
            # Coordinates derived from level 3 child masks
            coords = derive_coords_from_level_3_masks(level_1_node_id)
            
            # Infer child and active masks using classifiers
            child_mask, active_mask = infer_level_1_masks(
                neural_vdb_data.classifiers[level_1_node_id],
                coords
            )
            
            # Correct masks with false positives
            active_mask = correct_mask(active_mask, false_positives, level_1_node_id)
            
            # Infer tile values using value regressor
            tile_values = infer_level_1_values(
                neural_vdb_data.regressors[level_1_node_id],
                coords
            )
            
            # Create level 1 node in range VDB
            range_vdb.add_level_1_node(level_1_node_id, coords, child_mask, active_mask, tile_values)
        
        # Step 4: Reconstruct Level 0 leaf nodes (size 8^3)
        for level_1_node_id in range_data.level_1_nodes:
            level_1_node = range_vdb.get_level_1_node(level_1_node_id)
            if level_1_node.active_mask:
                # Visit all leaf nodes under this level 1 node
                for leaf_node_id in level_1_node.child_nodes:
                    # Infer voxel values and active states
                    active_state = infer_level_0_active_state(
                        neural_vdb_data.classifiers[leaf_node_id],
                        leaf_node_id
                    )
                    voxel_values = infer_level_0_values(
                        neural_vdb_data.regressors[leaf_node_id],
                        leaf_node_id
                    )
                    
                    # Create leaf node in range VDB
                    range_vdb.add_level_0_node(leaf_node_id, active_state, voxel_values)
        
        reconstructed_ranges.append(range_vdb)
    
    # Step 5: Merge results
    standard_vdb = merge_vdb_ranges(reconstructed_ranges)
    return standard_vdb

def derive_coords_from_level_3_masks(node_id):
    """
    Derives coordinates of a level 1 node from its child masks at level 3.
    Implementation details depend on the specific bit-mask encoding scheme [S1].
    """
    # Placeholder for coordinate derivation logic based on Museth [2013]
    pass

def infer_level_1_masks(classifier, coords):
    """
    Uses standard inference to reconstruct child and active masks for a level 1 node.
    """
    # Placeholder for classifier inference
    pass

def correct_mask(mask, false_positives, node_id):
    """
    Corrects the active mask using the list of false positives.
    """
    # Placeholder for false positive correction logic
    pass

def infer_level_1_values(regressor, coords):
    """
    Uses the value regressor to reconstruct tile values for a level 1 node.
    """
    # Placeholder for regressor inference
    pass

def infer_level_0_active_state(classifier, leaf_node_id):
    """
    Uses the binary classifier to infer the active state of a level 0 leaf node.
    """
    # Placeholder for classifier inference
    pass

def infer_level_0_values(regressor, leaf_node_id):
    """
    Uses the value regressor to infer voxel values for a level 0 leaf node.
    """
    # Placeholder for regressor inference
    pass

def merge_vdb_ranges(ranges):
    """
    Merges multiple reconstructed VDB ranges into a single VDB.
    """
    # Placeholder for merging logic
    pass
```

**Parameters:**
- **Configuration:** `[Hash, 5, NN(4), NN(3)]` [S1][S2, unofficial].
- **Level 1 Node Size:** $16^3$ [S1][S2, unofficial].
- **Level 0 Leaf Node Size:** $8^3$ [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
- **False Positives:** Masks must be corrected using the explicitly encoded list of false positives to ensure accuracy [S1][S2, unofficial].
- **Discontinuity:** Although each blocked range has dedicated networks, access to all networks ensures no discontinuity at range boundaries [S1][S2, unofficial].
- **Memory Efficiency:** This method prioritizes low memory footprint, so it is not suitable for applications requiring fast random access [S1][S2, unofficial].

## Subtopics

### Online Random Access — NeuralVDB [Hash, 5, 4, NN(3)] Decoding
A reconstruction method for applications requiring fast random access and a small memory footprint, using standard VDB acceleration techniques for tree traversal and neural representation only for voxel values [S2, unofficial]. It uses the `[Hash, 5, 4, NN(3)]` configuration, where the upper levels (3, 2, 1) are standard VDB structures, and only the leaf level (0) uses neural networks [S2, unofficial]. To query a point, standard VDB acceleration is used to determine if it maps to a tile or a voxel; if it is a tile, the value is returned directly; if it is a voxel, the value is predicted by the regressor [S2, unofficial]. Compared to `[Hash, 5, NN(4), NN(3)]`, this method is faster for real-time random access because it avoids hierarchical neural network inference for the upper levels, making it suitable for rendering and collision detection [S2, unofficial]. It can be trivially reconstructed from `[Hash, 5, NN(4), NN(3)]` by leaving the voxel regressor unchanged, acting as a pre-cached representation [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Single sequential pass over the compressed data [S1][S2, unofficial]. Reconstruction time is higher than online random access methods due to the sequential nature and neural inference for lower levels [S1][S2, unofficial].
- **Memory Complexity:** Low memory footprint, as it reconstructs the entire VDB tree into memory [S1][S2, unofficial]. The in-memory footprint is an order of magnitude smaller than standard VDB with neural values at all levels [S2, unofficial].
- **Performance:** Suitable for offline storage and low-bandwidth transmission [S1][S2, unofficial]. Not suitable for real-time random access applications due to the slowness of hierarchical neural network inference for two levels [S1][S2, unofficial].
- **Strengths:** Low memory usage, efficient for sequential reconstruction, concurrent processing on multiple GPUs [S1][S2, unofficial].
- **Limitations:** Slow for random access, requires full reconstruction before use [S1][S2, unofficial].
- **When to Use:** Offline storage, network transmission, applications prioritizing memory efficiency over reconstruction time [S1][S2, unofficial].
- **When Not to Use:** Real-time rendering, collision detection, applications requiring fast random access [S1][S2, unofficial].

## Background
Originated in the NeuralVDB paper by [S1][S2, unofficial], which introduced hierarchical neural networks for sparse volume representation. The offline sequential reconstruction method was developed to address the need for efficient decoding of NeuralVDBs for offline applications, such as storage on slow secondary-storage devices or transmission over low-bandwidth networks [S1][S2, unofficial]. It builds on the standard VDB structure for upper levels and uses neural networks for the lower levels to achieve high compression ratios [S1][S2, unofficial].

## Sources
[S1] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks, https://www.emergentmind.com/papers/2208.04448, fetched 2026-10-27
[S2] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks | NVIDIA, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
