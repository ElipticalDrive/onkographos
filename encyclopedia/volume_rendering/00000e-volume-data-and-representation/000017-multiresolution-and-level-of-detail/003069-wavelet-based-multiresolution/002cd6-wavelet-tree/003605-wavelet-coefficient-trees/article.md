# Wavelet Coefficient Trees

Also called: SPIHT Tree Structure, Tier-based Wavelet Tree

## Overview
Wavelet Coefficient Trees provide a hierarchical data structure for volumetric data that exploits the spatial correlation of wavelet coefficients to enable progressive compression and level-of-detail rendering [S1, S2]. Unlike standard octrees that split space uniformly, this method defines parent-child relationships based on the geometric refinement of the underlying mesh (tier-based structure) or subband decay (zero-tree structure) [S1, S2]. It sets itself apart by allowing efficient encoding of insignificant coefficients as "zero trees," where a single symbol represents an entire hierarchy of small coefficients, significantly reducing bitstream size for smooth regions [S2].

## How it works
**Inputs:** A 3D volume represented as a sequence of wavelet coefficients generated from a multiresolution decomposition (e.g., subdivision surfaces or regular grids) [S1, S2].
**Outputs:** A compressed bitstream (for coding) or a hierarchical tree structure (for rendering/compression) [S1, S2].

### Data Structures
The structure is defined by two primary models: the Tier-based Tree (for mesh-based volumes) and the Zero-Tree Hierarchy (for general subband ordering) [S1, S2].

1. **Tier-based Tree Layout:**
   * Rooted in a base mesh (Tier 0) [S1].
   * Nodes are associated with specific geometric elements of the Tier 0 mesh:
     * **Tier 0 Nodes:** Associated with Tier 0 cell centers [S1].
     * **Tier 1 Nodes:** Associated with Tier 0 face centers [S1].
     * **Tier 2 Nodes:** Associated with Tier 0 edge centers [S1].
   * Each node contains a scalar wavelet coefficient value and a list of child pointers [G].

2. **Zero-Tree Hierarchy Layout:**
   * Based on subband ordering [S2].
   * In 2D, hierarchies are defined for specific directions (e.g., x-direction subbands 1, 3, 5; y-direction subbands 2, 4) [S2].
   * Structurally similar to a Quadtree (2D) or Octree (3D) [S2].
   * Each node represents a coefficient $c$ [S2].
   * Children are coefficients at finer resolutions (higher levels) that correspond to the same spatial support [S2].

### Numbered Steps

#### A. Constructing the Tier-based Tree Structure
1. **Initialize Base Mesh:** Define the Tier 0 mesh consisting of cells, faces, and edges [S1].
2. **Assign Tier 0 Coefficients:** For each cell in the Tier 0 mesh, compute the coarsest wavelet coefficient associated with the cell center [S1].
3. **Assign Tier 1 Coefficients:** For each face in the Tier 0 mesh, compute wavelet coefficients associated with the face center [S1].
4. **Assign Tier 2 Coefficients:** For each edge in the Tier 0 mesh, compute wavelet coefficients associated with the edge center [S1].
5. **Define Parent-Child Relationships:**
   * **Tier 0 Parent:** A Tier 0 wavelet at level $l$ has three sets of children at level $l+1$: one wavelet for each subcell the Tier 0 cell is divided into, one wavelet for each face inserted to produce those subcells, and one wavelet for each internal edge required to subdivide the Tier 0 cell [S1].
   * **Tier 1 Parent:** A Tier 1 wavelet at level $l$ has two sets of children at level $l+1$: one child wavelet for each face produced by dividing the Tier 0 face, and one child wavelet for each edge introduced into the Tier 0 face [S1].
   * **Tier 2 Parent:** A Tier 2 wavelet at level $l$ has only Tier 3 descendants associated with that specific edge [S1].

#### B. Zero-Tree Coding Algorithm
1. **Initialize Threshold:** Let $T$ be the significance threshold. Set $T = \frac{1}{2} \max(|c|)$ for all coefficients $c$ in the volume [S2].
2. **Traverse Coefficients:** Iterate through the wavelet coefficients in a predefined order (coarsest to finest or by subband) [S2].
3. **Classify Coefficients:** Divide coefficients into two groups: significant ($|c| \ge T$) and insignificant ($|c| < T$) [S2].
4. **Identify Zero Trees:** For each insignificant coefficient $c$, check its entire descendant hierarchy. If $|c'| \le T$ for every coefficient $c'$ in the hierarchy, the structure is a **Zero Tree** [S2].
   * Formula: $\forall c' \in \text{Hierarchy}(c), |c'| \le T$.
