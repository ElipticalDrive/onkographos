# Build Setup & Indirect Dispatch Management

Also called: Build Setup Compute, Indirect Dispatch Setup

## Overview
This method finalizes the parallel volume building process by synchronizing atomic counters and configuring indirect dispatch or draw parameters for subsequent rendering passes [S1, unofficial]. It acts as a single-thread compute shader that clamps counters to their allocated limits to prevent out-of-bounds execution, bridging the gap between parallel work generation and sequential GPU submission [S1, unofficial]. It differs from [Indirect Dispatch Orchestration](15194-indirect-dispatch-orchestration.md) by focusing specifically on the data preparation and boundary clamping logic rather than the broader scheduling strategy.

## How it works
**Inputs and Outputs:**
*   **Input:** `push.setup` (enum controlling operation type), `build.maxRenderClusters` (limit), `BLAS_INSERT_CLUSTERS_WORKGROUP` (constant).
*   **Input/Output:** `SceneBuilding buildRW` (coherent SSBO), `Readback` (SSBO).
*   **Output:** Configured indirect parameters in `buildRW` and readback values in `Readback`.

**Data Structures:**
1.  `SceneBuilding buildRW` (Coherent SSBO):
    *   `traversalTaskCounter`: `uint`
    *   `traversalInfoWriteCounter`: `uint`
    *   `traversalGroupCounter`: `uint`
    *   `renderClusterCounter`: `uint`
    *   `indirectDrawClusters`: Struct `{ count: uint, first: uint }`
    *   `indirectDispatchBlasInsertion`: Struct `{ gridX: uint, gridY: uint, gridZ: uint }`
    *   `maxTraversalInfos`: `uint`
    *   `maxRenderClusters`: `uint`
2.  `Readback` (SSBO):
    *   `numRenderClusters`: `uint`
    *   `numTraversalInfos`: `uint`
    *   `numRenderedClusters`: `uint`

**Steps:**
The method executes as a single-thread compute shader. The logic branches based on `push.setup`.

1.  **Clamp and Configure Draw Parameters:**
    If `push.setup == BUILD_SETUP_DRAW`:
    *   Calculate `numRenderedClusters = min(renderClusterCounter, build.maxRenderClusters)`.
    *   Set `buildRW.indirectDrawClusters.count = numRenderedClusters`.
    *   Set `buildRW.indirectDrawClusters.first = 0`.

2.  **Record Traversal Info for Readback:**
    *   Determine the effective traversal count. If `USE_SEPARATE_GROUPS` is false, use `buildRW.traversalInfoWriteCounter`. Otherwise, use `buildRW.traversalGroupCounter`.
    *   Calculate `effectiveTraversalCount = max(buildRW.traversalInfoWriteCounter, effectiveGroupCounter)`.
    *   Set `Readback.numTraversalInfos = effectiveTraversalCount`.

