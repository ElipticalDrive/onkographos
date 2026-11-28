# Supercube-based Partial DMV

Also called: Supercube-based sparse DMV

## Overview
Supercube-based partial DMV is a multiresolution volume data representation that encodes nested tetrahedral meshes using supercubes as high-level primitives, exploiting spatial and hierarchical coherence to create sparse partial DMVs for isosurface extraction [S1]. It differs from full DMV ($\Delta_f$) and diamond-based DMV ($\Delta_d$) by clustering diamonds into supercubes, which reduces storage overhead for sparse isosurfaces while maintaining performance comparable to diamond-based indexing [S1].

## How it works
**Inputs and Outputs:**
- Input: A full Diamond Model of Volumes ($\Delta_f$) defined over a domain, and one or more predetermined isovalue(s) [S1].
- Output: A partial DMV ($\Delta_s$) containing only the supercubes and diamonds necessary to reconstruct the isosurface(s) via selective refinement [S1].

**Data Structures:**
The representation relies on three key size parameters [S1]:
1.  $b_\delta$: Size of each refinement (typically 4 bytes) [S1].
2.  $b_v$: Size of vertex encoding (typically 6 bytes, stored as three unsigned shorts) [S1].
3.  $b_\sigma$: Size of supercube metadata (17 bytes total) [S1].

**Supercube Metadata Layout ($b_\sigma$):**
The metadata for each supercube is packed into 17 bytes [S1]:
-   **Origin:** 6 bytes [S1].
-   **Bitflags:** 7 bytes, indicating which of the possible diamonds within the supercube are encoded [S1].
-   **Pointer:** 4 bytes, pointing to the data array containing the actual diamond information [S1].

**Algorithm Steps:**
1.  **Generate Partial DMV ($\Delta_p$):** From the full DMV ($\Delta_f$), retain all diamonds whose value range intersects the predetermined isovalue(s). Retain non-intersecting diamonds only if they are ancestors of the required diamonds to maintain hierarchical integrity [S1].
2.  **Cluster into Supercubes:** Group the retained diamonds into supercubes. A supercube represents the basic unit of symmetry within the hierarchy of diamonds model and encodes all distinct types of diamonds within the hierarchy (up to scaling factors) [S1].
3.  **Encode Supercubes:** For each supercube, store the metadata ($b_\sigma$) and the associated diamond data. The data array is indexed by the supercube's origin using a hash table [S1].
4.  **Query via Selective Refinement:** To extract the isosurface, query the partial DMV using selective refinement. This extracts adaptive tetrahedral meshes and isosurfaces by traversing the retained supercubes and diamonds [S1].

**Formulas:**
Let $D$ be the density and $C$ be the concentration of the partial DMV [S1].
-   $\Delta_s$ is more compact than $\Delta_f$ when:
    $$ D < \frac{4 + 17}{C} $$
    [S1]
-   $\Delta_s$ is more compact than $\Delta_d$ when:
    $$ C > \frac{17}{6} $$
    [S1]

The overhead of $\Delta_s$ with respect to the ideal representation $\Delta_p$ is related to concentration $C$ as $(b_\sigma / C)$ bytes per diamond, whereas $\Delta_d$ has a constant overhead of $b_v$ bytes per diamond [S1].

**Parameters:**
-   $b_\delta = 4$ bytes [S1].
-   $b_v = 6$ bytes [S1].
-   $b_\sigma = 17$ bytes [S1].

**Edge Cases and Pitfalls:**
-   **Hash Table Overhead:** The storage overhead is inversely proportional to the load factor of the hash table. For $\Delta_d$, an average load factor of 73.5% results in 36% memory overhead compared to ideal values [S1]. For $\Delta_s$, an average load factor of 75% results in 33% memory overhead [S1].
-   **Inflation:** Hash table overhead can inflate $\Delta_s$ by up to 40%, potentially reducing its advantage over $\Delta_f$ when relative size differences are small [S1].
-   **Multiple Isovalues:** When encoding multiple isovalue sets, coherence among active cells of distant isovalue sets allows for efficient clustering. For example, adding a third isovalue to a dataset may increase density by only 3% and supercube concentration by 0.8 [S1].

## Subtopics

### Diamond-based Partial DMV ($\Delta_d$)
Uses indirect hashing to index diamonds directly. It is 5-6% slower than full DMV due to hashing overhead but avoids supercube clustering logic [S1].

### Full DMV ($\Delta_f$)
Directly accesses diamonds using array locations of central vertices. It is the fastest representation but lacks sparsity for isosurface extraction [S1].

## Operating characteristics
**Time and Memory Complexity:**
-   **Storage:** Supercube-based partial DMVs require an average of 25 times less storage than corresponding full DMVs across tested datasets [S1]. The largest dataset (Xmas{868}) requires only 1.3% of the samples of $\Delta_f$ and is over 65 times more compact [S1].
-   **Comparison:** Supercube-based encodings are approximately 2.3 times smaller than a corresponding diamond-based DMV [S1].
-   **Overhead:** Supercube-based encodings have very low overhead (around 13%) compared to the ideal representation $\Delta_p$ [S1].
-   **Active Front:** The supercube-based active front ($\Sigma_s$) requires 27 bytes overhead per supercube, averaging 1 byte per tetrahedron. The diamond-based active front ($\Sigma_d$) requires 7 bytes overhead per diamond, averaging 2.16 bytes per tetrahedron [S1]. $\Sigma_s$ uses less than half the storage of $\Sigma_d$ when both are hash-indexed [S1].

**Measured Performance:**
-   All three representations process approximately 300,000 diamonds per second [S1].
-   $\Delta_f$ is the fastest [S1].
-   $\Delta_d$ is 5-6% slower than $\Delta_f$ due to indirect hashing [S1].
-   $\Delta_s$ is 7.5% slower than $\Delta_f$ but within 2% performance of $\Delta_d$, despite extra processing (extracting supercube origin, diamond type, and prefix sum) [S1].
-   The supercube-based active front $\Sigma_s$ is 1-2% more efficient than the diamond-based active front $\Sigma_d$ on average [S1].

**Strengths and Limitations:**
-   **Strengths:** Significant storage reduction for sparse isosurfaces; efficient clustering of tetrahedra; low overhead per tetrahedron in active fronts [S1].
-   **Limitations:** Hash table overhead can inflate storage by up to 40%; performance is slightly lower than direct array access ($\Delta_f$) due to hashing and decoding overhead [S1].

**When to Use:**
-   Use when extracting isosurfaces from large datasets where sparsity is high and storage efficiency is critical [S1].
-   Avoid when the relative size difference between $\Delta_s$ and $\Delta_f$ is small, as hash overhead may negate the benefits [S1].

## Background
Supercubes were proposed as a high-level primitive for compactly encoding nested tetrahedral meshes generated through longest edge bisection [S1]. They represent the basic unit of symmetry within the hierarchy of diamonds model [S1]. The method exploits spatial and hierarchical coherence to associate information with a subset of diamonds in an LEB (Longest Edge Bisection) hierarchy [S1]. Potential future indexing structures include pointerless MX-octree or perfect spatial hash for static spatial data [S1].

## Sources
[S1] Supercubes: A High-Level Primitive for Diamond Hierarchies, https://doi.org/10.1109/tvcg.2009.186, fetched 2026-10-28
