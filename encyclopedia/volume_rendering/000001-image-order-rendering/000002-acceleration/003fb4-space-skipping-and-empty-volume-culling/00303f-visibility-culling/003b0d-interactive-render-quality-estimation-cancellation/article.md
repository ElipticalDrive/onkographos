# Interactive Render Quality Estimation & Cancellation

Also called: Interactive quality fallback, Octree render cancellation

## Overview
This method prevents infinite rendering loops in interactive volume rendering by detecting when traversal fails to converge and forcing a lower-quality fallback render. It sets apart from siblings like [Early Termination and Probabilistic Path Control](1791-early-termination-and-probabilistic-path-control.md) by acting as a safety guard for the traversal loop itself rather than optimizing the ray casting process, and from [Visibility Culling](6226-visibility-culling.md) by reacting to runtime failure states rather than precomputing visibility. It solves the problem of non-terminating or excessively long interactive sessions when the renderer cannot determine sufficient detail.

## How it works
The method operates during the frame rendering loop, specifically after the octree traversal phase but before final compositing. It relies on a flag set by the traversal engine and a predefined quality threshold.

**Inputs and Outputs:**
*   **Input:** The current rendering state, including the `_renderingRequired` structure containing the `OCTREE_RENDERING_CANCELED` flag.
*   **Output:** A modified render quality parameter (`OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD`) and an early return from the current rendering pass to prevent a loop.

**Data Structures:**
*   `_renderingRequired`: A boolean or enum structure.
    *   `OCTREE_RENDERING_CANCELED`: A flag indicating that the standard high-quality traversal has failed or been aborted.

**Parameters:**
*   `OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD`: A float or integer threshold defining the maximum acceptable detail level (e.g., maximum point size or minimum voxel resolution) for the fallback interactive render. Typical values are not explicitly given in the sources, but it must be low enough to guarantee termination. [S1, unofficial]

**Steps:**
1.  **Check Cancellation Flag:** After the primary octree traversal completes (or is interrupted), inspect the `_renderingRequired` structure.
2.  **Evaluate Condition:** If `OCTREE_RENDERING_CANCELED` is set to true, proceed to step 3. Otherwise, continue with standard high-quality rendering. [S1, unofficial]
3.  **Estimate Lower Quality:** Calculate a safe, lower render quality using the `OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD`. This typically involves increasing the point size or decreasing the sampling density to ensure the next frame renders quickly. [S1, unofficial]
4.  **Unlock API/Release Resources:** Ensure any locked rendering contexts or API handles are released to allow the fallback render to proceed without deadlock. [S1, unofficial]
5.  **Render Fallback:** Execute the rendering pass with the reduced quality settings. [S1, unofficial]
6.  **Return Early:** Exit the current high-quality rendering loop to avoid re-entering the failed state. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_frame(rendering_state):
    # 1. Perform primary octree traversal
    traversal_result = perform_octree_traversal(rendering_state)
    
    # 2. Check if traversal was canceled or failed
    if rendering_state._rendering_required.OCTREE_RENDERING_CANCELED:
        # 3. Estimate lower interactive render quality
        # Use the predefined threshold to determine safe parameters
        fallback_quality = estimate_quality(
            threshold=rendering_state.OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD
        )
        
        # 4. Unlock API resources to prevent deadlock
        unlock_rendering_api()
        
        # 5. Render with lower quality
        render_with_quality(fallback_quality)
        
        # 6. Return early to avoid infinite loop
        return

    # Standard high-quality rendering path
    render_high_quality(traversal_result)
```

**Edge Cases and Pitfalls:**
*   **Infinite Loop:** Overestimating the render quality in the fallback step can cause the renderer to enter an infinite loop if the high-quality traversal continues to fail. The cancellation check and forced quality reduction prevent this by guaranteeing a faster, albeit lower-quality, frame. [S1, unofficial]
*   **Precision:** The threshold must be set conservatively to ensure the fallback render completes within the interactive frame budget.

## Subtopics

### Visibility Coefficient Computation
A preprocessing step that computes view-dependent visibility coefficients for each octree leaf to estimate the likelihood of a node blocking light, used by the PLP algorithm at runtime. It differs from siblings by using rasterization-based screen coverage rather than primitive counts. It is chosen when fast, accurate visibility estimation is needed for prioritized rendering. [S2, unofficial]
*   **Definition:** Precompute a set of view-dependent values based on the screen coverage of primitives in a node relative to the screen coverage of the node's bounding box. [S2, unofficial]
*   **Steps:**
    1.  For each octree leaf, place an arbitrary number of sample viewpoints around the leaf. [S2, unofficial]
    2.  Pick viewpoints such that the entire node is visible from the viewpoint towards the center, maximizing the projected screen area of the node's bounding box. [S2, unofficial]
    3.  For each sample viewpoint:
        a. Rasterize the node's bounding box in green. [S2, unofficial]
        b. Rasterize the node's contents in red over a black background, without depth tests. [S2, unofficial]
        c. Read back the frame buffer. [S2, unofficial]
        d. Count green pixels ($n_g$) and red pixels ($n_r$). [S2, unofficial]
        e. Compute coefficient as $n_r / (n_r + n_g)$. [S2, unofficial]
    4.  At runtime, pick the coefficient whose corresponding sampling direction is closest to the current viewing direction. [S2, unofficial]
*   **Parameters:** `num_coefficients`: The number of visibility coefficients stored per node. Typical value is 20. [S2, unofficial]

## Operating characteristics
*   **Time Complexity:** The cancellation check is $O(1)$. The quality estimation is $O(1)$ if the threshold is a constant. The fallback render time depends on the reduced quality settings but is guaranteed to be faster than the failed high-quality attempt.
*   **Memory Complexity:** $O(1)$ additional memory for the flag and threshold.
*   **Strengths:** Prevents infinite loops and hangs in interactive applications. Simple to implement.
*   **Limitations:** Does not improve the quality of the final image, only ensures progress. Relies on the traversal engine correctly setting the cancellation flag.
*   **When to use:** In interactive volume rendering where user experience requires continuous frame updates, even if at reduced quality.
*   **When not to use:** In batch rendering where quality is paramount and timeouts are handled differently.

## Background
The method originates from interactive visualization systems for large datasets, specifically those using octree structures for out-of-core rendering. The concept of checking for rendering cancellation and falling back to lower quality is documented in the iVoxel project. The underlying visibility estimation techniques (visibility coefficients) are derived from research by Wagner Correa and Carlos Silva, building on earlier work by Klosowski and Silva. [S1, S2, unofficial]

## Sources
[S1] larsxschneider/iVoxel, https://github.com/larsxschneider/iVoxel, fetched 2026-10-27 (unofficial)
[S2] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
