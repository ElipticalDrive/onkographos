# Feature Graph

Also called: Feature Graph Construction, Feature Graph building

## Overview
The Feature Graph is a hierarchical data structure for volume rendering that represents classified voxel data as a directed acyclic graph (DAG) [S1, unofficial]. It solves the problem of efficiently managing and rendering complex voxel datasets by clustering voxels into features based on boolean classification criteria and organizing these clusters into a hierarchy of inclusion relationships [S1, unofficial]. Unlike flat transfer function approaches, it explicitly models the topological inclusion of voxel sets, allowing for multiresolution storage and direct access to submodels [S1, unofficial]. It is distinguished from siblings like [Texture-based Transfer Function](5450-texture-based-transfer-function.md) by its explicit graph-based hierarchy of inclusion rather than purely local feature descriptors, and from [Distance-based Transfer Function](5451-distance-based-transfer-function.md) by its use of boolean label combinations rather than geometric distance metrics.

## How it works
**Inputs and Outputs**
*   **Input:** A labeled voxel model where each voxel $v$ has a set of $n_c$ boolean labels $r_i(v)$ for $i \in [1, n_c]$, representing non-exclusive classification criteria (e.g., material type, anatomical region) [S1, unofficial].
*   **Output:** A Feature Graph DAG where nodes are features (non-empty clusters of voxels sharing specific label values) and arcs represent set inclusion (from larger ancestor sets to smaller descendant sets) [S1, unofficial]. Each node stores a Set of Voxels (SOV) at specific resolution levels.

**Data Structures**
1.  **Feature Hash Table:** A hash map providing direct index access to feature nodes. The key is the hash code derived from the voxel's label vector; the value is a reference to the feature node [S1, unofficial].
2.  **Feature Node:** Represents a feature $f$. Contains:
    *   `labels`: The subset of boolean expressions defining the feature.
    *   `children`: List of descendant nodes (sub-features).
    *   `parent`: Reference to the immediate ancestor node.
    *   `sov`: Reference to the Set of Voxels (SOV) stored at this node (only for leaf nodes in the current resolution hierarchy) [S1, unofficial].
3.  **Set of Voxels (SOV):** The collection of voxels belonging to a feature $f_i$, denoted $SOV(f_i)$.
    *   **Representation:** Can be a **Voxel Submodel** (e.g., octree) or a **Voxel List** [S1, unofficial].
    *   **Voxel Submodel:** Preserves spatial ordering. Voxels not belonging to the feature are assigned zero value, ensuring a voxel has a non-zero value in only one unique leaf SOV [S1, unofficial].
    *   **Voxel List:** Stores voxel coordinates directly. Does not preserve spatial ordering. Convenient for view-aligned splatting [S1, unofficial].
    *   **Compression:** Homogeneous regions are compacted using different voxel sizes. If using submodels, an octree is used. If using lists, only "black" nodes of the octree (representing homogeneous blocks) are stored with their associated size [S1, unofficial].

**Algorithm: Graph Construction**
The construction proceeds in three main steps: Clustering, Hierarchy Creation, and Multiresolution Distribution [S1, unofficial].

**Step 1: Clustering and SOV Creation**
1.  Traverse the labeled voxel model.
2.  For each voxel $v$, compute its hash code from its label vector $(r_1(v), ..., r_{n_c}(v))$.
3.  Insert $v$ into the SOV indexed by this hash code in the Feature Hash Table.
    *   Initially, SOVs are represented as lists of voxels [S1, unofficial].
4.  After traversal, for each non-empty SOV:
    *   Compute the bounding box.
    *   Create a voxel submodel (octree) for the SOV.
    *   Compress the submodel. Optionally, traverse the octree to create a list-based representation with variable-sized voxels [S1, unofficial].
5.  Create leaf nodes in the graph for each unique non-empty label combination and associate them with their computed SOVs [S1, unofficial].

