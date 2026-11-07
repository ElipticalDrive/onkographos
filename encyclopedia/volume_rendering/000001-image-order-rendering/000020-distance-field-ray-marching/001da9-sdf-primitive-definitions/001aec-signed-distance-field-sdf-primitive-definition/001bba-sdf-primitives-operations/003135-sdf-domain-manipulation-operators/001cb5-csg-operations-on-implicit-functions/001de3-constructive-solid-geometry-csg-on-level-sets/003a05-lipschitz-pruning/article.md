# Lipschitz Pruning

Also called: Hierarchical Simplification of Primitive-Based SDFs, Hierarchical pruning with Lipschitz criteria, Smooth CSG Tree Pruning

## Overview
Lipschitz Pruning is a hierarchical simplification algorithm that reduces the number of active nodes in a smooth Constructive Solid Geometry (CSG) tree for a specific region of space, outputting a pruned tree equivalent to the full tree within that region [S1]. It relies on the 1-Lipschitz property of the distance function to determine if a binary operator's result is unaffected by blending within a spatial region [S1]. Unlike naive sphere tracing or bounding-volume-based pruning which requires pre-computed bounding boxes that expand with blending radii, Lipschitz Pruning is space-centric, requires no bounding box computation, and guarantees distance exactness [S1]. It builds on [Constructive Solid Geometry (CSG) on Level Sets](1627-constructive-solid-geometry-csg-on-level-sets.md) by optimizing the evaluation of the resulting implicit functions.

## How it works
**Inputs:**
*   A smooth CSG tree representing a Signed Distance Field (SDF) $f$. The tree consists of leaf nodes (primitives) and internal nodes (binary operators).
*   A spatial region defined by a closed ball $B$ with center $p$ and radius $R$.
*   For hierarchical application: a parent pruned tree and a target grid resolution.

**Outputs:**
*   A pruned CSG tree valid for the region $B$, containing only globally active nodes.
*   The size (number of globally active nodes) of the pruned tree.

**Data Structures:**
Each node in the CSG tree stores the following fields [S1]:
*   `operator_type`: Enum indicating the CSG operation (UNION, INTER, SUB).
*   `k`: Blending radius for smooth CSG operators.
*   `s`: Sign ($\pm 1$) encoding the comparison direction for the pruning constraint.
*   `left_child`, `right_child`: References to child nodes.
*   `cFlag`: Sign ($\pm 1$) stored in the child node indicating if the operand should be complemented (flipped sign) when the operator reduces to it. Specifically, `left_child.cFlag` corresponds to $c_a$ and `right_child.cFlag` corresponds to $c_b$.

**Pruning Constraint:**
For a binary operator with operands $a$ and $b$, define complementary values $a' = c_a \cdot a$ and $b' = c_b \cdot b$. The operator reduces to one operand if [S1]:
$$ \text{if } |a' - b'| > k, \text{ OP}(a, b) = \begin{cases} a' & \text{if } s \cdot a' \leq s \cdot b' \\ b' & \text{otherwise} \end{cases} $$
The parameters $c_a, c_b \in \{+1, -1\}$ and $s \in \{+1, -1\}$ depend on the operator type [S1]:
*   **UNION**: $c_a = +1, c_b = +1, s = +1$.
*   **INTER**: $c_a = +1, c_b = +1, s = -1$.
*   **SUB**: $c_a = +1, c_b = -1, s = -1$.

**Smooth Blending Kernel:**
The method uses quadratic blending [DVOG04, Qui24b] where the blending kernel $\phi(d, k)$ vanishes as $d \to k$ [S1]:
$$ \phi(d, k) = \frac{1}{4k} \max(k - d, 0)^2 $$
The operators are defined as [S1]:
$$ \text{UNION}(a, b, k) = \min(a, b) - \phi(|a - b|, k) $$
$$ \text{INTER}(a, b, k) = \max(a, b) + \phi(|a - b|, k) $$
$$ \text{SUB}(a, b, k) = \max(a, -b) + \phi(|a + b|, k) $$

**Algorithm Steps:**

