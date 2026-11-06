# Hierarchical Wavelet Transform on Omnitrees

Also called: Moment-conserving anisotropic compression, Omnitree, Omnitree Wavelet Compression, Wavelet compression on omnitrees

## Overview
Hierarchical Wavelet Transform on Omnitrees is a compression strategy that transforms nodal omnitree data into wavelet coefficients to enable lossless or lossy compression via pruning and tree restructuring (downsplit) [S1, unofficial]. It solves the problem of minimizing storage size for 3D volumetric data by exploiting anisotropic refinement, setting itself apart from isotropic siblings like [Wavelet-based Multiresolution Rendering](1588-wavelet-based-multiresolution-rendering.md) by explicitly defining mathematical basis functions over the omnitree structure to achieve superior approximation rates [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric function $f$ represented on a grid of resolution level $l$ (subdividing the unit cube into $2^l$ voxels per dimension) or a continuous density distribution [S1, unofficial].
*   **Output:** A set of wavelet coefficients $\vec{w}$ and the omnitree structure, optionally coarsened based on a threshold $\epsilon$ [S1, unofficial].

**Data Structures**
The omnitree node $v$ contains:
*   `splitting_label` $\vec{b}(v)$: Indicates refinement direction.
*   `refined_dimensions` $\sigma(v)$: A set of dimensions along which the node is split.
*   `children`: $2^{|\sigma(v)|}$ child nodes. In the base case, children are leaves.
*   `nodal_coefficients`: Scaling values $w_{c, \emptyset}$ associated with the children's rectangles, ordered in Z-order [S1, unofficial].

**Algorithm Steps**

1.  **Initialization:**
    *   For **lossless** compression: Initialize a VDB BoolGrid with voxel size $2^{-l}$, call `prune()` to condense the hierarchy, and construct the omnitree [S1, unofficial].
    *   For **lossy** compression: Construct the omnitree from a zero-padded voxel grid representing the density distribution [S1, unofficial].

2.  **Gather Scaling Coefficients:**
    *   For each node $v$, gather the scaling coefficients of its children into a vector $\vec{s}^v = \{w_{c, \emptyset}\}$, ordered by Z-order [S1, unofficial].

3.  **Local Transformation (Decomposition):**
    *   Apply local transformations bottom-up from leaves to root.
    *   Compute wavelet coefficients $\vec{w}^v$ for each node $v$ using the local transformation matrix $H_{|\sigma(v)|}$:
        $$ \vec{w}^v = H_{|\sigma(v)|} \cdot \vec{s}^v \quad (1) $$
    *   The root scaling coefficient holds the mean value of the function over $(0,1)^d$. All other information is encoded in wavelet coefficients at non-leaf nodes [S1, unofficial].
    *   The function is represented as:
        $$ f = w_\theta^{\text{root}} + \sum_{v: \text{nonleaf}} \sum_{\emptyset \neq \tau \subset \sigma(v)} w_\tau^v \cdot \psi_\tau^v \quad (2) $$
    *   Here, $\psi_\tau^v$ are the basis functions, and $w_\tau^v$ are the detail coefficients measuring variation along dimensions in $\tau$ [S1, unofficial].

4.  **Compression (Coarsening and Downsplit):**
    *   **Coarsening:** Prune detail coefficients $w_\tau^v$ where $|\tau| \ge 1$ if their magnitude is below threshold $\epsilon$. If a parent-of-leaves node $v$ is coarsened in dimensions $C \subseteq \sigma(v)$, prune all $w_\tau^v$ with $\tau \cap C \neq \emptyset$ [S1, unofficial].
    *   **Downsplit:** Restructure the tree to expose new coarsening opportunities. This operation factors multi-dimensional splits into cascades of lower-dimensional splits, moving parent refinements down to children without modifying the set of leaf boxes [S1, unofficial].

5.  **Reconstruction (Nodalization):**
    *   To recover nodal coefficients, apply the inverse transformation successively from larger to finer scales (root down to leaves) [S1, unofficial].
    *   Use the nodalization matrix $R_{|\sigma(v)|}$, which is the inverse of $H_{|\sigma(v)|}$:
        $$ R_{|\sigma(v)|} H_{|\sigma(v)|} = I \quad (3) $$
    *   Reconstruct nodal coefficients $\vec{s}^v$ from $\vec{w}^v$ [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_haar_matrix(dimensions_count):
    """
    Computes the Haar transformation matrix H for a given number of refined dimensions.
    H = (1/2^k) * Kronecker product of [[1, 1], [1, -1]] for k dimensions.
    """
    if dimensions_count == 0:
        return np.array([[1.0]])
    
    # Base 1D Hadamard matrix
    H_1d = np.array([[1, 1], [1, -1]])
    
    # Start with identity for the first dimension
    H = H_1d
    
    # Kronecker product for remaining dimensions
    for _ in range(1, dimensions_count):
        H = np.kron(H, H_1d)
        
    # Normalize by 2^k
    H = H / (2 ** dimensions_count)
    return H

def wavelet_transform_omnitree(node, is_root=True):
    """
    Recursively transforms nodal coefficients to wavelet coefficients bottom-up.
    node: Object containing 'children' (list of nodes or leaf values) and 'refined_dimensions' (set of ints).
    Returns: wavelet_coefficients vector.
    """
    # Base case: leaf node
    if node.is_leaf():
        # Leaf nodes do not have children to transform; 
        # their value is stored as a scaling coefficient at the parent level.
        # This function is called on parents.
        return None

    # Step 1: Gather scaling coefficients from children
    # Children are ordered in Z-order
    s_v = []
    for child in node.children:
        if child.is_leaf():
            s_v.append(child.value)
        else:
            # Recursive call for non-leaf children
            # Note: The algorithm processes bottom-up, so children are already transformed.
            # However, to compute the parent's wavelet coeffs, we need the CHILDREN's scaling coeffs.
            # The recursive call below returns the wavelet coeffs of the child, 
            # but we need the child's scaling coeffs (nodal values) to compute the parent's wavelet coeffs.
            # Correction: The standard wavelet tree transform computes wavelet coeffs from scaling coeffs.
            # The scaling coeffs of a node are derived from its children's scaling coeffs.
            # Let's assume we have access to the raw nodal values or we reconstruct them first?
            # The source says: "The result includes the scaling coefficient w_{v, \emptyset}, 
            # which is consumed at the next higher level".
            # This implies a bottom-up pass where we compute wavelet coeffs AND the new scaling coeff for the parent.
            
            # Let's refine the pseudocode to match the "bottom-up" description precisely.
            pass 

    # Revised Bottom-Up Approach:
    # We need to compute wavelet coeffs for node v from its children's scaling coeffs.
    # And we need to compute the scaling coeff for node v (which becomes the input for v's parent).
    
    # Let's assume `node.children` are leaf nodes for this specific step's input, 
    # or we have already aggregated their scaling values.
    # The source states: "gather scaling coefficients into a vector sv".
    
    k = len(node.refined_dimensions)
    if k == 0:
        # No refinement, just return the single scaling coeff? 
        # Or handle as leaf.
        return [node.value]

    # Get scaling coeffs from children
    s_v = [child.value for child in node.children]
    s_v = np.array(s_v)
    
    # Compute Transformation Matrix
    H = compute_haar_matrix(k)
    
    # Compute Wavelet Coefficients
    w_v = H @ s_v
    
    # The first element of w_v (or specific element depending on basis ordering) 
    # is the new scaling coefficient for the parent.
    # In Haar, the mean is usually the first element after normalization.
    # Source: "The result includes the scaling coefficient w_{v, \emptyset}"
    # In the matrix H, the first row is all 1/2^k. So w_v[0] is the mean.
    s_v_parent = w_v[0]
    
    # Store wavelet coeffs in node
    node.wavelet_coeffs = w_v[1:] # Detail coefficients
    
    # Return the scaling coeff for the parent level
    return s_v_parent

def reconstruct_omnitree(node, parent_scaling_coeff):
    """
    Recursively reconstructs nodal coefficients from root down to leaves.
    """
    if node.is_leaf():
        node.value = parent_scaling_coeff
        return

    k = len(node.refined_dimensions)
    H = compute_haar_matrix(k)
    R = np.linalg.inv(H) # Or use the explicit inverse structure
    
    # Combine parent scaling coeff with node's wavelet coeffs
    # w_v = [s_v_parent, w_detail_1, ..., w_detail_m]
    w_v = np.concatenate(([parent_scaling_coeff], node.wavelet_coeffs))
    
    # Reconstruct scaling coeffs for children
    s_v_children = R @ w_v
    
    # Distribute to children
    for i, child in enumerate(node.children):
        reconstruct_omnitree(child, s_v_children[i])
```

**Parameters**
*   **Threshold $\epsilon$**: Controls lossy compression tolerance. $\epsilon = 0$ for lossless. Typical range $10^{-6}$ to $1$ [S1, unofficial].
*   **Resolution level $l$**: Subdivides unit cube into $2^l$ voxels per dimension [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Coarse Resolution Sampling:** At coarse resolutions ($l=2,3$), midpoint sampling may miss object interiors, causing OpenVDB to store zero coefficients while omnitrees store one. This creates a discrepancy in storage counts for very coarse grids [S1, unofficial].
*   **Precision:** The transformation matrices involve powers of 2, which are exact in floating-point arithmetic, but accumulation errors may occur in deep trees if not handled carefully [G].

## Subtopics

### [Haar Wavelet Transform on Omnitrees](2286-haar-wavelet-transform-on-omnitrees.md)
This method transforms nodal omnitree data into wavelet coefficients to enable lossless or lossy compression via pruning and tree restructuring (downsplit). It sets itself apart from sibling methods like [Downsplit](3404-downsplit.md) and [Coarsening Compression](3537-coarsening-compression.md) by explicitly defining the mathematical basis functions and the hierarchical decomposition of piecewise constant functions over the omnitree structure.

### [Downsplit](3404-downsplit.md)
Downsplit is a tree restructuring operation in hierarchical wavelet-based volume compression that factors a multi-dimensional split into a cascade of lower-dimensional splits. It restructures an omnitree without modifying the set of leaf boxes, serving as the inverse to normalization. By moving parent refinements down to children, it exposes new coarsening opportunities that multi-dimensional splits would otherwise obscure, enabling more efficient pruning of wavelet coefficients.

### [Coarsening Compression](3537-coarsening-compression.md)
Coarsening compression is a lossy compression step that prunes small wavelet detail coefficients to reduce the number of leaf nodes in a hierarchical omnitree, while preserving the integral of the represented function. It differs from sibling methods like [Downsplit](3404-downsplit.md) by operating directly on coefficient magnitudes to merge nodes, whereas downsplit restructures the tree topology to expose new coarsening opportunities.

## Operating characteristics
*   **Time Complexity:** The transform complexity is $O(2^d N)$, linear in the number of coefficients $N$, using data stacks [S1, unofficial].
*   **Memory Complexity:** Linear in the number of coefficients. Omnitrees use significantly fewer coefficients than isotropic structures like OpenVDB. For $l > 3$, plain omnitrees use on average 8.75$\times$ fewer coefficients than OpenVDB. Downsplit reduces this further by 2.28$\times$ to 3.96$\times$ [S1, unofficial].
*   **Measured Performance:** At $l=7$, the median storage size difference between OpenVDB and downsplit omnitree with blosc2 is 27.61$\times$ [S1, unofficial].
*   **Approximation Rate:** The approximation rate $r_a(7)$ for downsplit omnitrees is 0.51, undercutting the theoretical quadtree rate of 0.60 [S1, unofficial].
*   **Strengths:** Superior lossless compression compared to isotropic structures; effective lossy compression with mass conservation; anisotropic refinement handles complex shapes efficiently.
*   **Limitations:** Performance degrades or becomes ambiguous at very coarse resolutions ($l=2,3$) due to sampling artifacts; requires careful handling of downsplit operations for optimal compression [S1, unofficial].
*   **When to Use:** When minimizing storage size for 3D volumetric data is critical, especially for anisotropic or complex geometries, and when lossless or high-fidelity lossy compression is required.
*   **When Not to Use:** For very coarse resolutions where sampling artifacts dominate, or when isotropic structures are sufficient and simpler implementation is preferred.

## Background
The method originates from the work on "Towards Fully Dynamic Omnitrees: Moment-Conserving Anisotropic Compression With Wavelets" [S1, unofficial]. It builds on the concept of omnitrees, which allow anisotropic refinement in volumetric data structures. The integration of wavelet transforms enables efficient compression by separating the mean value (scaling coefficients) from the detail information (wavelet coefficients), allowing for effective pruning and tree restructuring [S1, unofficial].

## Sources
[S1] Towards Fully Dynamic Omnitrees: Moment-Conserving Anisotropic Compression With Wavelets, https://doi.org/10.48550/arxiv.2607.04881, fetched 2026-10-26 (unofficial)
