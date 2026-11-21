# Feature Graph

Also called: Feature Graph Construction, Feature Graph building

## Overview
The Feature Graph is a hierarchical data structure for volume rendering that represents classified voxel models as a directed acyclic graph (DAG) [S1, unofficial]. It solves the problem of visualizing complex semantic hierarchies where a single voxel belongs to multiple non-exclusive classes simultaneously, enabling focus+context rendering through multiresolution support [S1, unofficial]. It differs from standard single-criterion classification by explicitly modeling the subset relationships between disjoint classes of different criteria, creating a graph where nodes are features—defined as sets of values for a subset of the boolean expressions used in classification such that at least one voxel fulfills them—and edges represent inclusion [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A labeled voxel model where each voxel $v$ has a set of $n_c$ boolean labels $r_i(v)$ for $i \in [1, n_c]$, indicating fulfillment of $n_c$ non-exclusive classification criteria [S1, unofficial].
*   **Output:** A Feature Graph DAG where nodes represent features and arcs represent inclusion of voxel sets, along with a Feature Hash Table for direct node access [S1, unofficial].

**Data Structures:**
1.  **Feature Hash Table:** A hash table providing direct index access to feature nodes [S1, unofficial]. The hash code for a voxel is derived from its set of labels [S1, unofficial].
2.  **Feature Node:** Represents a feature $f$ [S1, unofficial]. Contains:
    *   Label signature (subset of boolean expressions) [S1, unofficial].
    *   Reference to a Set of Voxels (SOV) [S1, unofficial].
    *   Parent pointers (for traversal to ancestors) [G].
    *   Child pointers (for traversal to descendants) [G].
3.  **Set of Voxels (SOV):** The spatial representation of a feature [S1, unofficial]. Stored only in leaf nodes to avoid redundancy [S1, unofficial].
    *   **Representation:** Can be a list of voxels or a voxel submodel (octree) [S1, unofficial].
    *   **List-based:** Stores voxel coordinates [S1, unofficial]. No spatial ordering [S1, unofficial]. Convenient for non-compact distributions [S1, unofficial].
    *   **Submodel-based:** Octree representation [S1, unofficial]. Voxels not belonging to the feature are assigned zero value, ensuring a voxel has a non-zero value in only one unique leaf SOV [S1, unofficial]. Preserves spatial ordering [S1, unofficial].
    *   **Compaction:** Homogeneous regions within an SOV are compacted using different voxel sizes [S1, unofficial]. For submodels, an octree is used [S1, unofficial]. For lists, only black nodes of the octree with their associated size are stored [S1, unofficial].

**Algorithm Steps:**

1.  **Clustering (High Resolution):**
    *   Traverse the labeled voxel model [S1, unofficial].
    *   For each voxel $v$, compute its label set $\{r_1(v), ..., r_{n_c}(v)\}$ [S1, unofficial].
    *   Use this label set as a hash code to index into the Feature Hash Table [S1, unofficial].
    *   Add $v$ to the SOV list indexed by this hash code [S1, unofficial].
    *   After traversal, for each non-empty SOV:
        *   Compute the bounding box [S1, unofficial].
        *   Create a voxel submodel (octree) for the SOV [S1, unofficial].
        *   Compress the submodel [S1, unofficial]. Optionally, traverse the octree to create a list-based representation with variable-sized voxels [S1, unofficial].
    *   Create leaf nodes in the graph for each non-empty SOV and associate them with the computed SOVs [S1, unofficial].

2.  **Hierarchy Creation:**
    *   Derive the graph hierarchy iteratively using boolean operations on feature labels [S1, unofficial].
    *   Recursively create common ancestor nodes for all nodes sharing a bit in their hash code (i.e., features that are supersets of other features) [S1, unofficial].
    *   Update the Feature Hash Table to reference the newly created ancestor nodes [S1, unofficial].
    *   If the graph consists of multiple disconnected hierarchies, create a common root node called `fall` (feature all) that includes all features without an ancestor [S1, unofficial].
    *   **Storage Policy:** Store SOVs only in leaf nodes [S1, unofficial]. Ancestor SOVs are obtained by recursively traversing descendant nodes down to the leaves [S1, unofficial].

3.  **Multiresolution Handling:**
    *   Compute label sets at coarser resolution levels of the voxel model [S1, unofficial].
    *   Repeat the clustering process for larger voxels at each coarser level [S1, unofficial].
    *   Distribute the SOVs computed from these coarser clusters through the graph hierarchy, attaching them to corresponding nodes [S1, unofficial].
    *   Each node can contain SOVs from different resolution levels [S1, unofficial]. Note that some clusters non-empty at high resolution may become empty at lower resolution, so not all features exist at all levels [S1, unofficial]. A feature can be a leaf at one resolution and an intermediate node at another [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_bounding_box(voxels):
    min_x, min_y, min_z = float('inf'), float('inf'), float('inf')
    max_x, max_y, max_z = float('-inf'), float('-inf'), float('-inf')
    for v in voxels:
        min_x = min(min_x, v.x); max_x = max(max_x, v.x)
        min_y = min(min_y, v.y); max_y = max(max_y, v.y)
        min_z = min(min_z, v.z); max_z = max(max_z, v.z)
    return (min_x, min_y, min_z, max_x, max_y, max_z)

def build_octree(voxels, bbox):
    root = OctreeNode(bbox, voxels)
    root.build()
    return root

class OctreeNode:
    def __init__(self, bbox, voxels):
        self.bbox = bbox
        self.voxels = voxels
        self.children = [None] * 8
        self.is_leaf = len(voxels) <= 1

    def build(self):
        if self.is_leaf: return
        # Standard recursive octree subdivision omitted for brevity
        pass

class FeatureNode:
    def __init__(self, label_tuple):
        self.label_tuple = label_tuple
        self.sov = None
        self.children = []
        self.parent = None

class FeatureGraph:
    def __init__(self, voxel_model, n_c):
        self.n_c = n_c
        self.hash_table = {}
        self.nodes = []
        self.fall_node = None

    def build(self):
        clusters = {}
        for voxel in voxel_model:
            labels = tuple(voxel.get_label(i) for i in range(self.n_c))
            if labels not in clusters:
                clusters[labels] = []
            clusters[labels].append(voxel)

        leaf_nodes = []
        for labels, voxels in clusters.items():
            if not voxels: continue
            bbox = compute_bounding_box(voxels)
            octree = build_octree(voxels, bbox)
            sov = octree
            node = FeatureNode(labels)
            node.sov = sov
            self.hash_table[labels] = node
            self.nodes.append(node)
            leaf_nodes.append(node)

        self._create_hierarchy(leaf_nodes)
        self._create_fall_node()

    def _create_hierarchy(self, leaf_nodes):
        existing_sigs = set(n.label_tuple for n in leaf_nodes)
        all_sigs = set(existing_sigs)

        for bit in range(self.n_c):
            nodes_with_bit = [n for n in leaf_nodes if n.label_tuple[bit]]
            if len(nodes_with_bit) > 1:
                shared_sig = tuple(1 if i == bit else 0 for i in range(self.n_c))
                if shared_sig not in self.hash_table:
                    new_node = FeatureNode(shared_sig)
                    self.hash_table[shared_sig] = new_node
                    self.nodes.append(new_node)
                    all_sigs.add(shared_sig)

        for sig in all_sigs:
            node = self.hash_table[sig]
            for other_sig in all_sigs:
                if other_sig == sig: continue
                other_node = self.hash_table[other_sig]
                if all(s <= o for s, o in zip(sig, other_sig)):
                    if node not in other_node.children:
                        other_node.children.append(node)
                        node.parent = other_node
                elif all(o <= s for s, o in zip(sig, other_sig)):
                    if other_node not in node.children:
                        node.children.append(other_node)
                        other_node.parent = node

    def _create_fall_node(self):
        roots = [n for n in self.nodes if n.parent is None]
        if len(roots) > 1:
            self.fall_node = FeatureNode(None)
            for root in roots:
                root.parent = self.fall_node
                self.fall_node.children.append(root)
            self.nodes.append(self.fall_node)
        elif len(roots) == 1:
            self.fall_node = roots[0]
        else:
            self.fall_node = None
```

**Parameters:**
*   $n_c$: The number of non-exclusive classification criteria (boolean conditions) [S1, unofficial].
*   `fall`: The entry point node of the graph, representing a feature that includes all features that do not have an ancestor [S1, unofficial].
*   Leaf nodes: Nodes in the graph that do not include any other feature [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Clusters:** Many theoretical clusters ($2^{n_c}$) may be empty [S1, unofficial]. The algorithm only creates nodes for non-empty clusters [S1, unofficial].
*   **Multiresolution Disappearance:** Clusters non-empty at high resolution may become empty at lower resolution [S1, unofficial]. Not all features exist at all levels [S1, unofficial].
*   **Redundancy:** SOVs are stored only in leaf nodes to avoid redundancy [S1, unofficial]. Ancestor SOVs are derived by traversal [S1, unofficial].
*   **Spatial Ordering:** List-based SOVs do not preserve spatial ordering, requiring view-aligned splatting for rendering [S1, unofficial]. Submodel-based SOVs preserve ordering but may have overlapping bounding boxes [S1, unofficial].

## Subtopics

### [Multiclassified Structure Graph Construction](14781-multiclassified-structure-graph-construction.md)
Builds a directed acyclic graph representing inclusion relationships between isoclasses in a voxel model classified by multiple independent criteria. It differs from standard single-criterion classification by explicitly modeling subset relationships between disjoint classes of different criteria, creating a graph where nodes are class identifiers and edges represent inclusion.

## Operating characteristics
*   **Time Complexity:** Construction involves clustering ($O(V)$ where $V$ is number of voxels) and hierarchy creation [S1, unofficial]. Hierarchy creation depends on $n_c$ [S1, unofficial]. The number of possible features is $2^{n_c}$, but only non-empty ones are created [S1, unofficial].
*   **Memory Complexity:** Depends on the SOV representation [S1, unofficial]. List-based SOVs store only voxel coordinates [S1, unofficial]. Submodel-based SOVs store octrees [S1, unofficial]. SOVs are stored only in leaves, reducing redundancy [S1, unofficial].
*   **Performance:** View-aligned splatting is used for list-based SOVs to handle lack of spatial ordering [S1, unofficial]. Multiresolution support reduces aliasing and provides focus+context visualizations [S1, unofficial].
*   **Strengths:** Handles non-exclusive classification, supports multiresolution, allows focus+context rendering [S1, unofficial].
*   **Limitations:** List-based SOVs require view-aligned splatting [S1, unofficial]. Submodel-based SOVs may have overlapping bounding boxes [S1, unofficial].

## Background
The Feature Graph was introduced in the paper "Splatting multiresolution volume data using the feature graph" [S1, unofficial]. It is used for visualizing complex semantic hierarchies in volume data [S1, unofficial].

## Sources
[S1] Splatting multiresolution volume data using the feature graph, http://hdl.handle.net/2117/86461, fetched 2026-10-26 (unofficial)
