# Temporal Difference Encoding

Also called: Difference Encoding, Incremental Tree Merging, Tree merging

## Overview
Temporal Difference Encoding is a compression technique for time-varying volume data that exploits high temporal correlation between consecutive time steps [S1, S2, unofficial]. It predicts each sample based on past data and encodes the difference between the prediction and the actual value [S1, S2, unofficial]. Structurally, it merges octrees from consecutive time steps by storing pointers to unchanged subtrees from previous steps instead of duplicating data [S1, S2, unofficial]. Unlike methods that treat all dimensions uniformly, this approach applies difference encoding specifically to the time domain [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input:** A sequence of time-varying volume datasets, where each time step $t$ is represented as an octree structure.
- **Output:** A compressed representation where identical subtrees across time steps are linked via pointers, reducing storage requirements.

**Data Structures**
The method operates on the existing Sparse Voxel Octree (SVO) structure. Each node in the octree represents a spatial region. For temporal encoding, the structure is augmented to support references to nodes from previous time steps.
- **Node Structure:** Standard SVO node fields (e.g., child pointers, data values, metadata).
- **Temporal Pointer:** A reference or index pointing to the corresponding node in the previous time step's octree if the subtree is identical.

**Steps**
1. **Quantization:** Apply quantization to the volume data to attain voxel-level compression. This step significantly influences the performance of subsequent encoding and visualization [S1, S2, unofficial].
2. **Octree Encoding:** Encode the spatial structure of the volume at the current time step using standard octree encoding [S1, S2, unofficial].
3. **Incremental Merging:** Compare the octree of the current time step $t$ with the octree of the previous time step $t-1$ [S1, S2, unofficial].
   - Traverse both trees simultaneously.
   - If a subtree at time step $t$ is identical to the corresponding subtree at time step $t-1$, store only a pointer to the subtree from time step $t-1$ in the current time step's structure [S1, S2, unofficial].
   - If the subtree has changed, recursively process the children or store the new data [S1, S2, unofficial].
4. **Difference Prediction (Optional/General):** For individual samples, predict the value based on past data and encode the difference between the prediction and the actual sample value [S1, S2, unofficial]. This is particularly effective because time-varying volume data are highly correlated from time step to time step, similar to video and speech data [S1, S2, unofficial].

**Formulas**
Let $V_t$ be the volume at time $t$. The encoded representation $E_t$ for a subtree $S$ is:
$$
E_t(S) = 
\begin{cases} 
\text{Pointer}(E_{t-1}(S)) & \text{if } S_t = S_{t-1} \\
\text{Encode}(S_t) & \text{otherwise}
\end{cases}
$$
Where $S_t = S_{t-1}$ denotes structural and value identity.

**Parameters**
- **Quantization Step:** Not given in the sources. The sources note that quantization is used and may have a significant influence on performance [S1, S2, unofficial].
- **Merging Threshold:** Not given in the sources. The merging is based on exact identity of subtrees [S1, S2, unofficial].

**Edge Cases and Pitfalls**
- **Precision:** Quantization errors may prevent exact subtree matching, reducing compression efficiency. The sources emphasize the importance of investigating how quantization assists subsequent compression [S1, unofficial].
- **Degenerate Input:** If there is no temporal coherence (e.g., random noise), the method may not provide significant compression benefits compared to spatial encoding alone. The sources mention testing on datasets that are not all highly spatially and temporally coherent [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def temporal_difference_encoding(current_octree, previous_octree):
    """
    Merges the current octree with the previous octree using temporal difference encoding.
    
    Args:
        current_octree: The root node of the octree for the current time step.
        previous_octree: The root node of the octree for the previous time step.
        
    Returns:
        The merged octree structure with pointers to unchanged subtrees.
    """
    if current_octree is None:
        return None
    
    if previous_octree is None:
        # No previous data to compare against; encode fully
        return encode_spatial_structure(current_octree)
    
    # Check if subtrees are identical
    if is_subtree_identical(current_octree, previous_octree):
        # Store pointer to previous subtree
        return create_pointer_node(previous_octree)
    else:
        # Subtree has changed; process children
        new_node = create_new_node()
        new_node.children = []
        for i in range(8): # Octree has 8 children
            child_current = current_octree.children[i]
            child_previous = previous_octree.children[i] if previous_octree.has_child(i) else None
            new_node.children.append(
                temporal_difference_encoding(child_current, child_previous)
            )
        return new_node

def is_subtree_identical(node_a, node_b):
    """
    Checks if two subtrees are structurally and value-identical.
    """
    if node_a is None and node_b is None:
        return True
    if node_a is None or node_b is None:
        return False
    if node_a.value != node_b.value:
        return False
    if node_a.is_leaf != node_b.is_leaf:
        return False
    
    if node_a.is_leaf:
        return True
    
    # Recursively check children
    for i in range(8):
        if not is_subtree_identical(node_a.children[i], node_b.children[i]):
            return False
    return True

def encode_spatial_structure(node):
    """
    Encodes the spatial structure of a node without temporal references.
    """
    new_node = create_new_node()
    new_node.value = node.value
    new_node.is_leaf = node.is_leaf
    if not node.is_leaf:
        new_node.children = []
        for i in range(8):
            if node.has_child(i):
                new_node.children.append(encode_spatial_structure(node.children[i]))
            else:
                new_node.children.append(None)
    return new_node
```

## Subtopics

### Hierarchical Temporal Grouping
Building a hierarchical structure for each time step, then grouping and encoding together space-corresponding nodes [S3]. This approach extends the basic temporal difference encoding by organizing nodes hierarchically before applying temporal grouping, potentially improving compression efficiency for complex temporal variations [S3].

## Operating characteristics
**Time and Memory Complexity**
- **Memory:** Significantly reduced storage for time-varying data with high temporal coherence, as identical subtrees are represented by pointers rather than duplicated data [S1, S2, unofficial].
- **Time:** Encoding time depends on the comparison of subtrees across time steps. Merging is incremental over the time dimension [S1, unofficial].

**Measured Performance**
- The sources do not provide specific numerical performance metrics. However, they state that with these combined savings (spatial and temporal), a higher level of user interactivity is achieved [S1, S2, unofficial].

**Scaling**
- The method scales with the number of time steps and the degree of temporal coherence. Higher coherence leads to better compression [S1, S2, unofficial].

**Strengths and Limitations**
- **Strengths:** Reduces I/O costs and enables selective rendering by reusing rendered subtrees from previous time steps [S1, S2, unofficial]. The associativity of the over operation for compositing guarantees the correctness of composited results when reusing subtrees [S1, unofficial].
- **Limitations:** Effectiveness depends on the level of temporal coherence. Quantization can impact performance [S1, S2, unofficial].

**When to Use**
- Use for time-varying volume data with high temporal correlation, such as fluid simulations or medical imaging sequences [S1, S2, unofficial].

**When Not to Use**
- Avoid for data with low temporal coherence or where exact reconstruction is critical without quantization artifacts [S1, unofficial].

## Background
**Origin**
- Developed in the context of efficient encoding and rendering of time-varying volume data [S1, S2, unofficial]. The method is attributed to the authors of the NASA Technical Reports Server document [S1, unofficial] and the DTIC document [S2, unofficial].

**History and Development**
- The method builds on the observation that time-varying volume data are highly correlated from time step to time step, similar to video and speech data [S1, S2, unofficial]. It contrasts with Wilhelms and Van Gelder's model, which treats all dimensions the same way, by applying difference encoding specifically to the time domain [S1, unofficial].

**Where it is Used**
- Used in software rendering processes tailored to tree structures for time-varying volume data, enabling selective rendering and reduced I/O costs [S1, S2, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19980217078: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/NASA_NTRS_Archive_19980217078, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA349761: Efficient Encoding and Rendering of Time-Varying Volume Data, https://archive.org/details/DTIC_ADA349761, fetched 2026-10-26 (unofficial)
[S3] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