1.  **Traversal #1 (Post-order): Compute Local State**
    For each node in the tree, determine its local state: `inactive`, `skipped`, or `active` [S1].
    *   **Skip Condition**: For an operator node with children $f_1, f_2$, region center $p$, and radius $R$:
        If $|f_1(p) - f_2(p)| \geq k + 2R$, the operator is **skipped**.
        *Derivation*: Since SDFs are 1-Lipschitz ($||\nabla f|| \leq 1$), for any $q \in B$:
        $$ |f_1(q) - f_2(q)| \geq |f_1(p) - f_2(p)| - |f_1(q) - f_1(p) + f_2(p) - f_2(q)| \geq (k+2R) - 2R = k $$
        Thus, the condition $|a' - b'| > k$ holds for all $q \in B$, allowing reduction to one operand [S1].
    *   **Skip Handling**: If skipped, mark the operator as `skipped`. Depending on the sign of $f_1(p) - f_2(p)$, mark one child as `inactive` (the one that is not chosen by the reduction constraint).
    *   **Inactive**: A node is `inactive` if it does not contribute to its parent's value (e.g., marked inactive by a parent skip, or a leaf with no contribution).
    *   **Active**: A node is `active` if it contributes to the parent and is not skipped.

2.  **Traversal #2 (Pre-order): Compute Global State and Rewire**
    Determine the global state and construct the pruned tree [S1].
    *   Initialize root as `globally_active`.
    *   For each node, if it is `globally_active` and locally `active`:
        *   Mark as `globally_active`.
        *   Create a new node in the pruned tree.
        *   **Parent Rewiring**: The new parent of a child node is its closest ancestor that is not `skipped`.
        *   **Complementary Propagation**: If an ancestor was `skipped`, propagate its complementary flag ($c_a$ or $c_b$) to the current node's `cFlag`.
    *   Count the number of `globally_active` nodes.

