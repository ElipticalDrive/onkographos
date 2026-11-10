# Feature Graph

Also called: Feature Graph Construction, Feature Graph building

## Overview
The Feature Graph is a hierarchical data structure for volume rendering that represents classified voxel models as a directed acyclic graph (DAG) [S1, unofficial]. It solves the problem of efficiently managing and rendering complex, multi-label volumetric data by grouping voxels into "features" based on boolean classification criteria and organizing these features into an inclusion hierarchy [S1, unofficial]. Unlike flat transfer function approaches, it explicitly models the set-inclusion relationships between voxel clusters, allowing for direct access to voxel subsets and native multiresolution support [S1, unofficial]. It is distinguished from sibling methods like [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md) by its reliance on explicit boolean label combinations rather than geometric derivatives, and from [Robust Material Graph Deduction](7459-robust-material-graph-deduction.md) by its deterministic construction from user-defined criteria rather than learned similarity maps [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A voxel volume $V$.
*   A set of $n_c$ non-exclusive classification criteria (boolean expressions) [S1, unofficial].
*   For each voxel $v$, a label vector $\mathbf{r}(v) = [r_1(v), r_2(v), \dots, r_{n_c}(v)]$, where $r_i(v) \in \{0, 1\}$ indicates if voxel $v$ satisfies criterion $i$ [S1, unofficial].

**Outputs:**
*   A Feature Graph: a DAG where nodes are features and arcs represent voxel set inclusion [S1, unofficial].
*   A Feature Hash Table: providing direct index access to graph nodes [S1, unofficial].
*   Sets of Voxels (SOVs): associated with graph nodes, stored primarily at leaf nodes [S1, unofficial].

### Data Structures

**Feature Hash Table**
*   **Purpose:** Maps label combinations to graph nodes [S1, unofficial].
*   **Layout:** A hash map or array indexed by the label vector (or a hash of it).
*   **Content:** References to Feature Node objects.

**Feature Node**
*   **Fields:**
    *   `id`: Unique identifier.
    *   `labels`: The specific subset of boolean values defining this feature [S1, unofficial].
    *   `children`: List of descendant nodes (smaller features) [S1, unofficial].
    *   `parent`: Reference to the ancestor node (larger feature) [S1, unofficial].
    *   `sovs`: List of Set of Voxels objects associated with this node at various resolution levels [S1, unofficial].
    *   `optical_properties`: Opacity $\alpha$ and color $C$ for rendering [S1, unofficial].

**Set of Voxels (SOV)**
*   **Definition:** The set of voxels belonging to a specific feature $f_i$, denoted $SOV(f_i)$ [S1, unofficial].
*   **Storage Policy:** SOVs are stored **only** in leaf nodes to avoid redundancy. The SOV of an intermediate node is the union of the SOVs of its descendants [S1, unofficial].
*   **Representation Options:**
    1.  **Submodel (Octree):** Preserves spatial ordering. Voxels not belonging to the feature are assigned zero value, ensuring a voxel has a non-zero value in only one unique leaf SOV [S1, unofficial].
    2.  **Voxel List:** Stores coordinates only. No spatial ordering. Convenient for view-aligned splatting [S1, unofficial].
    3.  **Compacted List:** For lists, stores only "black" nodes of an implicit octree with their associated voxel size [S1, unofficial].

### Algorithm: Graph Construction

The construction process consists of three main steps: Clustering, Hierarchy Creation, and Multiresolution Distribution [S1, unofficial].

#### Step 1: Clustering (Leaf Node Creation)
1.  Initialize an empty `Feature Hash Table`.
2.  Initialize an empty list of `Leaf Nodes`.
3.  Traverse the input voxel volume $V$.
4.  For each voxel $v$:
    a. Compute its label vector $\mathbf{r}(v)$.
    b. Use $\mathbf{r}(v)$ as a key to index into the `Feature Hash Table`.
    c. If the key does not exist, create a new `Feature Node` with these labels and add it to the `Feature Hash Table`.
    d. Add voxel $v$ to the `SOV` list of this node.
5.  After traversal, identify all nodes with non-empty SOVs as `Leaf Nodes` (at the highest resolution) [S1, unofficial].
6.  **Compression (Optional):**
    a. Compute the bounding box for each filled SOV.
    b. Create a voxel submodel (octree) for each SOV.
    c. Compress these submodels. If using list representation, traverse octrees to create a list-based representation with variable voxel sizes [S1, unofficial].

#### Step 2: Hierarchy Creation
1.  For each `Leaf Node` created in Step 1:
    a. Identify all possible ancestor features. An ancestor feature is defined by a subset of the leaf node's labels (i.e., a label vector with some bits set to "don't care" or implicitly 1 if the leaf has 1).
    b. Specifically, recursively create common ancestor nodes for all nodes sharing a bit in their hash code [S1, unofficial].
    c. If an ancestor node does not exist in the `Feature Hash Table`, create it.
    d. Add the current node as a child of the ancestor node.
    e. Add the ancestor node as a parent of the current node.
2.  Update the `Feature Hash Table` to reference all newly created ancestor nodes [S1, unofficial].
3.  **Root Creation:**
    a. Identify all nodes that do not have a parent (disconnected hierarchies).
    b. Create a special root node `fall` (feature fall).
    c. Set `fall` as the parent of all identified root nodes. `fall` includes all features that do not have an explicit ancestor [S1, unofficial].

#### Step 3: Multiresolution Distribution
1.  Define coarser resolution levels for the voxel volume (e.g., by grouping $2 \times 2 \times 2$ voxels into larger super-voxels).
2.  For each coarser level:
    a. Compute the label vector for each super-voxel. The label of a super-voxel can be computed by aggregating the labels of its constituent fine voxels (e.g., logical OR for boolean presence) [S1, unofficial].
    b. Repeat the clustering process (Step 1 logic) for these super-voxels to generate SOVs at the coarser resolution.
    c. Distribute these coarser SOVs through the existing graph hierarchy.
    d. Attach the coarser SOV to the corresponding feature node in the graph. Note: A feature node may contain SOVs from multiple resolution levels [S1, unofficial].
    e. **Note:** Some clusters non-empty at high resolution may become empty at lower resolution. Not all features exist at all levels. A feature may be a leaf at one resolution and an intermediate node at another [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class FeatureNode:
    def __init__(self, labels):
        self.labels = labels  # Tuple of boolean values
        self.children = []
        self.parent = None
        self.sovs = []  # List of SOV objects at different resolutions
        self.optical_properties = None

class SOV:
    def __init__(self, voxels, resolution_level):
        self.voxels = voxels  # List of voxel coordinates or octree structure
        self.resolution_level = resolution_level

def build_feature_graph(volume, labels_grid, nc):
    """
    volume: 3D array of voxel data (not strictly needed for graph structure, but implied)
    labels_grid: 4D array [X, Y, Z, nc] of boolean labels
    nc: number of classification criteria
    """
    feature_hash_table = {}
    
    # Step 1: Clustering
    # Traverse all voxels
    for x in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for z in range(volume.shape[2]):
                label_vec = tuple(labels_grid[x, y, z])
                
                # Create node if not exists
                if label_vec not in feature_hash_table:
                    feature_hash_table[label_vec] = FeatureNode(label_vec)
                
                node = feature_hash_table[label_vec]
                # Add voxel to SOV
                node.sovs.append(SOV([(x,y,z)], resolution_level=0))

    # Step 2: Hierarchy Creation
    # Get all unique label vectors present in the data
    present_labels = list(feature_hash_table.keys())
    
    # Create ancestors
    # An ancestor is defined by a subset of labels. 
    # If a node has labels (1, 0, 1), its ancestors might be (1, *, 1), (1, 0, *), etc.
    # However, the source specifies: "recursively create common ancestor nodes for all the nodes sharing a bit in their hash code"
    # And "feature (fall) that includes all the features that do not have an ancestor"
    
    # We need to generate all possible subsets of labels that could form valid ancestors.
    # A valid ancestor label vector has '1' where the descendant has '1', and '*' (or None) elsewhere.
    # The 'fall' node corresponds to all '*' (or empty set of constraints).
    
    fall_node = FeatureNode(tuple([None] * nc))
    feature_hash_table[tuple([None] * nc)] = fall_node
    
    for label_vec in present_labels:
        node = feature_hash_table[label_vec]
        
        # Generate all possible ancestor masks
        # An ancestor mask is a tuple where each element is either the value from label_vec or None
        # But we must ensure at least one bit is constrained? 
        # The source says "fall" includes features without ancestors. 
        # Let's generate all sub-masks.
        
        ancestors = get_ancestors(label_vec, nc)
        
        for ancestor_labels in ancestors:
            if ancestor_labels not in feature_hash_table:
                feature_hash_table[ancestor_labels] = FeatureNode(ancestor_labels)
            
            ancestor_node = feature_hash_table[ancestor_labels]
            
            # Add edge if not already present
            if ancestor_node not in node.children:
                node.children.append(ancestor_node)
                ancestor_node.parent = node # Note: DAG, so multiple parents possible? 
                # Source says "directed acyclic graph... arcs represent inclusion". 
                # Usually inclusion is a tree if labels are exclusive, but here they are non-exclusive.
                # So it's a DAG.
            
            # Update parent reference? In a DAG, a node can have multiple parents.
            # The source says "associate to them the computed SOVs" for leaves.
            # And "ancestor SOVs are obtained by recursively traversing descendant nodes".
            # This implies the graph structure defines the hierarchy.

    # Handle disconnected components by ensuring 'fall' is the root
    # The 'fall' node should be the parent of all nodes that don't have a more specific ancestor?
    # Actually, the 'fall' node is created for features that do not have an ancestor.
    # In our generation, every node has 'fall' as an ancestor (all None).
    # So 'fall' is the global root.

    # Step 3: Multiresolution
    # This step requires coarser labels_grid. Assuming we have a function to get coarser labels
    for level in range(1, max_resolution_levels):
        coarse_labels_grid = get_coarser_labels(labels_grid, level)
        
        # Re-cluster for this level
        # Note: We reuse the same graph structure, just adding SOVs to existing nodes
        for x in range(coarse_volume_shape[0]):
            for y in range(coarse_volume_shape[1]):
                for z in range(coarse_volume_shape[2]):
                    label_vec = tuple(coarse_labels_grid[x, y, z])
                    
                    # Find the node in the graph that matches these labels
                    # Since the graph contains all possible combinations, we look it up
                    if label_vec in feature_hash_table:
                        node = feature_hash_table[label_vec]
                        # Add coarse voxel to SOV
                        node.sovs.append(SOV([(x,y,z)], resolution_level=level))
                    else:
                        # If the combination is empty at this resolution, no SOV is added
                        pass

    return feature_hash_table[tuple([None] * nc)] # Return the fall node

def get_ancestors(label_vec, nc):
    """
    Generate all ancestor label vectors.
    An ancestor has '*' (None) in positions where the descendant has 0 or 1,
    but must have '1' where the descendant has '1'? 
    Actually, inclusion means: SOV(ancestor) contains SOV(descendant).
    If descendant has labels (1, 0), it satisfies criteria A and not B.
    Ancestor (1, *) satisfies A. SOV(1, *) contains SOV(1, 0).
    Ancestor (*, *) satisfies nothing. SOV(*, *) contains SOV(1, 0).
    """
    ancestors = []
    # Iterate through all subsets of the bits that are 1
    # And all combinations of 0s becoming *
    # Actually, simpler: any label vector L' is an ancestor of L if for all i:
    # L'[i] is None OR L'[i] == L[i]
    # AND L'[i] == 1 implies L[i] == 1? 
    # No, if L'[i] == 1 and L[i] == 0, then L' requires criterion i, but L does not satisfy it.
    # So L cannot be in SOV(L'). Thus, L' cannot be an ancestor.
    # So condition: For all i, if L'[i] == 1, then L[i] must be 1.
    # If L'[i] == 0, L[i] can be 0 or 1? 
    # If L'[i] == 0, it means "not i". If L[i] == 1, it means "is i". Contradiction.
    # So if L'[i] == 0, L[i] must be 0.
    # If L'[i] == None, L[i] can be 0 or 1.
    
    # So, L' is an ancestor of L if:
    # For all i: L'[i] == None OR L'[i] == L[i]
    
    # Generate all such L'
    import itertools
    # For each bit in L, the ancestor can have the same value or None
    options = []
    for bit in label_vec:
        if bit == 1:
            options.append([1, None])
        else: # bit == 0
            options.append([0, None])
            
    for combo in itertools.product(*options):
        ancestors.append(combo)
        
    return ancestors
```

### Parameters and Edge Cases
*   **$n_c$**: The number of classification criteria. The theoretical number of clusters is $2^{n_c}$, but many may be empty [S1, unofficial].
*   **`fall`**: The root node representing the feature with no constraints (all `None` labels). It includes all features that do not have a more specific ancestor [S1, unofficial].
*   **Empty Clusters**: Clusters that are empty at a high resolution may become empty at lower resolutions. The algorithm must handle the absence of nodes/SOVs for these cases [S1, unofficial].
*   **Precision**: When using submodels (octrees), assigning zero values to non-feature voxels ensures that a voxel has a non-zero value in only one unique leaf SOV, preserving spatial integrity [S1, unofficial].
*   **Rendering Choice**: The choice between submodel and list representation depends on the spatial distribution of the feature and the rendering algorithm (e.g., splatting favors lists) [S1, unofficial].

## Operating characteristics

*   **Time Complexity:**
    *   Construction: $O(N \cdot 2^{n_c})$ in the worst case for hierarchy creation, where $N$ is the number of voxels. However, only present label combinations are processed, so it is proportional to the number of non-empty clusters times the number of ancestors per cluster.
    *   Rendering: Depends on the SOV representation. View-aligned splatting on lists is efficient for non-spatially-ordered data [S1, unofficial].
*   **Memory Complexity:**
    *   The graph structure is compact. SOVs are stored only at leaves, reducing redundancy.
    *   Multiresolution SOVs are stored at corresponding nodes, allowing for out-of-core strategies if SOVs are stored in separate files [S1, unofficial].
*   **Strengths:**
    *   Direct access to voxel subsets via the hash table.
    *   Native multiresolution support.
    *   Efficient handling of non-exclusive labels.
    *   Compact storage of homogeneous regions via octrees [S1, unofficial].
*   **Limitations:**
    *   Requires pre-computation of labels and graph construction.
    *   Performance depends on the spatial distribution; list representation loses spatial ordering, requiring view-aligned techniques [S1, unofficial].
    *   Not suitable for dynamic data without graph updates [S1, unofficial].

## Background

The Feature Graph was introduced in the paper "Splatting multiresolution volume data using the feature graph" [S1, unofficial]. It addresses the need for efficient visualization of complex volumetric data with multiple overlapping features. The method builds on the principles of data feature-based classification, where derived features (boolean labels) are used to group voxels. It is particularly useful for medical imaging and scientific visualization where anatomical or physical structures overlap (e.g., bone and toe) [S1, unofficial].

## Sources

[S1] Splatting multiresolution volume data using the feature graph, http://hdl.handle.net/2117/86461, fetched 2026-10-26 (unofficial)
