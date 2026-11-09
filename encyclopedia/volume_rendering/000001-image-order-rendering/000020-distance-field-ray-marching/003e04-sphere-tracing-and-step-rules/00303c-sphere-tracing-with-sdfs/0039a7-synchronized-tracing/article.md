# Synchronized-tracing

Also called: Synchronized Tracing of Primitive-based Implicit Volumes

## Overview
Synchronized-tracing is a tile-based rendering pipeline for implicit volumes (blobtrees) that synchronizes threads within a subfrustum to allow coherent field evaluation [S1, unofficial][S2, abstract only, unofficial]. It sets itself apart from standard sphere tracing by decoupling evaluation complexity from the full blobtree size through sparse bottom-up traversal and thread synchronization via a low-resolution A-buffer [S1, unofficial][S2, abstract only, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A blobtree representing implicit surfaces composed of thousands of primitives [S1, unofficial][S2, abstract only, unofficial].
*   **Output:** Rendered image of the implicit volume.

**Data Structures:**
*   **Blobtree:** Hierarchical structure of primitives combined via CSG operators [S2, abstract only, unofficial].
*   **Low-resolution A-buffer:** Stores the primitives of interest for a given screen tile [S1, unofficial][S2, abstract only, unofficial].

**Steps:**
1.  **Bounding Volume Computation:** Compute a tight augmented bounding volume (or volume of interest) for all primitives of the blobtree using compact, smooth CSG operators extended from standard bounded operators [S1, unofficial][S2, abstract only, unofficial]. This is particularly relevant when using approximate signed distance fields with a Lipschitz bound close to 1 [S1, unofficial].
2.  **Tile Processing:** For each screen tile, populate the low-resolution A-buffer with the primitives of interest [S1, unofficial][S2, abstract only, unofficial].
3.  **Thread Synchronization:** Use the A-buffer during ray processing to synchronize threads within a subfrustum [S1, unofficial][S2, abstract only, unofficial]. This synchronization enables coherent field evaluation within workgroups [S1, unofficial][S2, abstract only, unofficial].
4.  **Sparse Traversal and Pruning:** Perform a sparse bottom-up tree traversal to prune the blobtree on-the-fly [S1, unofficial][S2, abstract only, unofficial]. This step decorrelates field evaluation complexity from the full blobtree size [S1, unofficial][S2, abstract only, unofficial].
5.  **Ray Processing:** Execute ray processing using the sphere tracing algorithm [S1, unofficial][S2, abstract only, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def synchronized_tracing(blobtree, screen_tiles):
    # Step 1: Compute tight bounding volume for the entire blobtree
    # Uses compact, smooth CSG operators extended from standard bounded operators
    # [S1, unofficial][S2, abstract only, unofficial]
    global_bounding_volume = compute_tight_augmented_bounding_volume(blobtree)

    for tile in screen_tiles:
        # Step 2: Populate low-resolution A-buffer with primitives of interest for this tile
        # [S1, unofficial][S2, abstract only, unofficial]
        a_buffer = populate_a_buffer(tile, global_bounding_volume)

        # Step 3: Synchronize threads within the subfrustum defined by the tile
        # using the A-buffer to enable coherent field evaluation
        # [S1, unofficial][S2, abstract only, unofficial]
        synchronize_threads_in_subfrustum(tile, a_buffer)

        # Step 4: Perform sparse bottom-up tree traversal to prune the blobtree
        # This decorrelates evaluation complexity from the full blobtree size
        # [S1, unofficial][S2, abstract only, unofficial]
        pruned_subtree = sparse_bottom_up_traversal(blobtree, tile, a_buffer)

        # Step 5: Execute ray processing using sphere tracing on the pruned subtree
        # [S1, unofficial][S2, abstract only, unofficial]
        render_tile(tile, pruned_subtree)
```

**Parameters:**
*   **Lipschitz Bound:** Relevant when using approximate signed distance fields; should be close to 1 [S1, unofficial].
*   **Threshold:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Approximate SDFs:** When using approximate signed distance fields, the Lipschitz bound must be considered for the correctness of the bounding volume computation [S1, unofficial].
*   **Primitive Count:** The method is designed to scale well to surfaces consisting of thousands of primitives [S1, unofficial][S2, abstract only, unofficial].

## Operating characteristics
*   **Complexity:** The method decorrelates field evaluation complexity from the full blobtree size [S1, unofficial][S2, abstract only, unofficial].
*   **Performance:** Scales well to surfaces consisting of thousands of primitives [S1, unofficial][S2, abstract only, unofficial].
*   **Strengths:** No preprocessing is required when primitive parameters are updated, making it well suited for modeling scenarios [S1, unofficial][S2, abstract only, unofficial]. Coherent field evaluation is achieved through thread synchronization [S1, unofficial][S2, abstract only, unofficial].
*   **Limitations:** Not given in the sources.

## Background
The method was introduced in the context of implicit surfaces represented by blobtrees, which allow for smooth objects of arbitrary topology through hierarchical combinations of primitives [S2, abstract only, unofficial]. It was presented as a new tile-based rendering pipeline well suited for modeling scenarios where no preprocessing is required upon primitive parameter updates [S1, unofficial][S2, abstract only, unofficial].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S2] Enhanced Sphere Tracing, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
