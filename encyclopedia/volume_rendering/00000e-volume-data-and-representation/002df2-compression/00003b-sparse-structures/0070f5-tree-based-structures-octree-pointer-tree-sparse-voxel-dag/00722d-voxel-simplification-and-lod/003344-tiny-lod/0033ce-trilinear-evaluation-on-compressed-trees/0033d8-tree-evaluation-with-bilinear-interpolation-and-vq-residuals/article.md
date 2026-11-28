# Tree Evaluation with Bilinear Interpolation and VQ Residuals

Also called: Adaptive Tree Evaluation, Primal Subdivision Evaluation

## Overview
This method evaluates a signal value at a point $\mathbf{x}$ by traversing an adaptive quadtree structure, using bilinear interpolation of node values and Vector Quantization (VQ) residuals to reconstruct the signal [S1, unofficial]. It serves as the evaluation engine for the Compressed Random-Access Tree (CRAT) framework, enabling fine-grain random queries required by rendering algorithms [S1, unofficial]. Unlike its sibling [Trilinear Evaluation on Compressed Trees](7074-trilinear-evaluation-on-compressed-trees.md), which operates in 3D with trilinear filtering, this method is designed for 2D spatially coherent data using bilinear filtering and primal subdivision logic [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A point $\mathbf{x} = (x, y)$ in 2D space, a starting level $l$ (typically the root of a forested mipmap structure), and a compressed tree structure where nodes store addresses and values [S1, unofficial].
*   **Output:** The interpolated signal value at $\mathbf{x}$ [S1, unofficial].

**Data Structures:**
*   **Node:** Contains an address (pointer/offset to children) and a value [S1, unofficial].
*   **VQ Codebook:** A lookup table storing residual values for quantized node differences [S1, unofficial].
*   **Tree Topology:** Primal subdivision structure where children of a node at location $(x, y)$ in level $l$ have locations $(x, y)$, $(x + 2^l, y)$, $(x, y + 2^l)$, and $(x + 2^l, y + 2^l)$ [S1, unofficial].

**Algorithm Steps:**
1.  At point $\mathbf{x}$, bilinearly interpolate the four node values [S1, unofficial].
2.  If depth $l < 1$ or all four node addresses are NULL, return the value at $\mathbf{x}$ linearly interpolated (lerped) with the value from the prior level using $l$ as the weight [S1, unofficial].
3.  Set the new cell as the quadrant containing point $\mathbf{x}$ [S1, unofficial].
4.  For each of the four new cell corners: Predict the new node value using bilinear interpolation [S1, unofficial].
5.  If the node parent address is non-NULL, access the VQ codebook to add the residual value [S1, unofficial].
6.  Update the node address to the appropriate child [S1, unofficial].
7.  Re-express point $\mathbf{x}$ in the new cell’s coordinates [S1, unofficial].

**Pseudocode:**
The following pseudocode generalizes the 1D adaptive primal tree evaluation to the 2D bilinear case [S1, unofficial]. Note that the 2D case requires handling four corners, but the logic per corner follows the 1D iterative refinement [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_2d_adaptive_tree(x, y, start_level, nodes, codebook):
    """
    Evaluates signal at (x, y) using adaptive quadtree with VQ residuals.
    nodes: List of 4 node objects with .addr and .val attributes.
    codebook: Function that takes an address and returns a residual float.
    start_level: Initial depth l.
    """
    l = start_level
    v_prior = 0.0  # Value from the level above; initialized to 0 if unavailable
    
    while True:
        # 1. Bilinearly interpolate the four node values
        v00, v10, v01, v11 = nodes[0].val, nodes[1].val, nodes[2].val, nodes[3].val
        v_c = bilinear(v00, v10, v01, v11, x, y)
        
        # 2. Termination check
        if l < 1 or all(n.addr is None for n in nodes):
            return lerp(v_c, v_prior, l)
        
        # 3. Set new cell as the quadrant containing point x
        qx = 1 if x >= 0.5 else 0
        qy = 1 if y >= 0.5 else 0
        quad = qy * 2 + qx  # 0:BL, 1:BR, 2:TL, 3:TR
        
        # 4 & 5. Predict new node values and add VQ residuals
        new_nodes = []
        for i in range(4):
            # Relative corner position in parent cell (0 or 1)
            px = 1 if (i & 1) else 0
            py = 1 if (i & 2) else 0
            pred = bilinear(v00, v10, v01, v11, px, py)
            
            child_addr = nodes[i].addr
            if child_addr is not None:
                residual = codebook(child_addr)
                new_val = pred + residual
                new_nodes.append(Node(addr=None, val=new_val))
            else:
                new_nodes.append(Node(addr=None, val=pred))
        
        # 6. Update node addresses to appropriate children
        nodes = new_nodes
        
        # 7. Re-express point x in the new cell's coordinates
        x = (x - qx * 0.5) * 2
        y = (y - qy * 0.5) * 2
        
        l -= 1
        v_prior = v_c

def lerp(a, b, t):
    return a + t * (b - a)

def bilinear(v00, v10, v01, v11, px, py):
    # px, py in [0, 1]
    top = lerp(v01, v11, px)
    bot = lerp(v00, v10, px)
    return lerp(bot, top, py)

class Node:
    def __init__(self, addr, val):
        self.addr = addr
        self.val = val
```

**Parameters:**
*   **VQ Codebook:** A lookup table storing residual values for quantized node differences [S1, unofficial].

**Edge Cases and Pitfalls:**
*   If node addresses are NULL or depth is insufficient ($l < 1$), the algorithm must blend with the prior level to avoid undefined behavior [S1, unofficial].
*   In primal subdivision, some subtrees may fall outside the input domain (dangling links) due to the "slanting" of the tree structure; these are ignored by the compression scheme [S1, unofficial].

## Subtopics

### [Compressed Random-Access Tree Framework](7109-compressed-random-access-tree-framework.md)
The Compressed Random-Access Tree (CRAT) framework compresses adaptive multiresolution hierarchies into a compact, randomly-accessible structure. Unlike prior schemes that use entropy coding on sequential traversals, CRAT employs fixed-rate encoding for both topology and data, enabling fine-grain random queries required by rendering algorithms. It replaces pointers with local offsets, uses a forested mipmap structure, and applies vector quantization (VQ) to inter-level residuals, storing both offsets and codebook indices as byte records.

### Primal Subdivision Tree for Adaptive Mipmapping
In 2D, children of a node at location $(x, y)$ in level $l$ have locations $(x, y)$, $(x + 2^l, y)$, $(x, y + 2^l)$, and $(x + 2^l, y + 2^l)$ [S1, unofficial]. This 'slanting' causes some subtrees to fall outside the input domain (dangling links), which the compression scheme ignores [S1, unofficial]. The core loop for evaluating a 1D adaptive primal tree iteratively initializes $l$ and $r$ as the two current nodes, computes the coarse value $v_C$, and loops: if both pruned, return $v_C$; compute midpoint $v_m$; select left or right subtree based on $x$; update $l, r, v_l, v_r$; compute fine value $v_F$; if level $\le 1.0$, blend $v_C$ and $v_F$ and return; else update $v_C=v_F$, level--, and continue [S1, unofficial]. Dual subdivision requires 3 lookups per level and large support for pruned nodes (interpolating from next-coarser level), equivalent to multiquadratic B-splines [S1, unofficial]. Primal subdivision requires only 2 lookups per level and simple multilinear interpolation when all 2 local nodes are pruned [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The number of tree levels traversed is relatively low on average due to tree adaptivity and the collapse of coarsest levels into a forested mipmap [S1, unofficial].
*   **Memory Complexity:** Fixed-rate encoding for topology and data allows for compact storage [S1, unofficial].
*   **Strengths:** Efficient random access, low lookup count (2 per level for primal vs 3 for dual), and simple interpolation in pruned areas [S1, unofficial].
*   **Limitations:** Dangling links in primal subdivision require handling [S1, unofficial].
*   **When to use:** When evaluating 2D spatially coherent data with adaptive resolution and random access patterns [S1, unofficial].

## Background
The method originates from the work on Compressed Random-Access Trees for Spatially Coherent Data [S1, unofficial]. It builds on primal subdivision concepts from [FPRJ00] and [LKS*06], contrasting with dual-subdivision approaches like those in [ZS01] and [BD02] [S1, unofficial]. The primal subdivision approach was highlighted for its efficiency in mipmap filtering compared to dual subdivision, which suffers from expensive interpolation in pruned areas [S1, unofficial].

## Sources
[S1] Compressed Random-Access Trees for Spatially Coherent Data, https://doi.org/10.2312/egwr/egsr07/339-349, fetched 2026-10-26 (unofficial)
