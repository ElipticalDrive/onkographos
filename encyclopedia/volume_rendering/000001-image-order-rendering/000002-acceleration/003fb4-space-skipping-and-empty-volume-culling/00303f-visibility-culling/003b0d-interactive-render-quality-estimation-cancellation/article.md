# Interactive Render Quality Estimation & Cancellation

Also called: Interactive quality fallback, Octree render cancellation

## Overview
This method prevents infinite rendering loops in interactive volume visualization by detecting when the rendering pipeline cannot keep up with the frame budget and forcing a quality reduction on the subsequent frame [S1, unofficial]. It acts as a safety mechanism within the traversal loop, distinct from standard early termination which stops a single ray based on opacity [1791-early-termination-and-probabilistic-path-control.md]. Unlike [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md) which culls based on precomputed depth, this method reacts to runtime performance failures by lowering the sampling density or resolution [S1, unofficial].

## How it works
The method integrates with an octree-based traversal loop. It relies on a flag indicating that the previous frame's rendering was not completed in time, triggering a fallback to a lower-quality estimation.

**Inputs:**
- Current frame status flag: `OCTREE_RENDERING_CANCELED` (boolean).
- Quality threshold parameter: `OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD` (float).
- Current octree structure and camera state.

**Outputs:**
- A reduced set of samples or a lower-resolution image for the current frame.
- A signal to skip the full rendering loop.

**Data Structures:**
- `_renderingRequired`: A structure or flag container holding the `OCTREE_RENDERING_CANCELED` state.
- `OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD`: A scalar value defining the minimum acceptable quality level (e.g., minimum point size or maximum step size) when cancellation occurs.

**Steps:**
1. **Check Cancellation Flag:** At the start of the frame or after octree traversal, inspect `_renderingRequired` for the `OCTREE_RENDERING_CANCELED` flag [S1, unofficial].
2. **Evaluate Condition:** If the flag is set, it indicates that the previous frame exceeded the time budget, risking an infinite loop if the same high-quality parameters are reused [S1, unofficial].
3. **Estimate Lower Quality:** Calculate a safe, lower render quality using `OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD` [S1, unofficial]. This typically involves increasing the step size (coarser sampling) or reducing the number of samples per ray.
4. **Unlock and Return:** Unlock the rendering API/resources and return early from the current rendering loop, skipping the expensive full-resolution traversal [S1, unofficial].
5. **Next Frame:** The next frame will render with the reduced quality parameters, ensuring the frame completes within the budget and the flag is cleared.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_frame(octree, camera, rendering_state):
    # Check if previous frame was canceled due to time budget overflow
    if rendering_state._rendering_required.OCTREE_RENDERING_CANCELED:
        # Estimate a lower interactive render quality based on the threshold
        # This forces a coarser sampling or lower resolution to guarantee completion
        quality = estimate_interactive_quality(
            threshold=rendering_state.OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD
        )
        
        # Unlock API/resources to prevent deadlock
        rendering_state.unlock_api()
        
        # Return early to avoid the expensive rendering loop
        return render_low_quality(octree, camera, quality)
    
    # Normal high-quality rendering path
    try:
        result = full_octree_traversal(octree, camera)
        rendering_state._rendering_required.OCTREE_RENDERING_CANCELED = False
        return result
    except TimeLimitExceeded:
        # Set flag for next frame to trigger cancellation logic
        rendering_state._rendering_required.OCTREE_RENDERING_CANCELED = True
        raise
```

**Parameters:**
- `OCTREE_INTERACTIVE_RENDERING_POINT_THRESHOLD`: Threshold used to estimate a safe render quality when interactive traversal is canceled. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Infinite Loop:** Overestimating render quality when the cancellation flag is set can cause an infinite rendering loop, as the system continues to request more work than the hardware can deliver [S1, unofficial]. The cancellation check and forced quality estimation prevent this by guaranteeing a lower computational load on the next frame [S1, unofficial].

## Subtopics

### Visibility Coefficient Computation
A preprocessing step to compute view-dependent visibility coefficients for each octree leaf, used by the Prioritized-Layered Projection (PLP) algorithm to prioritize visible nodes [S2, unofficial]. It estimates the likelihood of a node blocking light by rasterizing the node's contents against its bounding box from multiple viewpoints [S2, unofficial].
- **Steps:**
    1. For each octree leaf, place an arbitrary number of sample viewpoints around the leaf [S2, unofficial].
    2. Pick viewpoints such that the entire node is visible from the viewpoint towards the center, maximizing the projected screen area of the node's bounding box [S2, unofficial].
    3. For each sample viewpoint:
        a. Rasterize the node's bounding box in green [S2, unofficial].
        b. Rasterize the node's contents in red over a black background, without depth tests [S2, unofficial].
        c. Read back the frame buffer [S2, unofficial].
        d. Count green pixels ($n_g$) and red pixels ($n_r$) [S2, unofficial].
        e. Compute coefficient as $n_r / (n_r + n_g)$ [S2, unofficial].
    4. At runtime, pick the coefficient whose corresponding sampling direction is closest to the current viewing direction [S2, unofficial].
- **Parameters:**
    - `num_coefficients`: The number of visibility coefficients stored per node. Typical value is 20 [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The cancellation check is $O(1)$. The quality estimation is $O(1)$ if the threshold maps directly to a parameter. The actual rendering cost is reduced significantly compared to full traversal.
- **Memory Complexity:** Minimal additional memory for the flag and threshold. The visibility coefficients (if used in conjunction) add $O(N \cdot K)$ memory where $N$ is the number of leaves and $K$ is the number of coefficients [S2, unofficial].
- **Performance:** Guarantees frame completion by degrading quality, preventing hangs [S1, unofficial].
- **Strengths:** Robust against infinite loops; simple to implement.
- **Limitations:** Visual quality may fluctuate between frames; does not optimize the *current* frame's quality, only ensures *completion*.

## Background
The method originates from interactive visualization systems for large datasets, specifically those using octree structures for out-of-core rendering [S2, unofficial]. It is implemented in systems like iVoxel to handle the latency of interactive traversal [S1, unofficial]. The visibility coefficient variant is described in Wagner Correa's thesis on out-of-core visualization techniques [S2, unofficial].

## Sources
[S1] larsxschneider/iVoxel, https://github.com/larsxschneider/iVoxel, fetched 2026-10-27 (unofficial)
[S2] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