3.  **Clamp and Configure BLAS Insertion Dispatch:**
    If `push.setup == BUILD_SETUP_BLAS_INSERTION`:
    *   Use the clamped `numRenderedClusters` from Step 1 (or re-calculate if not previously set).
    *   Calculate grid X dimension: `gridX = (numRenderedClusters + BLAS_INSERT_CLUSTERS_WORKGROUP - 1) / BLAS_INSERT_CLUSTERS_WORKGROUP`.
    *   Set `buildRW.indirectDispatchBlasInsertion.gridX = gridX`.
    *   Set `gridY` and `gridZ` to 1 (standard 1D dispatch assumption unless specified otherwise).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_setup_compute(push_setup, build, readback, BLAS_INSERT_CLUSTERS_WORKGROUP, USE_SEPARATE_GROUPS):
    # Step 1: Handle Draw Setup
    if push_setup == BUILD_SETUP_DRAW:
        # Clamp atomic counter to allocated limit to prevent OOB
        num_rendered_clusters = min(build.renderClusterCounter, build.maxRenderClusters)
        build.indirectDrawClusters.count = num_rendered_clusters
        build.indirectDrawClusters.first = 0

    # Step 2: Record Traversal Info
    # Determine which counter represents the active group count
    if not USE_SEPARATE_GROUPS:
        group_counter = build.traversalInfoWriteCounter
    else:
        group_counter = build.traversalGroupCounter
    
    # Take max to ensure all info is captured
    effective_traversal_count = max(build.traversalInfoWriteCounter, group_counter)
    readback.numTraversalInfos = effective_traversal_count

    # Step 3: Handle BLAS Insertion Setup
    if push_setup == BUILD_SETUP_BLAS_INSERTION:
        # Re-calculate or use existing clamped count
        # Note: If BUILD_SETUP_DRAW was not run, renderClusterCounter might be unclamped.
        # Safe practice: clamp again here if count is needed for dispatch sizing.
        current_cluster_count = build.renderClusterCounter
        if push_setup != BUILD_SETUP_DRAW:
            current_cluster_count = min(current_cluster_count, build.maxRenderClusters)
        
        # Ceiling division for grid size
        grid_x = (current_cluster_count + BLAS_INSERT_CLUSTERS_WORKGROUP - 1) // BLAS_INSERT_CLUSTERS_WORKGROUP
        build.indirectDispatchBlasInsertion.gridX = grid_x
        build.indirectDispatchBlasInsertion.gridY = 1
        build.indirectDispatchBlasInsertion.gridZ = 1

    # Optional: Update readback for clusters if not done in draw setup
    if push_setup == BUILD_SETUP_DRAW:
        readback.numRenderedClusters = build.indirectDrawClusters.count
        readback.numRenderClusters = build.renderClusterCounter # Original unclamped or total generated
```

**Parameters:**
*   `push.setup`: Enum. Values: `BUILD_SETUP_TRAVERSAL_RUN`, `BUILD_SETUP_DRAW`, `BUILD_SETUP_BLAS_INSERTION` [S1, unofficial].
*   `BLAS_INSERT_CLUSTERS_WORKGROUP`: Constant defining workgroup size for BLAS insertion dispatch. Typical value: Not given in the sources.
*   `USE_SEPARATE_GROUPS`: Boolean flag. If false, `traversalInfoWriteCounter` is used for readback logic [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Counter Overshoot:** Traversal and cluster counters are atomic and may overshoot their allocated limits during concurrent execution. The `min()` clamp is critical to prevent out-of-bounds indirect dispatches [S1, unofficial].
*   **Coherence:** `SceneBuilding` must be a coherent SSBO to ensure visibility of atomic updates across different shader stages or invocations [S1, unofficial].
*   **Branching:** The method is single-threaded but handles multiple setup types. Ensure the caller invokes the correct setup type corresponding to the required subsequent pass.

## Operating characteristics
*   **Time Complexity:** $O(1)$ relative to scene size, as it performs a fixed number of atomic reads, comparisons, and writes.
*   **Memory Complexity:** $O(1)$ auxiliary memory, relying on existing SSBOs.
*   **Performance:** Negligible overhead compared to the parallel build phases. The single-thread nature ensures no contention.
*   **Strengths:** Prevents GPU hangs or crashes due to invalid dispatch sizes; centralizes parameter configuration.
*   **Limitations:** Single-threaded bottleneck if the setup logic were more complex; relies on correct atomic counter management in prior stages.
*   **When to use:** After any parallel volume building or traversal phase that produces atomic counters requiring clamping before submission.

## Background
This method is part of the Proxy-GS rendering pipeline, specifically handling the transition from parallel compute to indirect drawing/dispatch [S1, unofficial]. It addresses the common GPU programming challenge of synchronizing parallel atomic counters with sequential submission requirements.

## Sources
[S1] Visionary-Laboratory/Proxy-GS, https://github.com/Visionary-Laboratory/Proxy-GS, fetched 2026-10-27 (unofficial)
