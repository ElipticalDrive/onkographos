# ComputeLocalState

Also called: Hierarchical SDF Tree State Computation

## Overview
ComputeLocalState is a procedure that determines the local state (ACTIVE, INACTIVE, or SKIPPED) of nodes in a hierarchical Signed Distance Field (SDF) tree relative to a specific spatial cell. It utilizes Lipschitz pruning to identify and ignore primitives that are provably too far from the cell's surface to affect the local distance field, thereby optimizing subsequent ray marching or evaluation steps. It differs from standard tree traversal by combining post-order evaluation with immediate state-based culling, building on the hierarchical CSG pruning framework described in [GPU Implementation of Hierarchical CSG Pruning](8626-gpu-implementation-of-hierarchical-csg-pruning.md).

## How it works
**Inputs:**
- `nodes`: An array representing the hierarchical SDF tree structure.
- `p`: The center point of the current spatial cell.
- `R`: The radius of the current spatial cell.

**Outputs:**
- The `state` field of each node in `nodes` is updated to one of: `ACTIVE`, `INACTIVE`, or `SKIPPED`.

**Data Structures:**
- **Node**: Contains fields for `state` (enum), `blendData` (tuple: operator type `OP`, blending radius `k`, sign `s`, complementary flags `ca`, `cb`), `cFlag` (sign multiplier), and `primData` (for leaf primitives).
- **Stack**: Stores tuples of `(distance, node_index)`.

**Algorithm:**
The procedure iterates through the tree nodes in post-order traversal. For each node, it evaluates the local state based on the distances of its children (or the primitive itself if a leaf) and the pruning bound derived from the cell radius $R$ and blending radius $k$.

1. Initialize an empty stack.
2. For each node index `i` in `PostOrderTraversal(nodes)`:
    a. Retrieve node `n = nodes[i]`.
    b. If `n` is a leaf:
        i. Set `n.state = ACTIVE`.
        ii. Evaluate the primitive SDF at `p` using `n.primData` to get distance `d`.
    c. If `n` is not a leaf:
        i. Pop `(b', right)` from the stack (right child).
        ii. Pop `(a', left)` from the stack (left child).
        iii. Extract `OP, k, s, ca, cb` from `n.blendData`.
        iv. Check pruning condition: if $|a' - b'| > k + 2R$:
            1. If $s \cdot a' \le s \cdot b'$, set `nodes[right].state = INACTIVE`.
            2. Else, set `nodes[left].state = INACTIVE`.
            3. Set `n.state = SKIPPED`.
        v. Else:
            1. Set `n.state = ACTIVE`.
        vi. Compute distance `d` by applying operator `OP` to transformed child distances: `d = OP(ca * a', cb * b', k)`.
    d. Push `(n.cFlag * d, i)` onto the stack.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ComputeLocalState(nodes, p, R):
    stack = []
    
    # Post-order traversal ensures children are processed before parents
    for i in PostOrderTraversal(nodes):
        n = nodes[i]
        
        if IsLeaf(n):
            n.state = ACTIVE
            # Evaluate primitive SDF at cell center p
            d = EvalPrimitive(p, n.primData)
        else:
            # Pop children results: right child first, then left
            b_prime, right_idx = stack.pop()
            a_prime, left_idx = stack.pop()
            
            # Extract blend parameters
            OP, k, s, ca, cb = n.blendData
            
            # Lipschitz pruning condition
            if abs(a_prime - b_prime) > k + 2 * R:
                if s * a_prime <= s * b_prime:
                    nodes[right_idx].state = INACTIVE
                else:
                    nodes[left_idx].state = INACTIVE
                n.state = SKIPPED
            else:
                n.state = ACTIVE
            
            # Compute combined distance
            d = OP(ca * a_prime, cb * b_prime, k)
            
        # Push current node's signed distance and index
        stack.push((n.cFlag * d, i))
```

**Parameters:**
- `R`: Cell radius. Typical values depend on the spatial partitioning resolution.
- `k`: Blending radius from the CSG operator.

**Edge Cases and Pitfalls:**
- **Precision**: The pruning condition relies on the difference between child distances. Floating-point inaccuracies may lead to incorrect pruning if $|a' - b'|$ is very close to $k + 2R$.
- **Boundary Conditions**: The method assumes the cell radius $R$ accurately bounds the region of interest. If $R$ is underestimated, valid primitives may be pruned.
- **Empty Stack**: The algorithm assumes a valid tree structure where every internal node has two children.

## Subtopics

### Far-field Culling
An optimization that replaces distant trees with a single constant node providing a lower-bound distance field. This is not differentiable but preserves the 0-isosurface. [S1]

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of nodes in the tree, $O(N)$, as each node is visited exactly once in post-order.
- **Memory Complexity**: $O(H)$ for the stack, where $H$ is the height of the tree.
- **Performance**: The procedure is fast enough to be useful in dynamic contexts where the scene is modified (e.g., parameter changes, adding/deleting sub-trees) [S1].
- **Strengths**: Efficiently prunes irrelevant parts of the SDF tree based on spatial locality and Lipschitz properties.
- **Limitations**: The pruning is conservative; it may not prune all irrelevant nodes if the Lipschitz bound is loose. The far-field approximation is not continuous or differentiable.

## Background
The method is described in "Lipschitz Pruning: Hierarchical Simplification of Primitive‐Based SDFs" [S1]. It is part of a broader effort to optimize SDF evaluation and rendering by leveraging hierarchical structures and spatial culling. The approach builds on narrow-band optimization techniques used in discrete SDF rendering.

## Sources
[S1] Lipschitz Pruning: Hierarchical Simplification of Primitive‐Based SDFs, https://doi.org/10.1111/cgf.70057, fetched 2026-10-26