**Step 2: Hierarchy Creation**
1.  Derive the hierarchy iteratively using boolean operations on feature labels.
2.  Recursively create common ancestor nodes for all nodes sharing a bit in their hash code.
    *   If feature $A$'s label set is a superset of feature $B$'s label set (i.e., $B$ implies $A$), then $SOV(B) \subseteq SOV(A)$. An arc is created from $A$ (ancestor) to $B$ (descendant) [S1, unofficial].
3.  Update the Feature Hash Table to reference the newly created ancestor nodes [S1, unofficial].
4.  If the graph consists of multiple disconnected hierarchies, create a root node `fall` (feature fall) that includes all features that do not have an ancestor. `fall` serves as the entry point to the graph [S1, unofficial].

**Step 3: Multiresolution Handling**
1.  Compute label sets at coarser resolution levels (larger voxels).
2.  Repeat the clustering process (Step 1) for these larger voxels.
    *   Note: Clusters non-empty at high resolution may become empty at lower resolution. Thus, not all features exist at all levels. A feature may be a leaf at one resolution and an intermediate node at another [S1, unofficial].
3.  Distribute the SOVs computed from these coarser clusters through the graph hierarchy.
4.  Attach SOVs to the corresponding nodes. Each node can contain SOVs from different resolution levels.
5.  **Storage Policy:** Store SOVs only in leaf nodes of the current resolution hierarchy to avoid redundancies. The SOV of an intermediate feature is obtained by recursively traversing its descendant nodes down to the leaves [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class FeatureNode:
    def __init__(self, labels):
        self.labels = labels  # Tuple of boolean values or subset indices
        self.children = []
        self.parent = None
        self.sov = None  # Set of Voxels, only populated for leaves

class FeatureGraph:
    def __init__(self, voxels, labels_per_voxel):
        self.voxels = voxels
        self.labels = labels_per_voxel
        self.hash_table = {}  # Maps label_hash -> FeatureNode
        self.root = None

    def build(self):
        # Step 1: Clustering
        self._cluster_voxels()
        
        # Step 2: Hierarchy Creation
        self._create_hierarchy()
        
        # Step 3: Multiresolution (simplified for single resolution here)
        # In full implementation, repeat clustering for coarser grids and attach SOVs
        pass

    def _cluster_voxels(self):
        # Traverse voxels and group by label hash
        for v in self.voxels:
            label_hash = self._compute_hash(self.labels[v])
            if label_hash not in self.hash_table:
                self.hash_table[label_hash] = FeatureNode(self.labels[v])
            
            node = self.hash_table[label_hash]
            if node.sov is None:
                node.sov = VoxelList()
            node.sov.add(v)
            
            # Create leaf nodes for unique non-empty clusters
            # Note: In the full algorithm, leaf nodes are created for the finest resolution clusters
            # and stored in a temporary list for hierarchy construction

    def _create_hierarchy(self):
        # Create leaf nodes from the hash table entries
        leaf_nodes = list(self.hash_table.values())
        
        # Build inclusion hierarchy
        # An arc exists from A to B if SOV(B) is a subset of SOV(A)
        # This corresponds to label sets: if labels(B) implies labels(A)
        # i.e., for all i, labels(B)[i] >= labels(A)[i] (if 1 means true)
        
        # Iteratively find common ancestors
        # The source describes creating common ancestors for nodes sharing a bit
        # This effectively builds a lattice of label subsets
        
        # Simplified logic:
        # 1. Identify all unique label combinations present
        # 2. For each pair (A, B), if A's labels are a superset of B's labels,
        #    add B as a child of A.
        
        # To handle the "fall" root:
        fall_node = FeatureNode(None) # Represents all features
        self.root = fall_node
        
        # Connect leaves to their immediate parents in the inclusion lattice
        # This requires comparing all pairs or using a trie-like structure on labels
        # The source mentions: "recursively create common ancestor nodes for all 
        # the nodes sharing a bit in their hash code"
        
        # Implementation detail:
        # Use the hash table to find nodes. 
        # For each node, find its parent by finding the smallest superset label set
        # that is also present in the hash table (or create it if not present but implied).
        
        # Note: The source states SOVs are stored only in leaves.
        # Intermediate nodes do not store SOVs directly but derive them from children.
        
        # Create 'fall' node if needed
        # If graph has disconnected components, 'fall' connects them.
        pass

    def _compute_hash(self, labels):
        # Convert boolean labels to a hashable key
        return tuple(labels)

class VoxelList:
    def __init__(self):
        self.voxels = []
        
    def add(self, v):
        self.voxels.append(v)
```

**Parameters**
*   $n_c$: The number of non-exclusive classification criteria (boolean conditions) [S1, unofficial].
*   `fall`: The entry point node of the graph, representing a feature that includes all features that do not have an ancestor [S1, unofficial].
*   Leaf nodes: Nodes in the graph that do not include any other feature (i.e., they have no children) [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Empty Clusters:** The theoretical number of clusters is $2^{n_c}$, but many are empty. The algorithm must handle this by only creating nodes for non-empty clusters [S1, unofficial].
*   **Multiresolution Discrepancies:** A feature may be a leaf at one resolution level and an intermediate node at another. The storage policy (storing SOVs only in leaves) must be applied per resolution level [S1, unofficial].
*   **Spatial Ordering:** If using Voxel Lists, spatial ordering is lost. This necessitates view-aligned splatting for rendering, as opposed to submodel representations which preserve ordering [S1, unofficial].
*   **Redundancy:** Storing SOVs in intermediate nodes would cause redundancy because descendant SOVs are totally included in ancestor SOVs. The algorithm strictly stores SOVs in leaves [S1, unofficial].

## Subtopics

### View-Aligned Splatting
A rendering technique used with Feature Graphs when SOVs are represented as voxel lists. It transforms and inserts voxels orderly to compensate for the lack of spatial ordering in lists [S1, unofficial].

### Multiresolution Feature Graph
An extension where SOVs are stored at different resolution levels within the hierarchy. This allows for focus+context visualizations and reduces aliasing in perspective projection [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Graph construction involves clustering ($O(N)$ where $N$ is the number of voxels) and hierarchy creation. The hierarchy creation involves boolean operations and potentially pairwise comparisons of label sets, which can be expensive if $n_c$ is large, though the hash table optimizes lookup. Multiresolution construction repeats the clustering process for coarser grids.
*   **Memory Complexity:** Memory usage depends on the SOV representation. Voxel submodels (octrees) preserve spatial ordering but may have overhead. Voxel lists are more compact for sparse data but lose spatial information. The graph structure itself adds overhead for nodes and edges. Storing SOVs only in leaves reduces redundancy [S1, unofficial].
*   **Performance:** The Feature Graph is efficient for focused features and multiresolution rendering. It reduces aliasing and supports focus+context visualizations. Performance depends on the spatial distribution of features; compact distributions benefit from submodel representations, while sparse distributions may benefit from lists [S1, unofficial].
*   **Strengths:** Explicit modeling of inclusion relationships; efficient multiresolution support; direct access to features via hash table; compact storage of homogeneous regions.
*   **Limitations:** Loss of spatial ordering in voxel list representation; potential complexity in hierarchy construction for high-dimensional label spaces; requires careful management of multiresolution levels [S1, unofficial].

## Background
The Feature Graph was introduced in the paper "Splatting multiresolution volume data using the feature graph" [S1, unofficial]. It is used in volume rendering to manage complex voxel data through hierarchical classification and multiresolution storage. The method supports efficient rendering by leveraging the inclusion relationships between features and optimizing data storage through compression and multiresolution techniques.

## Sources
[S1] Splatting multiresolution volume data using the feature graph, http://hdl.handle.net/2117/86461, fetched 2026-10-26 (unofficial)