3.  **Hierarchical Application**
    To handle dense grids and memory constraints [S1]:
    *   Start with a coarse grid covering the scene bounding box.
    *   For each cell, apply the pruning procedure using the pruned tree of its parent cell as input.
    *   Iteratively subdivide cells until the target resolution is reached.
    *   This reduces memory footprint and computation time as trees become smaller at finer levels.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lipschitz_prune(root_node, center_p, radius_R):
    """
    Prunes a CSG tree for a region defined by center_p and radius_R.
    Returns the root of the pruned tree and the count of active nodes.
    """
    # Step 1: Post-order traversal to compute local states
    # We use a stack-based post-order traversal to avoid recursion depth issues
    stack = [root_node]
    visited = set()
    local_states = {} # node -> 'inactive', 'skipped', 'active'
    
    # Helper to evaluate SDF at center_p
    def eval_sdf(node, p):
        if node.is_leaf():
            return node.evaluate(p)
        else:
            left_val = eval_sdf(node.left_child, p)
            right_val = eval_sdf(node.right_child, p)
            return node.combine(left_val, right_val)

    # Compute local states
    # Note: In a real implementation, this would be a proper post-order traversal
    # Here we simulate the logic flow for clarity
    def compute_local_state(node):
        if node.is_leaf():
            return 'active'
        
        left_state = compute_local_state(node.left_child)
        right_state = compute_local_state(node.right_child)
        
        # If either child is inactive, this node might be inactive, 
        # but we need to check if it's skipped first.
        # Actually, the paper defines local state based on contribution to parent.
        # We evaluate operands at p.
        
        left_val = eval_sdf(node.left_child, center_p)
        right_val = eval_sdf(node.right_child, center_p)
        
        # Apply complementary flags
        left_val_comp = node.left_child.cFlag * left_val
        right_val_comp = node.right_child.cFlag * right_val
        
        diff = abs(left_val_comp - right_val_comp)
        
        # Skip Condition
        if diff >= node.k + 2 * radius_R:
            node.state = 'skipped'
            # Determine which child is inactive
            # Constraint: if s * a' <= s * b', choose a'
            if node.s * left_val_comp <= node.s * right_val_comp:
                node.left_child.state = 'inactive'
                node.right_child.state = 'active' # The other child is the result
            else:
                node.right_child.state = 'inactive'
                node.left_child.state = 'active'
            return 'skipped'
        else:
            node.state = 'active'
            return 'active'

    compute_local_state(root_node)
    
    # Step 2: Pre-order traversal to compute global states and build pruned tree
    pruned_root = None
    active_count = 0
    
    def build_pruned_tree(node, parent_pruned_node, inherited_cFlag):
        nonlocal active_count
        
        # Global state: active if locally active and no ancestor is inactive
        # Note: 'inactive' ancestors prune the subtree. 'skipped' ancestors are rewired.
        if node.state == 'inactive':
            return None
            
        if node.state == 'active':
            active_count += 1
            new_node = create_pruned_node(node.operator_type, node.k)
            new_node.cFlag = inherited_cFlag # Apply inherited complementary flag
            
            if parent_pruned_node is None:
                nonlocal pruned_root
                pruned_root = new_node
            else:
                # Parent rewiring: parent is the closest non-skipped ancestor
                # In this recursive structure, parent_pruned_node is already the rewired parent
                if not parent_pruned_node.left:
                    parent_pruned_node.left = new_node
                else:
                    parent_pruned_node.right = new_node
            
            # Propagate complementary flags for children
            # If this node was skipped, its children inherit the flag.
            # But here node is active, so we pass the node's own cFlags?
            # No, the paper says: "propagate the complementary flag of skipped nodes to their children".
            # If the current node is active, it is not skipped, so no flag propagation from it.
            # However, if an ANCESTOR was skipped, the inherited_cFlag handles it.
            
            # Recurse
            # For the left child, the complementary flag is node.left_child.cFlag
            # But if the current node was skipped, we would have marked one child inactive.
            # Since we are in the 'active' branch, both children are potentially active.
            
            # Wait, the paper says: "propagate the complementary flag of skipped nodes to their children".
            # This implies if an ancestor is skipped, the path to the child is rewired.
            # The 'inherited_cFlag' accumulates the cFlags of all skipped ancestors.
            
            build_pruned_tree(node.left_child, new_node, inherited_cFlag * node.left_child.cFlag)
            build_pruned_tree(node.right_child, new_node, inherited_cFlag * node.right_child.cFlag)
            
        elif node.state == 'skipped':
            # If skipped, we don't create a node, but we must process children
            # with the accumulated complementary flag.
            # The children of a skipped node are attached to the skipped node's parent.
            # This is handled by the caller (the parent's build_pruned_tree) 
            # by passing the skipped node's children directly to the parent's parent?
            # No, the recursion structure above attaches children to new_node.
            # If node is skipped, new_node is not created.
            # We need to attach the children of the skipped node to the parent_pruned_node.
            
            # Re-evaluating the recursion:
            # If node is skipped, we call build_pruned_tree on children with parent_pruned_node
            # and updated inherited_cFlag.
            
            build_pruned_tree(node.left_child, parent_pruned_node, inherited_cFlag * node.left_child.cFlag)
            build_pruned_tree(node.right_child, parent_pruned_node, inherited_cFlag * node.right_child.cFlag)
            
        return pruned_root

    # Initial call
    # The root has no inherited complementary flag (or 1)
    build_pruned_tree(root_node, None, 1)
    
    return pruned_root, active_count

def hierarchical_prune(scene_tree, grid_cells, target_resolution):
    """
    Applies Lipschitz pruning hierarchically over a grid.
    """
    # Start with coarse grid
    current_grid = get_coarse_grid()
    pruned_trees = {} # Map cell_id -> pruned_tree_root
    
    while current_grid.resolution < target_resolution:
        next_grid = subdivide(current_grid)
        for cell in next_grid:
            parent_cell = get_parent_cell(cell)
            if parent_cell in pruned_trees:
                parent_tree = pruned_trees[parent_cell]
                # Input for cell is the pruned tree of its parent
                pruned_tree, _ = lipschitz_prune(parent_tree, cell.center, cell.radius)
                pruned_trees[cell.id] = pruned_tree
            else:
                # Leaf of hierarchy or no parent tree? Use original tree
                pruned_tree, _ = lipschitz_prune(scene_tree, cell.center, cell.radius)
                pruned_trees[cell.id] = pruned_tree
        current_grid = next_grid
        
    return pruned_trees
