# Brood-based Vector Quantization

Also called: Residual VQ, VQ compression

## Overview
Brood-based Vector Quantization is a lossy compression technique for spatially coherent data stored in adaptive trees (such as quadtrees or octrees). It solves the problem of storing inter-level data residuals by grouping them into "broods" (blocks of $2^d$ children) and replacing each block with a single codebook index [S1, unofficial]. It sets itself apart from standard VQ applied to uniform grids by operating on the residuals of an adaptive tree structure, which yields significantly less error for the same bit rate because it corrects coarse-level errors in finer levels and exploits spatial coherence [S1, unofficial]. It builds on [Vector Quantization](907-vector-quantization.md) by applying it specifically to residual vectors derived from tree broods.

## How it works
**Inputs and Outputs:**
*   **Input:** An adaptive primal tree $T$ containing spatially coherent data (e.g., a single-channel image or volume).
*   **Output:** A VQ tree $T'$, which is one level shorter than $T$. Each leaf node in $T'$ stores a codebook index that encodes the residuals for a brood of samples in the leaves of $T$.

**Data Structures:**
*   **VQ Tree $T'$:** A tree structure where each node stores a codebook index.
    *   **Internal Nodes:** Store a 7-bit index (for a codebook size of 128).
    *   **Leaves at Finest Level $L$:** Store an 8-bit index (for a codebook size of 256).
*   **Codebook:** A set of $K$ codevectors. Each codevector is a vector of residuals.
    *   **Dimension:** For 2D single-channel data, the residual vector $v$ has dimension 4 (representing the 4 pixel residuals of a brood). For color images, it is 12-dimensional [S1, unofficial].
    *   **Quantization:** Codevectors are quantized to 8 bits per coordinate to reduce storage size [S1, unofficial].

**Steps:**
1.  **Compute Residuals:** For each node in the original tree $T$ (except the root and finest leaves), compute the residual vector $v$ consisting of the differences between the children's values and the parent's value. A brood consists of the $2^d$ children of a parent node.
2.  **Cluster Residuals:** For each level of the tree (except the finest level $L$), collect all residual vectors from the broods at that level. Apply $k$-means clustering to these vectors to generate a codebook of size $K=128$ [S1, unofficial].
    *   Use cluster teleportation to escape local minima during $k$-means convergence [S1, unofficial].
    *   Quantize the resulting codevectors to 8 bits per coordinate [S1, unofficial].
3.  **Quantize Finest Level:** For the finest level $L$, all nodes are leaves. Omit the leaf-child bit. Use a separate codebook of size $K=256$ with 8-bit indices [S1, unofficial].
4.  **Encode Indices:** Replace each brood's residual vector in the original tree with the index of the nearest codevector in the corresponding level's codebook. Store these indices in the VQ tree $T'$.

**Formulas:**
*   **Residual Vector:** For a brood of children $c_1, \dots, c_{2^d}$ and parent $p$, the residual vector $v$ is:
    $$ v = [c_1 - p, c_2 - p, \dots, c_{2^d} - p] $$
*   **Distance Metric:** The distance between a residual vector $v$ and a codevector $c$ is typically the Euclidean distance (implied by $k$-means minimization of intra-cluster variance) [S1, unofficial].

