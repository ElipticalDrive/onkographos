# Omnitree Coarsening

Also called: Last-level coarsening, reverse refinement

## Overview
Omnitree coarsening is the inverse operation of omnitree refinement, designed to reduce the resolution of a sparse voxel representation by fusing child nodes back into their parent. It solves the problem of compressing or simplifying a tree structure that has been previously refined, allowing for dynamic updates where fine-grained details are no longer required. Unlike general tree restructuring, this method specifically targets "last-level" changes, ensuring that coarsening operations are deterministic and do not require complex validation of subtree symmetry across arbitrary depths.

## How it works
The method operates by modifying the splitting labels of internal nodes and fusing their children along specified dimensions. It is implemented as a special case of the refinement algorithm, utilizing a marker-sweep approach.

### Inputs and Outputs
*   **Input:** An initial omnitree $T$ with nodes characterized by splitting labels $\vec{b}(v)$ and a set of coarsening requests.
*   **Output:** A target omnitree $T'$ with a linearized descriptor and a mapping $M$ from initial node indices to target node indices.

### Data Structures
*   **Splitting Label ($\vec{b}(v)$):** A binary vector of length $d$ (dimensionality). $\vec{b}(v) \in \{0, 1\}^d$. $b_j = 1$ indicates a split in dimension $j$. The all-zero vector $\vec{0}$ denotes a leaf.
*   **Coarsening Marker ($\vec{m}(v)$):** A vector attached to internal nodes. $\vec{m}(v) \in \{-1, 0\}^d$.
    *   $m_j = -1$ if dimension $j$ is to be coarsened (removed from split).
    *   $m_j = 0$ otherwise.
*   **Location Code Stack ($\$$):** A stack used during traversal to track the binary path from the root to the current node.
*   **Mapping ($M$):** An array or hash map where $M[a]$ stores the index of the target node corresponding to the initial node at index $a$.

### Parameters
*   **Coarsening Dimensions ($C$):** The subset of split dimensions to remove, $C \subseteq \sigma(v)$, where $\sigma(v) = \{j : b_j(v) = 1\}$.
*   **Constraint:** The method restricts requests to **last-level coarsening**. This means markers are only attached to nodes whose children are all leaves. This ensures that the subtrees beneath the node are identical in structure, guaranteeing that fusion is well-defined without checking for symmetry at deeper levels [S1, unofficial].

### Step-by-Step Method

1.  **Attach Markers:**
    For each node $v$ in the initial tree where coarsening is desired, attach a negative marker $\vec{m}(v)$.
    *   If dimension $j \in C$, set $m_j(v) = -1$.
    *   Otherwise, set $m_j(v) = 0$.
    *   Markers are attached directly to the internal (parent) nodes [S1, unofficial].

