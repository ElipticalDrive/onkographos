# Hierarchical LOD for 3D Gaussian Splitting

Also called: 3DGS Hierarchy

## Overview
Hierarchical LOD for 3D Gaussian Splitting enables real-time rendering of very large scenes by organizing 3D Gaussian primitives into a tree-based hierarchy with interior and leaf nodes [S1]. It solves the resource limitation problem where optimizing and rendering every primitive in a massive scene is infeasible [S1]. Unlike sibling methods that may rely on static tiling or specific street-level constraints, this method constructs a dynamic Level-of-Detail (LOD) cut based on projected screen size, allowing for smooth transitions between detail levels [S1, 6143-citygaussian-vastgaussian-dogaussian.md].

## How it works
**Inputs:** A set of 3D Gaussian primitives $\{g_i\}$, each defined by mean $\mu_i$, covariance $\Sigma_i$, spherical harmonic (SH) coefficients, and opacity $o_i$.
**Outputs:** A hierarchy tree where every node (leaf or interior) is associated with a 3D Gaussian, and a rendering cut for a given view $V$.

**Data Structures:**
*   **Node:** Contains attributes $\mu, \Sigma, SH, \text{falloff}$ (opacity for interior nodes), and an Axis-Aligned Bounding Box (AABB).
*   **Hierarchy:** A binary tree structure. Leaf nodes correspond to original optimized Gaussians. Interior nodes correspond to merged Gaussians derived from their children.

**Steps:**

1.  **Build BVH:**
    *   Create an initial AABB enclosing all primitives. The extent is calculated as $3\times$ the stored size of the primitives (using the cutoff from the original 3DGS paper) [S1]. This AABB is the root node.
    *   Recursively perform a binary median split on the current node to create child nodes [S1].
    *   To split, project the means of all Gaussians in the group onto the longest axis of their bounding box.
    *   Partition the group by assigning each primitive to a child based on the position of its projected mean relative to the median of all projections [S1].