**Parameters:**
*   **Codebook Size:** 128 elements for internal levels (7-bit index). 256 elements for the finest level (8-bit index) [S1, unofficial].
*   **Codevector Precision:** 8 bits per coordinate [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Undefined Data:** Some subtrees may extend beyond the defined spatial domain, resulting in undefined residual values. These must be handled by ignoring undefined components during distance computation and centroid updates (see [VQ with Undefined Data](8007-vq-with-undefined-data.md)) [S1, unofficial].
*   **Local Minima:** $k$-means can converge to local minima. Cluster teleportation is recommended to improve codebook quality [S1, unofficial].
*   **Precision:** Quantizing codevectors to 8 bits per coordinate reduces accuracy but is necessary for compact storage [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def brood_based_vq(tree T, d=2):
    """
    Compresses tree T using brood-based vector quantization.
    Returns VQ tree T'.
    """
    T_prime = new Tree()
    levels = get_all_levels(T)
    
    # Process each level except the finest
    for level in levels[:-1]:
        residuals = []
        nodes = get_nodes_at_level(T, level)
        
        # Collect residuals for all broods at this level
        for node in nodes:
            children = get_children(node)
            if len(children) == 2**d:
                # Compute residual vector
                v = [child.value - node.value for child in children]
                residuals.append(v)
        
        # Compute codebook using k-means
        codebook = k_means(residuals, k=128, teleportation=True)
        # Quantize codevectors to 8 bits per coordinate
        codebook = quantize_codevectors(codebook, bits=8)
        
        # Store codebook in T' (implementation detail: map level to codebook)
        store_codebook(T_prime, level, codebook)
        
        # Quantize residuals to indices
        for i, node in enumerate(nodes):
            children = get_children(node)
            if len(children) == 2**d:
                v = [child.value - node.value for child in children]
                index = find_nearest_codebook_index(v, codebook)
                # Create corresponding node in T'
                create_node_in_T_prime(T_prime, level, i, index)

    # Process finest level L
    finest_level = levels[-1]
    nodes = get_nodes_at_level(T, finest_level)
    residuals = []
    
    # For finest level, residuals are relative to parent (which is at level-1)
    # However, the source implies we treat leaves as broods of their parent's children
    # The source says "at the finest level L... use an 8-bit codebook index"
    # We assume residuals are computed similarly, but with a larger codebook
    
    for node in nodes:
        parent = get_parent(node)
        if parent is not None:
            # Get all siblings of this node
            siblings = get_siblings(node)
            # Compute residual relative to parent
            v = [sibling.value - parent.value for sibling in siblings]
            residuals.append(v)
            
    codebook_L = k_means(residuals, k=256, teleportation=True)
    codebook_L = quantize_codevectors(codebook_L, bits=8)
    store_codebook(T_prime, finest_level, codebook_L)
    
    for i, node in enumerate(nodes):
        parent = get_parent(node)
        siblings = get_siblings(node)
        v = [sibling.value - parent.value for sibling in siblings]
        index = find_nearest_codebook_index(v, codebook_L)
        create_node_in_T_prime(T_prime, finest_level, i, index)
        
    return T_prime

def k_means(data, k, teleportation=False):
    """
    Standard k-means clustering with optional teleportation.
    Returns list of k codevectors.
    """
    # Initialize centroids (e.g., random selection or k-means++)
    centroids = initialize_centroids(data, k)
    
    while not converged(centroids, data):
        # Assign each data point to nearest centroid
        assignments = assign_clusters(data, centroids)
        
        # Update centroids
        new_centroids = update_centroids(data, assignments, k)
        
        # Teleportation step to escape local minima [S1, unofficial]
        if teleportation and should_teleport(centroids, new_centroids):
            new_centroids = teleport(centroids, new_centroids)
            
        centroids = new_centroids
        
    return centroids

def quantize_codevectors(codebook, bits=8):
    """
    Quantize each coordinate of each codevector to 'bits' precision.
    """
    quantized = []
    for cv in codebook:
        q_cv = [quantize_coord(c, bits) for c in cv]
        quantized.append(q_cv)
    return quantized

def quantize_coord(value, bits):
    """
    Quantize a single float value to 'bits' precision.
    [G] Assuming symmetric range [-1, 1] or similar; exact range depends on data normalization.
    """
    # Placeholder for specific quantization scheme (e.g., uniform or adaptive)
    # Not given in sources. Assume standard uniform quantization.
    min_val = -1.0
    max_val = 1.0
    levels = 2**bits
    normalized = (value - min_val) / (max_val - min_val)
    index = int(normalized * levels)
    index = max(0, min(levels - 1, index))
    return min_val + (index / levels) * (max_val - min_val)
```

## Subtopics

### [Adaptive VQ Tree Construction](7771-adaptive-vq-tree-construction.md)
Adaptive VQ Tree Construction is a method for compressing spatially coherent data by building a simplified, error-bounded tree structure on top of a complete Vector Quantization (VQ) compressed tree. It solves the problem of achieving a specific target compression rate or maximum approximation error by selectively removing ("pruning") fine-level detail nodes that contribute negligible error. It sets itself apart from standard VQ tree methods by dynamically altering the tree topology based on a global error tolerance $\tau$, rather than using a fixed resolution hierarchy.

### [VQ with Undefined Data](8007-vq-with-undefined-data.md)
This method extends vector quantization (VQ) to handle residual blocks where some data values are undefined, such as those falling outside the defined spatial domain. It solves the problem of introducing quantization error for non-existent data by ignoring undefined components during both distance computation and centroid updates. It differs from standard VQ by modifying the $k$-means convergence steps to be robust to missing data, and from [Adaptive VQ Tree Construction](7771-adaptive-vq-tree-construction.md) by focusing specifically on the data handling aspect rather than tree pruning logic.

## Operating characteristics
*   **Compression Ratio:** For a 2D single-channel image, a codebook of 128 elements yields 1.75 bits per data sample at the finest level, or 2.33 bits per data sample when accounting for all levels [S1, unofficial].
*   **Accuracy:** Tree-based VQ yields significantly less error than VQ applied to a uniform grid of blocks [S1, unofficial].
*   **Complexity:** Time complexity is dominated by $k$-means clustering at each level. Memory complexity is proportional to the size of the VQ tree plus the codebooks.
*   **Strengths:** Efficient random access, low bit rate, good visual quality for spatially coherent data.
*   **Limitations:** Lossy compression, requires adaptive tree structure, sensitive to local minima in clustering.

## Background
Brood-based Vector Quantization was introduced by Sylvain Lefebvre and Hugues Hoppe in their 2007 paper "Compressed Random-Access Trees for Spatially Coherent Data" [S1, unofficial]. It is used in the context of compressed random-access trees for spatially coherent data, such as images and volumes.

## Sources
[S1] Compressed Random-Access Trees for Spatially Coherent Data, https://doi.org/10.2312/egwr/egsr07/339-349, fetched 2026-10-26 (unofficial)
