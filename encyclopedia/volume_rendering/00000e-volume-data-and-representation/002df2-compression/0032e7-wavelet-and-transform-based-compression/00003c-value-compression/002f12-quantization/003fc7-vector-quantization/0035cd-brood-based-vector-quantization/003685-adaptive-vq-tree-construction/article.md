# Adaptive VQ Tree Construction

Also called: Error-bounded VQ Tree, Pruning VQ Tree

## Overview
Adaptive VQ Tree Construction is a method for compressing spatially coherent data by building a simplified, error-bounded tree structure on top of a complete Vector Quantization (VQ) compressed tree. It solves the problem of achieving a specific target compression rate or maximum approximation error by selectively removing ("pruning") fine-level detail nodes that contribute negligible error. It sets itself apart from standard VQ tree methods by dynamically altering the tree topology based on a global error tolerance $\tau$, rather than using a fixed resolution hierarchy [S1, unofficial].

## How it works
The method constructs a simplified tree $T'$ from a complete VQ tree $T$ through three main phases: mipmap construction, VQ compression, and adaptive pruning.

**Inputs:**
- Original data values at the finest level $L$.
- Target approximation error tolerance $\tau$ (or target compression rate).
- Subdivision scheme (e.g., primal or dual subdivision) defining interpolation weights.

**Outputs:**
- A simplified tree $T'$ with pruned nodes.
- VQ codebooks for each level (or a shared codebook).
- Indices mapping pruned nodes to their coarse approximations.

**Data Structures:**
- **Tree Node:** Contains value $v$, residual $r$, codebook index $idx$, and pointers to children.
- **Codebook:** A set of representative residual vectors.
- **Pruning State:** A boolean flag or list indicating which broods are pruned.

**Steps:**

1.  **Mipmap Construction:**
    Create a complete mipmap tree $T$ of desired data values. Compute values $d_l$ at level $l$ from level $l+1$ via least-squares optimization:
    $$ \min_{P_l, d_l} \| d_l - P_l d_{l+1} \| $$
    where $P_l$ contains the multilinear interpolation weights (0, 1, or powers of 0.5 for primal subdivision) [S1, unofficial].

2.  **VQ Compression of Complete Tree:**
    Apply brood-based vector quantization to form a complete VQ tree $T$. Process each level $l = 1 \dots L$ in coarse-to-fine order. Compute predicted values $p$ by multilinear interpolation of parent values. Compute residuals $r = d_l - p$. Quantize residuals using codebooks to get indices [S1, unofficial].

3.  **Adaptive Pruning:**
    Prune the tree $T'$ fine-to-coarse ($l = L-1 \dots 0$).
    -   Identify candidate broods (blocks of $2^d$ leaf nodes).
    -   Visit candidate broods in order of increasing residual norm $\|\tilde{r}_l\|$ to maximize pruning of smaller residuals [S1, unofficial].
    -   For each candidate brood, check if removing it (replacing fine values with interpolated coarse values) increases the approximation error at any affected node in the original tree $T$ beyond tolerance $\tau$.
    -   The error $e$ at finer levels $l \ge l+1$ is computed by interpolating the subtracted residuals: $e = \tilde{r} - P_l \tilde{r}$ (Note: Source text implies error propagation via interpolation of residuals). Check if $\|e\| \le \tau$ [S1, unofficial].
    -   If the error bound is satisfied, prune the brood (remove $2^d$ leaf nodes). Otherwise, keep them.

**Parameters:**
-   **Tolerance $\tau$:** Maximum allowed approximation error norm. Used to bound error during pruning. Can be adjusted via binary search to hit a target compression rate [S1, unofficial].