5. **Encode:**
   * If a Zero Tree is found, transmit a single symbol indicating all coefficients in the hierarchy are insignificant with respect to $T$ [S2].
   * If a coefficient is significant, transmit its sign bit and its Most Significant Bit (MSB) [S2].
6. **Update Threshold:** Divide $T$ by 2 ($T \leftarrow T/2$) [S2].
7. **Refine:** On subsequent traversals, for previously significant coefficients, transmit the next bit of their magnitude (one bit per traversal) [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wavelet_coefficient_trees_encode(coefficients, hierarchy):
    # coefficients: dict mapping node_id -> float value
    # hierarchy: dict mapping parent_id -> list of child_ids
    T = 0.5 * max(abs(v) for v in coefficients.values())
    significant = {}
    insignificant = set(coefficients.keys())
    while insignificant:
        zero_tree_nodes = []
        new_significant = {}
        for node in insignificant:
            if abs(coefficients[node]) < T:
                # Check if entire descendant hierarchy is insignificant
                # Source does not specify exact traversal order or descendant check implementation [G]
                is_zero_tree = True
                for child in hierarchy.get(node, []):
                    if child in coefficients and abs(coefficients[child]) > T:
                        is_zero_tree = False
                        break
                if is_zero_tree:
                    zero_tree_nodes.append(node)
                else:
                    new_significant[node] = coefficients[node]
            else:
                new_significant[node] = coefficients[node]
        # Transmit sign bit and MSB for newly significant coefficients [S2]
        # Source does not specify exact bit-packing format [G]
        for node, val in new_significant.items():
            pass
        significant.update(new_significant)
        # Transmit zero tree symbol for identified zero trees [S2]
        # Source does not specify exact symbol encoding [G]
        # transmit_zero_tree_symbol(zero_tree_nodes)
        T /= 2
        # Refine: transmit next bit for previously significant coefficients [S2]
        # Source does not specify exact bit extraction logic [G]
        for node in significant:
            pass
        insignificant = set(new_significant)
```

### Parameters
* **Threshold $T$:** Starts at half the maximum coefficient magnitude. Halved after each traversal [S2].
* **Subband Ordering:** Determines the hierarchy links in non-mesh-based volumes (e.g., subbands 1,3,5 for x-direction; 2,4 for y-direction) [S2].

### Edge Cases and Pitfalls
* **Precision:** Coefficients must be normalized before quantization to ensure accurate thresholding [S1].
* **Boundary Conditions:** The tier-based structure relies on the validity of the underlying mesh elements (cells, faces, edges) [S1].

## Subtopics

### Zero Tree Coding
A specific encoding strategy within the Wavelet Coefficient Tree framework [S2]. It utilizes the observation that wavelet magnitudes decay at finer resolutions, meaning child coefficients are likely smaller than parents [S2]. It encodes the positions of insignificant coefficients efficiently by representing entire hierarchies of small coefficients with a single symbol [S2]. It is particularly well-suited for distance-based thresholding methods where thresholded coefficients are spatially contiguous [S2].

## Operating characteristics
* **Compression Efficiency:** High for smooth volumes or surfaces with large uniform regions, as zero trees allow skipping large numbers of insignificant coefficients [S2].
* **Complexity:** Encoding requires traversing the tree and checking significance against a dynamic threshold. Decoding is progressive, allowing reconstruction at lower quality by stopping early [S2].
* **Memory:** Requires storing the tree structure and the coefficients. The tier-based structure requires explicit storage of mesh element associations [S1].
* **Strengths:** Progressive transmission (coarse to fine detail); efficient handling of smooth data; compatibility with subdivision surfaces [S1, S2].

## Background
The concept originates from progressive image and volume compression techniques, specifically SPIHT (Set Partitioning in Hierarchical Trees) and its adaptations for volumetric subdivision meshes [S1, S2]. The tier-based structure was developed to handle the irregular connectivity of subdivision surfaces, mapping wavelet coefficients to geometric elements (cells, faces, edges) of the base mesh [S1]. Zero-tree coding was introduced for wavelet-based image compression and adapted for volumetric distance fields, leveraging the spatial decay of wavelet coefficients [S2].

## Sources
[S1] Progressive Compression of Volumetric Subdivision Meshes, https://digital.library.unt.edu/ark:/67531/metadc1405535/?q=volume data compression rendering, fetched 2026-10-28
[S2] Multiresolution Distance Volumes for Progressive Surface Compression, https://digital.library.unt.edu/ark:/67531/metadc1409942/?q=volume data compression rendering, fetched 2026-10-28
