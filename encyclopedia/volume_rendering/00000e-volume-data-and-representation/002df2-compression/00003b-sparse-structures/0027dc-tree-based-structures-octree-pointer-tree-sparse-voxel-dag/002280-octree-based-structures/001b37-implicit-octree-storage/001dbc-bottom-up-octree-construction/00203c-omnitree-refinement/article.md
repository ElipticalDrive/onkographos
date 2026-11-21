# Omnitree Refinement

Also called: Anisotropic refinement, Omnitree construction

## Overview
Omnitree refinement transforms an initial hierarchical dyadic partition of a $d$-dimensional Cartesian domain into a target partition with higher resolution in specific dimensions [S1, unofficial]. It solves the problem of anisotropically increasing the granularity of a sparse voxel representation by attaching refinement markers to nodes and propagating them through the tree structure [S1, unofficial]. The method explicitly supports per-dimension refinement levels ($\vec{m} \in \mathbb{N}^d$), allowing for non-cubic, rectangular subdomains [S1, unofficial]. It builds on [Bottom-up Octree Construction](1590-bottom-up-octree-construction.md) by providing the mechanism to adjust the topology of the resulting tree after initial construction or during dynamic updates [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
1. An initial omnitree $T$ with $N$ nodes, linearized in Z-curve order [S1, unofficial].
2. A set of refinement markers $\vec{m}(v) \in \mathbb{N}^d$ attached to nodes $v \in T$ [S1, unofficial].
3. The dimensionality $d$ [S1, unofficial].

**Outputs:**
1. A target omnitree $T'$ with $N'$ nodes [S1, unofficial].
2. An index mapping $M: \{0, \dots, N-1\} \rightarrow \mathcal{P}(\{0, \dots, N'-1\})$ that maps initial nodes to their corresponding regions in the target tree [S1, unofficial].

### Data Structures
The omnitree is stored as a binary descriptor linearized via Z-curve order [S1, unofficial].
*   **Splitting Label ($\vec{b}(v)$):** For a node at index $i$, the splitting label is a tuple of $d$ bits [S1, unofficial]. $\vec{b}(v)_j = 1$ indicates a bisection in dimension $j$ [S1, unofficial]. $\vec{b}(v) = \vec{0}$ denotes a leaf node [S1, unofficial].
*   **Marker ($\vec{m}(v)$):** A vector of $d$ non-negative integers [S1, unofficial]. $\vec{m}(v)_j$ specifies the number of *additional* refinement levels required in dimension $j$ [S1, unofficial].
*   **Rectangle ($Q_{\vec{i},\vec{\ell}}$):** Each node maps to a hyper-rectangle defined by per-dimension refinement levels $\vec{\ell} \in \mathbb{N}^d$ and index $\vec{i}$ [S1, unofficial]. The extent in dimension $j$ is $2^{-\ell_j}$ [S1, unofficial]. The domain partition satisfies $\overline{\Omega} = \bigcup_{1 \leq i \leq N} \overline{Q}_i$ with $Q_i \cap Q_j = \emptyset$ for $i \neq j$ [S1, unofficial].

### Algorithm Steps
Refinement proceeds in four distinct phases [S1, unofficial]:

1.  **Marker Attachment:**
    Assign the initial refinement markers $\vec{m}(v)$ to each node $v$ in the input tree $T$ [S1, unofficial]. If no specific refinement is requested for a node, $\vec{m}(v) = \vec{0}$ [S1, unofficial].

2.  **Bottom-Up Sweep:**
    Markers migrate upward when all siblings share a common marker or already carry a split ($b_j = 1$) in the relevant dimension [S1, unofficial]. If siblings already possess a split in dimension $j$, the upward migration is compensated by attaching a negative marker to the parent [S1, unofficial]. The marker value at the parent is updated to reflect the maximum required refinement level among siblings, adjusted for existing splits [S1, unofficial].

3.  **Top-Down Sweep:**
    Markers lifted too high are pushed back down to children where the refinement can be resolved [S1, unofficial]. If a node carries a marker $\vec{m}(v)$ that cannot be resolved by its current children (e.g., the children are leaves but $m_j > 0$), the excess refinement is pushed down to the children [S1, unofficial]. This ensures that refinement levels are applied at the deepest possible nodes that still maintain the hierarchical structure [S1, unofficial].

4.  **Tree Construction:**
    The target tree is built with preorder depth-first traversal, greedily constructing the target descriptor [S1, unofficial]. At each rectangle $q$, the initial node $s$ whose rectangle covers $q$ is located [S1, unofficial]. Child rectangles are identified by substituting trailing $\lambda$ entries in $\vec{b}(v)$ with 0 or 1, and the procedure recurses [S1, unofficial]. At leaf nodes, the subtree is expanded directly from $\vec{m}$ [S1, unofficial]. Positive markers increase branching (requiring grandchild adoption) and negative markers decrease it (requiring node culling) [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def omnitree_refinement(initial_tree, markers, d):
    """
    Refines an omnitree based on per-dimension markers.
    
    Args:
        initial_tree: List of dicts with keys 'idx', 'b' (tuple of d bits), 'm' (tuple of d ints)
        markers: Dict mapping node index to tuple of d ints (refinement request)
        d: int (dimensionality)
        
    Returns:
        target_tree: List of nodes in Z-curve order
        mapping: Dict mapping initial index to list of target indices
    """
    # Helper: Z-curve parent/child arithmetic [G]
    def get_parent(idx):
        return idx >> d
    
    def get_children(idx):
        base = idx << d
        return [base | k for k in range(1 << d)]
    
    def get_siblings(idx):
        p = get_parent(idx)
        if p is None:
            return []
        return get_children(p)

    # Step 1: Marker Attachment
    for node in initial_tree:
        node['m'] = markers.get(node['idx'], (0,) * d)
        
    # Step 2: Bottom-Up Sweep
    sorted_nodes = sorted(initial_tree, key=lambda n: n['idx'], reverse=True)
    for node in sorted_nodes:
        if node['idx'] == 0:
            continue
        parent_idx = get_parent(node['idx'])
        parent = next((n for n in initial_tree if n['idx'] == parent_idx), None)
        if parent is None:
            continue
        siblings = get_siblings(node['idx'])
        for dim in range(d):
            m_dim = node['m'][dim]
            all_split = all(s['b'][dim] == 1 for s in siblings)
            all_marker = all(s['m'][dim] == m_dim for s in siblings)
            if all_split:
                parent['m'] = list(parent['m'])
                parent['m'][dim] -= 1
                parent['m'] = tuple(parent['m'])
            elif all_marker and m_dim > 0:
                parent['m'] = list(parent['m'])
                parent['m'][dim] = max(parent['m'][dim], m_dim)
                parent['m'] = tuple(parent['m'])
                
    # Step 3: Top-Down Sweep
    sorted_nodes = sorted(initial_tree, key=lambda n: n['idx'])
    for node in sorted_nodes:
        children = get_children(node['idx'])
        if not children:
            continue
        for dim in range(d):
            m_dim = node['m'][dim]
            if m_dim > 0 and all(c['b'] == (0,)*d for c in children):
                for child in children:
                    child['m'] = list(child['m'])
                    child['m'][dim] += m_dim
                    child['m'] = tuple(child['m'])
                node['m'] = list(node['m'])
                node['m'][dim] = 0
                node['m'] = tuple(node['m'])
                
    # Step 4: Tree Construction
    target_tree = []
    mapping = {}
    target_index = 0
    
    def build_tree(node):
        nonlocal target_index
        b = node['b']
        m = node['m']
        # If initial node is a leaf, target split comes from marker
        if b == (0,)*d:
            target_b = m
        else:
            target_b = b
            
        target_node = {'idx': target_index, 'b': target_b, 'm': (0,)*d}
        target_tree.append(target_node)
        
        num_targets = 2 ** sum(target_b)
        mapping[node['idx']] = list(range(target_index, target_index + num_targets))
        target_index += num_targets
        
        if sum(target_b) == 0:
            return
            
        # Generate children by varying split dimensions
        split_dims = [j for j, val in enumerate(target_b) if val == 1]
        for k in range(2 ** len(split_dims)):
            child_b = list(target_b)
            for idx_dim, dim in enumerate(split_dims):
                child_b[dim] = (k >> idx_dim) & 1
            child_node = {'idx': target_index, 'b': tuple(child_b), 'm': (0,)*d}
            target_tree.append(child_node)
            target_index += 1
            build_tree(child_node)
            
    for node in initial_tree:
        build_tree(node)
        
    return target_tree, mapping

def normalize_omnitree(tree, d):
    """Applies normalization loop until uniqueness condition is met [S1, unofficial]."""
    while True:
        violations = []
        for node in tree:
            p_idx = node['idx'] >> d
            parent = next((n for n in tree if n['idx'] == p_idx), None)
            if parent is None:
                continue
            for dim in range(d):
                if node['b'][dim] == 1 and parent['b'][dim] == 1:
                    violations.append((node, dim))
        if not violations:
            break
        for node, dim in violations:
            node['m'] = list(node['m'])
            node['m'][dim] += 1
            node['m'] = tuple(node['m'])
            children = get_children(node['idx'])
            for child in children:
                child['m'] = list(child['m'])
                child['m'][dim] -= 1
                child['m'] = tuple(child['m'])
        tree, _ = omnitree_refinement(tree, {n['idx']: n['m'] for n in tree}, d)
    return tree
```

### Normalization
The algorithm guarantees a valid omnitree but not necessarily the unique normalized representation [S1, unofficial]. Normalization ensures that splits are gathered as high in the tree as possible [S1, unofficial]. To normalize, detect nodes that violate the uniqueness condition (i.e., a node has a split in dimension $j$ but its parent also has a split in dimension $j$) [S1, unofficial]. Attach a positive marker to the violating node and negative markers to its children [S1, unofficial]. Apply the Tree Construction step again [S1, unofficial]. Repeat until no nodes violate the uniqueness condition [S1, unofficial].

### Index Mapping
The mapping $M$ from initial tree $T$ to target tree $T'$ is defined as $M: \{0, \dots, N-1\} \rightarrow \mathcal{P}(\{0, \dots, N'-1\})$ [S1, unofficial]. For a refined leaf node $v_i$ with target refinement $\vec{b}(v_i)$, the mapping is $M[i] = \{i', i'+1, \dots, i'+2^{|\vec{b}(v_i)|}\}$ where $i'$ is the target index of $v_i$ [S1, unofficial]. For all other nodes $j > i$, the shift is $\delta = 2^{|\vec{b}(v_i)|}$: $M[j] = \{j + \delta\}$ [S1, unofficial]. This accounts for the expansion of the tree due to refinement [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision:** Marker values must be handled as integers [S1, unofficial]. Negative markers are used for compensation and must not result in negative final marker values at leaves [S1, unofficial].
*   **Boundaries:** The domain is normalized to $(0,1)^d$ [S1, unofficial]. Refinement levels $\vec{\ell}$ must ensure that indices $\vec{i}$ remain within bounds $\{0, \dots, 2^{\ell_j}-1\}$ [S1, unofficial].
*   **Degenerate Input:** If the initial tree is empty, the output is empty [G]. If markers are all zero, the output tree is identical to the input [G].
*   **Uniqueness:** Failure to normalize results in a valid but non-unique tree, which may cause inconsistencies in downstream applications like compression or rendering [S1, unofficial].

## Subtopics

### [Omnitree Coarsening](3450-omnitree-coarsening.md)
The inverse operation, reducing resolution by fusing children into parents. Chosen when simplifying a refined tree or reducing memory footprint.

### [Min/Max Tree Construction](1623-min-max-tree-construction.md)
Builds a min/max tree for acceleration. Chosen when ray casting or bounding volume queries are required.

### [Octree Construction from Structured Volumes](1854-octree-construction-from-structured-volumes.md)
Constructs an octree from a regular grid. Chosen when the input is a dense volume rather than a sparse set of markers.

### [Hierarchical Partition Optimization](1953-hierarchical-partition-optimization.md)
Optimizes the partition for error metrics. Chosen when minimizing reconstruction error is the primary goal.

### [Octree Compaction & Construction](2857-octree-compaction-construction.md)
Compacts the tree structure. Chosen when memory efficiency is critical.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with the number of nodes [S1, unofficial]. Anisotropic refinement allows for efficient handling of non-cubic features, potentially reducing $N'$ compared to isotropic refinement for elongated structures [S1, unofficial].
*   **Strengths:** Supports anisotropic refinement, allowing for efficient representation of non-cubic features [S1, unofficial]. Provides a unique normalized representation [S1, unofficial].
*   **Limitations:** Requires careful handling of markers and normalization [S1, unofficial]. The Z-curve linearization can complicate parent-child lookups [S1, unofficial].
*   **When to use:** When anisotropic refinement is needed, or when dynamic updates to the tree structure are required [S1, unofficial].
*   **When not to use:** When isotropic refinement is sufficient, or when the overhead of marker management is prohibitive [S1, unofficial].

## Background
Omnitrees were introduced to provide a flexible representation for hierarchical dyadic partitions in $d$-dimensions [S1, unofficial]. The method is described in "Towards Fully Dynamic Omnitrees: Moment-Conserving Anisotropic Compression With Wavelets" [S1, unofficial]. The approach allows for dynamic updates and anisotropic compression, making it suitable for applications such as volume rendering and scientific visualization where data may have varying levels of detail in different dimensions [S1, unofficial].

## Sources
[S1] Towards Fully Dynamic Omnitrees: Moment-Conserving Anisotropic Compression With Wavelets, https://doi.org/10.48550/arxiv.2607.04881, fetched 2026-10-26 (unofficial)