**Pitfalls:**
-   Do not recompute codebooks on the final simplified tree. This may cause approximation errors to exceed tolerance $\tau$ due to the inter-dependence between tree structure and codebooks [S1, unofficial].
-   Pruned nodes must be interpolated from coarser values. This is more complicated in dual subdivision than primal subdivision [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_vq_tree_construction(data, tolerance_tau, subdivision_weights):
    """
    Construct an adaptive VQ tree with error bounded by tolerance_tau.
    
    Args:
        data: List of data values at finest level L.
        tolerance_tau: Maximum allowed L-norm approximation error.
        subdivision_weights: Matrix P_l containing interpolation weights for each level.
        
    Returns:
        simplified_tree: Tree structure with pruned nodes.
        codebooks: List of codebooks for each level.
    """
    # Step 1: Create complete mipmap tree T
    # Compute desired values d_l at level l from level l+1 via least-squares
    # min_{P_l, d_l} ||d_l - P_l * d_{l+1}||
    # This creates a full hierarchy of values d_l for l = 0 to L.
    complete_tree = build_mipmap_tree(data, subdivision_weights)
    
    # Step 2: Apply brood-based VQ to form complete VQ tree T
    # Process levels coarse-to-fine (l = 1 to L)
    # Compute predicted values p = P_l * a (where a is parent values)
    # Compute residuals r = d_l - p
    # Quantize residuals using codebooks to get indices
    # Store residuals and indices in complete_tree
    complete_vq_tree = apply_brood_vq(complete_tree, subdivision_weights)
    
    # Step 3: Adaptively prune T' fine-to-coarse (l = L-1 down to 0)
    simplified_tree = copy_tree_structure(complete_vq_tree)
    
    # Collect all candidate broods from finest level up
    candidate_broods = []
    for level in range(L-1, -1, -1):
        for brood in get_broods_at_level(simplified_tree, level):
            candidate_broods.append(brood)
            
    # Sort broods by increasing residual norm ||r_tilde_l||
    candidate_broods.sort(key=lambda b: norm(b.residual))
    
    for brood in candidate_broods:
        level = brood.level
        # Check if pruning this brood violates tolerance tau
        # Compute error e by interpolating subtracted residuals to finer levels
        # e = e - P_l * r_tilde (propagate error)
        # Check if ||e|| <= tau for all affected nodes
        if is_pruning_safe(brood, simplified_tree, subdivision_weights, tolerance_tau):
            prune_brood(brood, simplified_tree)
            
    return simplified_tree, complete_vq_tree.codebooks

def is_pruning_safe(brood, tree, weights, tau):
    """
    Check if removing brood exceeds tolerance tau.
    """
    # Interpolate residuals to finer levels
    # e = e - P_l * r_tilde
    # Check norm of e at all affected nodes
    affected_nodes = get_affected_nodes(brood, tree)
    for node in affected_nodes:
        error = compute_error_propagation(node, brood.residual, weights)
        if norm(error) > tau:
            return False
    return True
```

## Subtopics

### [Texture Atlas Compression with Sparse Regions](7775-texture-atlas-compression-with-sparse-regions.md)
A method to compress texture atlases containing unused space by ignoring undefined regions through adaptive pruning and sparse Vector Quantization (VQ). It sets itself apart from standard tree compression by modifying the tree structure to exclude empty areas and adapting the VQ codebook search to handle residual blocks with undefined data, preventing boundary artifacts.

### [Hybrid Vector Quantization](8634-hybrid-vector-quantization.md)
Hybrid Vector Quantization is a multiresolution compression scheme that applies Vector Quantization (VQ) to each resolution level separately to maintain multiresolution capability, while exploiting spatial redundancy by creating vectors from pixels in the three detail levels surrounding every pixel in the lowest resolution image. It sets itself apart from sibling methods like [Texture Atlas Compression with Sparse Regions](7775-texture-atlas-compression-with-sparse-regions.md) by explicitly linking vector construction across multiple wavelet detail levels (DL1, DL2, DL3) to a single low-resolution pixel, rather than treating levels or regions in isolation. This approach maintains a fairly constant Signal-to-Noise Ratio (SNR) while keeping a high compression ratio when higher detail levels are added, unlike Scheme 1 which sees a sharp drop in compression ratio.

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** Not given in the sources.
-   **Strengths:** Allows precise control over approximation error via tolerance $\tau$. Can achieve target compression rates by adjusting $\tau$ via binary search. Codebooks can be shared across levels for smooth data, reducing storage overhead (e.g., 1 KB vs 3 KB) [S1, unofficial].
-   **Limitations:** Computing per-level VQ codebooks uses all block residuals in the complete tree, even though the final simplified tree contains only a subset. Recomputing codebooks on the simplified tree is discouraged as it may exceed tolerance $\tau$ [S1, unofficial]. Pruning is more complex in dual subdivision than primal subdivision [S1, unofficial].
-   **When to use:** When a specific error bound or compression rate is required for spatially coherent data.
-   **When not to use:** When the data is not spatially coherent or when the inter-dependence between tree structure and codebooks makes error bounding difficult.

## Background
The method was introduced by S. Lefebvre and H. Hoppe in "Compressed Random-Access Trees for Spatially Coherent Data" (2007) [S1, unofficial]. It builds on the concept of brood-based vector quantization to create a simplified tree structure that maintains error bounds.

## Sources
[S1] Compressed Random-Access Trees for Spatially Coherent Data, https://doi.org/10.2312/egwr/egsr07/339-349, fetched 2026-10-26 (unofficial)