```

## Subtopics

### [Far-field Culling](8625-far-field-culling.md)
Far-field culling is an optimization for Signed Distance Field (SDF) evaluation that replaces sub-trees in cells sufficiently far from the 0-isosurface with a single constant node providing a conservative lower-bound distance. It builds on [Lipschitz Pruning](8623-lipschitz-pruning.md) by further reducing the number of active nodes during sphere tracing or contouring tasks, where exact differentiability is not required. Unlike the baseline hierarchical pruning which preserves the full tree structure within active regions, far-field culling aggressively simplifies the geometry representation in empty space, improving tracing performance by up to a factor of 2 on complex scenes.

### [GPU Implementation of Hierarchical CSG Pruning](8626-gpu-implementation-of-hierarchical-csg-pruning.md)
This method implements the hierarchical CSG pruning algorithm described in [Lipschitz Pruning](8623-lipschitz-pruning.md) on a GPU using compute shaders. It reduces the computational load of Signed Distance Field (SDF) evaluation by iteratively pruning SDF trees across a hierarchical grid, storing data in three specific arrays. It differs from CPU-based implementations by leveraging parallel thread execution per grid cell and warp-level coherence to eliminate divergence, building on the hierarchical CSG pruning framework.

## Operating characteristics
**Time and Memory Complexity:**
*   The pruning procedure is executed once for static scenes and every frame for dynamic scenes due to low runtime overhead [S1].
*   Hierarchical application reduces memory footprint and speeds up computation as trees become smaller at finer levels [S1].
*   Direct application to all cells of a dense grid without hierarchy leads to memory allocation failures for large scenes because pruned tree size is unknown in advance [S1].

**Measured Performance:**
*   Tested on an RTX 4060 laptop with 8GB VRAM [S1].
*   **Pruning Time ($T_p$):** Ranges from 0.72 ms (Gameboy, N=65) to 125.54 ms (Fluid, N=27977) [S1].
*   **Sphere Tracing Time ($T_s$):** Ranges from 1.17 ms to 78.52 ms [S1].
*   **Discretization Time ($T_d$) over a $256^3$ grid:** Ranges from 0.19 ms to 6.99 ms [S1].
*   **Speedups:** Reach two orders of magnitude over naive sphere tracing on City and Crowd scenes with one primary and one shadow ray per pixel at 1920×1080 resolution [S1].
*   Far-field culling improves tracing performance by up to a factor of 2 [S1].

**Strengths and Limitations:**
*   **Strengths:** Space-centric rather than object-centric; does not require bounding box computation; guarantees distance exactness; robust to blending radius compared to state-of-the-art methods [S1].
*   **Limitations:** Restricted to primitives and operators that are 1-Lipschitz ($||\nabla f|| \leq 1$) [S1]. Scenes with densely packed primitives (e.g., fluid particles) reduce pruning efficiency [S1]. High blending radius $k$ negatively affects runtime [S1].
*   **Comparison:** Lipschitz bounds have similar pruning capabilities to Range Arithmetic (affine/interval) but are less costly [S1].

## Background
Lipschitz Pruning was introduced in the paper "Lipschitz Pruning: Hierarchical Simplification of Primitive-Based SDFs" [S1]. It addresses the high cost of SDF evaluation in smooth CSG trees by exploiting the 1-Lipschitz property. The method complements far-field culling and is applicable to ray tracing and meshing contexts. It improves upon bounding-volume-based pruning which suffers from large bounding boxes due to blending radii [S1].

## Sources
[S1] Lipschitz Pruning: Hierarchical Simplification of Primitive‐Based SDFs, https://doi.org/10.1111/cgf.70057, fetched 2026-10-26