2.  **Merge Interior Nodes (Bottom-Up):**
    *   Starting from the leaves, recursively merge children to compute attributes for interior nodes [S1].
    *   Compute normalized weights $w_i$ for each child $i$:
        $$ w_i = \frac{w'_i}{\sum_{i'} w'_{i'}} $$
        where $w'_i$ are unnormalized merging weights [S1].
    *   Calculate unnormalized merging weights $w'_i$ based on the projected contribution of the child Gaussian. Assuming isotropic Gaussians with little overlap:
        $$ w'_i = o_i S_i $$
        where $S_i$ is the surface of the ellipsoid, proportional to $\sqrt{|\Sigma'_i|}$ (the square root of the determinant of the 2D covariance) [S1]. In practice, $S_i$ is computed directly as the projected surface area [S1].
    *   Compute the merged mean $\mu^{(l+1)}$:
        $$ \mu^{(l+1)} = \sum_{i}^{N} w_i \mu_i^{(l)} $$
    *   Compute the merged covariance $\Sigma^{(l+1)}$ using the parallel axis theorem:
        $$ \Sigma^{(l+1)} = \sum_{i}^{N} w_i \left( \Sigma_i^{(l)} + (\mu_i^{(l)} - \mu^{(l+1)})(\mu_i^{(l)} - \mu^{(l+1)})^T \right) $$
    *   Compute the merged SH coefficients:
        $$ SH^{(l+1)} = \sum_{i}^{N} SH_i^{(l)} w_i $$
    *   Compute the merged falloff (opacity for interior nodes):
        $$ \text{falloff}^{(l+1)} = \sum_{i}^{N} o_i^{(l)} w_i $$
        Note: This value can exceed 1. It is called "falloff" because it behaves as opacity during rendering, but resulting alpha values are clamped to 1 [S1].

3.  **Hierarchy Cut Selection:**
    *   For a given view $V$, define the granularity $\epsilon(n)$ of a node $n$ as the projected screen size of its bounding box over leaf Gaussians. Specifically, use the largest dimension of the bounding box [S1].
    *   Select a cut by including nodes $n$ whose projected AABB axis lengths $\epsilon(n)$ are smaller than a target granularity $\tau_\epsilon$ (e.g., 1 pixel) [S1].
    *   The cut is found in linear time (or constant time per node in parallel settings) because a parent's AABB is never smaller than its children's [S1].

4.  **Smooth Transition:**
    *   The cut may contain nodes that over-fulfill the granularity criterion. To ensure smooth transitions, interpolate between a node $n$ and its parent $p$ [S1].
    *   Calculate interpolation weight $t_n$ based on $\epsilon(n)$ and $\epsilon(p)$ [S1].
    *   **Pitfall Handling:** Direct interpolation of rotation axes can cause undesired rotations. Instead, render the two children using the same parameters as the parent, and progressively interpolate towards their separate values [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_hierarchy(primitives):
    """
    primitives: List of Gaussian objects with attributes:
        mu: (3,) mean position
        Sigma: (3,3) covariance matrix
        SH: Spherical Harmonic coefficients
        o: opacity
        size: stored size for AABB expansion
    """
    # Step 1: Build BVH
    root_aabb = compute_initial_aabb(primitives) # 3x stored size
    root = Node(aabb=root_aabb, children=[], is_leaf=False)
    queue = [root]
    
    while queue:
        node = queue.pop(0)
        if node.is_leaf: continue # Leaf nodes are original primitives
        
        # Project means onto longest axis of node's AABB
        longest_axis = get_longest_axis(node.aabb)
        projections = [dot(p.mu, longest_axis) for p in node.children]
        median_proj = median(projections)
        
        left_children = []
        right_children = []
        
        for i, p in enumerate(node.children):
            if projections[i] < median_proj:
                left_children.append(p)
            else:
                right_children.append(p)
                
        # Create child nodes
        left_node = create_node(left_children)
        right_node = create_node(right_children)
        
        node.children = [left_node, right_node]
        queue.append(left_node)
        queue.append(right_node)
        
    return root

def merge_node(node):
    """
    Recursively merge children to compute interior node attributes.
    Called bottom-up.
    """
    if node.is_leaf:
        return node # Leaf attributes are already optimized
        
    left = merge_node(node.children[0])
    right = merge_node(node.children[1])
    
    # Compute unnormalized weights based on projected surface
    # S_i is proportional to sqrt(det(Sigma'_i))
    w_prime_left = left.o * left.S_i
    w_prime_right = right.o * right.S_i
    
    w_sum = w_prime_left + w_prime_right
    w_left = w_prime_left / w_sum
    w_right = w_prime_right / w_sum
    
    # Merge Mean
    node.mu = w_left * left.mu + w_right * right.mu
    
    # Merge Covariance (Parallel Axis Theorem)
    diff_left = left.mu - node.mu
    diff_right = right.mu - node.mu
    
    node.Sigma = (w_left * (left.Sigma + outer_product(diff_left, diff_left)) +
                  w_right * (right.Sigma + outer_product(diff_right, diff_right)))
    
    # Merge SH
    node.SH = w_left * left.SH + w_right * right.SH
    
    # Merge Falloff (Opacity)
    node.falloff = w_left * left.o + w_right * right.o
    
    # Update AABB
    node.aabb = union_aabb(left.aabb, right.aabb)
    
    return node

def select_cut(root, view_matrix, resolution, tau_epsilon=1.0):
    """
    Select nodes for rendering based on projected granularity.
    Returns list of (node, interpolation_weight) tuples.
    """
    cut = []
    
    def traverse(node, parent_node):
        # Project AABB to screen space
        projected_bbox = project_aabb(node.aabb, view_matrix, resolution)
        # Granularity is the largest dimension of the projected bbox
        epsilon_n = max(projected_bbox.width, projected_bbox.height)
        
        # If node is smaller than threshold, include it
        if epsilon_n <= tau_epsilon:
            # Check if parent is also in cut or needs interpolation
            if parent_node is not None:
                projected_bbox_p = project_aabb(parent_node.aabb, view_matrix, resolution)
                epsilon_p = max(projected_bbox_p.width, projected_bbox_p.height)
                
                # Interpolation weight calculation
                # t_n determines how much to blend between parent and child
                # If epsilon_n is much smaller than epsilon_p, t_n approaches 1 (use child)
                # If epsilon_n is close to epsilon_p, t_n approaches 0 (use parent)
                # Implementation detail: exact formula for t_n not fully specified in sources,
                # but relies on comparing epsilon(n) and epsilon(p).
                t_n = compute_interpolation_weight(epsilon_n, epsilon_p)
                
                cut.append((node, t_n, parent_node))
            else:
                cut.append((node, 1.0, None))
        else:
            # If node is too large, do not include it or its children
            return
            
        for child in node.children:
            traverse(child, node)

    traverse(root, None)
    return cut

def render_cut(cut, view_matrix):
    """
    Render the selected cut with smooth transitions.
    """
    for node, t_n, parent_node in cut:
        if parent_node is not None:
            # Render child with parent's parameters, interpolating to child's
            render_gaussian_with_interpolation(parent_node, node, t_n, view_matrix)
        else:
            render_gaussian(node, view_matrix)
```

## Operating characteristics
*   **Time Complexity:** Hierarchy construction is linear with respect to the number of primitives for the BVH build and merging [S1]. Cut selection is linear time (or constant time per node in parallel settings) [S1].
*   **Memory Complexity:** Requires storing the hierarchy tree. Interior nodes add memory overhead but allow for significant reduction in the number of primitives rendered at a distance [S1].
*   **Performance:** Enables real-time rendering of very large datasets by reducing the number of rasterized Gaussians for distant content [S1].
*   **Strengths:** Maintains the fast rasterization routine of leaf nodes for interior nodes; provides a good tradeoff between quality and speed [S1].
*   **Limitations:** Merging strategy assumes isotropic Gaussians with little overlap for weight calculation; violations of this assumption may reduce quality at higher LOD levels [S1]. Ambiguity in rotation axes requires special handling during interpolation [S1].

## Background
This method was introduced in the paper "A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Very Large Datasets" [S1]. It addresses the problem of limited available resources for optimization when dealing with data that is much sparser than traditional radiance field captures [S1]. The method subdivides the scene into chunks, optimizes them independently, and builds a hierarchy for each chunk to enable LOD-based real-time rendering [S1].

## Sources
[S1] A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Very Large Datasets, https://www.youtube.com/watch?v=p-mb2TzVJZk, fetched 2026-10-31