2.  **Compute New Splitting Label:**
    For each node $v$ with a marker, compute the new splitting label $\vec{b}(v')$ for the corresponding node $v'$ in the target tree:
    $$ \vec{b}(v') = \vec{b}(v) + \vec{m}(v) $$
    The addition is component-wise. Since $b_j \in \{0, 1\}$ and $m_j \in \{-1, 0\}$:
    *   If $b_j(v) = 1$ and $m_j(v) = -1$, then $b_j(v') = 0$.
    *   If $b_j(v) = 1$ and $m_j(v) = 0$, then $b_j(v') = 1$.
    *   If $b_j(v) = 0$, then $b_j(v') = 0$ (markers are only on split dimensions).

3.  **Determine Fusion Groups:**
    Let $\bar{C} = \sigma(v) \setminus C$ be the dimensions that remain split.
    *   The node $v$ initially has $2^{|\sigma(v)|}$ children.
    *   These children are grouped by their coordinates in the dimensions $\bar{C}$.
    *   Each group of children sharing the same $\bar{C}$-indices is fused into a single child of $v'$ in the target tree.
    *   The target node $v'$ will have $2^{|\bar{C}|}$ children.

4.  **Construct Target Descriptor and Mapping:**
    Traverse the initial tree (e.g., using the algorithm from [29]) to build the target linearized descriptor.
    *   **Case A: Partial Coarsening ($\bar{C} \neq \emptyset$):**
        When the algorithm encounters the first member $v_a$ of a fusion group (the one with the lexicographically smallest location code in $\bar{C}$):
        1.  Emit the target node $v'_a$ with label $\vec{b}(v')$.
        2.  Record the mapping: $M[a] = \text{index}(v'_a)$.
        3.  Identify all other members $v_i$ of the same fusion group. This is done by appending their local binary locations to the location code stack $\$$ to reconstruct their full paths.
        4.  Map all other members: $M[i] = \text{index}(v'_a)$ for all $i$ in the group.
    *   **Case B: Full Coarsening ($\vec{b}(v') = \vec{0}$):**
        If $C = \sigma(v)$, all split dimensions are removed. $v'$ becomes a leaf with no children.
        1.  No children of $v'$ are emitted to the target tree.
        2.  All $2^{|\sigma(v)|}$ direct children of $v$ in the initial tree (which are leaves) are mapped to the index of $v'$.
        3.  These children are identified as the immediate successors of $v$ in the initial tree's linearized descriptor.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def omnitree_coarsening(initial_tree, coarsening_requests):
    """
    initial_tree: List of nodes, where each node has:
        - id: integer index
        - b: splitting label vector (list of 0/1)
        - children: list of child node IDs
    coarsening_requests: Dict mapping node_id -> set of dimensions to coarsen (C)
    
    Returns:
        target_tree: List of target nodes
        mapping: Dict mapping initial_node_id -> target_node_id
    """
    target_nodes = []
    mapping = {}
    
    # Precompute new splitting labels
    new_labels = {}
    for node in initial_tree:
        nid = node.id
        b = node.b
        C = coarsening_requests.get(nid, set())
        
        # Compute marker m
        m = [0] * len(b)
        for j in C:
            m[j] = -1
            
        # Compute new label b' = b + m
        b_new = [b[j] + m[j] for j in range(len(b))]
        new_labels[nid] = b_new
        
        # Identify split dimensions
        sigma_v = [j for j, val in enumerate(b) if val == 1]
        C_set = set(C)
        bar_C = [j for j in sigma_v if j not in C_set]
        
        # Group children by bar_C coordinates
        # Children are indexed by local Morton codes corresponding to sigma_v
        # We need to group them by their indices in bar_C dimensions
        child_groups = {}
        
        # Assuming children are stored in Morton order corresponding to sigma_v
        # The number of children is 2^len(sigma_v)
        num_children = len(node.children)
        
        for idx, child_id in enumerate(node.children):
            # Decode local Morton code to binary tuple for sigma_v dimensions
            # This is a helper function to get binary representation of index
            local_bits = get_local_binary_bits(idx, len(sigma_v))
            
            # Create a key based on bar_C dimensions
            # Map sigma_v index to bar_C index
            key = []
            for dim_idx, dim in enumerate(sigma_v):
                if dim in bar_C:
                    key.append(local_bits[dim_idx])
            
            key_tuple = tuple(key)
            if key_tuple not in child_groups:
                child_groups[key_tuple] = []
            child_groups[key_tuple].append(child_id)

        # Process each fusion group
        for key_tuple, group_children in child_groups.items():
            # Emit target node for the first child in the group
            first_child_id = group_children[0]
            
            # Create target node
            target_node = {
                'id': len(target_nodes),
                'b': new_labels[nid],
                'children': [] # Will be populated recursively or later
            }
            target_nodes.append(target_node)
            target_node_id = target_node['id']
            
            # Map all children in this group to this target node
            for child_id in group_children:
                mapping[child_id] = target_node_id
                
            # Note: The actual recursive construction of the subtree 
            # for the target node depends on the children's own states.
            # In a full implementation, this would trigger recursive processing 
            # or be handled by the main traversal loop.
            # For the purpose of this method description, we record the mapping.
            
    return target_nodes, mapping

def get_local_binary_bits(index, num_bits):
    """Convert integer index to binary list of length num_bits."""
    bits = []
    for i in range(num_bits - 1, -1, -1):
        bits.append((index >> i) & 1)
    return bits
```

### Edge Cases and Pitfalls
*   **Arbitrary Internal Nodes:** Placing fusion markers at arbitrary internal nodes (not just last-level) requires that all branches contain identical subtrees over the non-coarsened dimensions. This is complex to verify. The method restricts to last-level coarsening to avoid this [S1, unofficial].
*   **Full Coarsening:** If all split dimensions are removed ($\vec{b}(v') = \vec{0}$), the node becomes a leaf. The algorithm must explicitly map all its children to the parent's index, as no child nodes are emitted [S1, unofficial].
*   **Mapping Consistency:** The naive top-down construction algorithm may fail to find mappings for subsequent members of a fusion group. The algorithm must explicitly identify all members of the fused group when the first member is emitted [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of nodes in the initial tree, assuming the marker-sweep and grouping operations are efficient. The cost of identifying fused groups is amortized over the mapping process [S1, unofficial].
*   **Memory Complexity:** Requires storage for the initial tree, the target tree descriptor, and the mapping array. The mapping array size is proportional to the number of nodes in the initial tree.
*   **Strengths:**
    *   Allows dynamic compression and simplification of omnitrees.
    *   Integrates seamlessly with the refinement algorithm's marker-sweep machinery.
    *   Guarantees correct fusion for last-level changes without complex subtree validation.
*   **Limitations:**
    *   Restricted to last-level coarsening in the standard presentation. General coarsening requires additional checks for subtree symmetry.
    *   Requires maintaining a mapping between initial and target node indices, which adds overhead.

## Background
Omnitree coarsening was introduced as part of the framework for fully dynamic omnitrees, specifically in the context of moment-conserving anisotropic compression with wavelets. It serves as the inverse operation to omnitree refinement, enabling the tree structure to adapt to changing data requirements by removing unnecessary detail. The method relies on the unique, shallowest representation of the omnitree, where splitting labels are normalized to gather splits as high as possible.

## Sources
[S1] Towards Fully Dynamic Omnitrees: Moment-Conserving Anisotropic Compression With Wavelets, https://doi.org/10.48550/arxiv.2607.04881, fetched 2026-10-26 (unofficial)
