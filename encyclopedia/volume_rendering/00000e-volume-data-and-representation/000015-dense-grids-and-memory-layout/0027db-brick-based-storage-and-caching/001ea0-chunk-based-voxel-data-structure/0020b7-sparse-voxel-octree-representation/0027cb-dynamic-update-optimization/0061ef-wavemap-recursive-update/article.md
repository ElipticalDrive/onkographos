# Wavemap Recursive Update

Also called: RecursiveAdaptiveUpdate

## Overview
The Wavemap Recursive Update is an algorithm for updating a volumetric map posterior given a current measurement, utilizing an octree structure with adaptive resolution and early termination based on approximation error [S1]. It distinguishes itself from sibling methods by combining wavelet-based compression with a recursive traversal that prunes branches based on sensor likelihood and approximation bounds, rather than relying solely on priority queues or static memory management [S1].

## How it works
**Inputs:**
*   Current measurement $z_t$
*   Previous map posterior $p(m | z_{1:t-1})$
*   Lower log-odds threshold $L_{min}$
*   Approximation error threshold $\epsilon_{thresh}$
*   Maximum resolution $res_{max}$
*   Octree’s root partition $V_{root}$

**Outputs:**
*   Updated map posterior $p(m | z_{1:t})$

**Data Structures:**
The method operates on an octree where each node $V$ represents a voxel partition. Each node stores:
*   Log-odds occupancy value $L_p(m_V | z_{1:t-1})$.
*   Current resolution $V_{res}$.
*   Children pointers (for non-leaf nodes).

**Steps:**
1.  Initialize the map posterior: $p(m | z_{1:t}) \leftarrow p(m | z_{1:t-1})$ [S1].
2.  Call `RecursiveAdaptiveUpdate(V_root, z_t)` to start the recursion from the root partition [S1].
3.  Inside `RecursiveAdaptiveUpdate(V, z_t)`:
    a. Determine the update type using Equation 11: `update_type` $\leftarrow$ `UpdateType(V, z_t)` [S1].
    b. If `update_type` is `FullyUnobserved`, return immediately (skip partition) [S1].
    c. If `update_type` is `FreeOrUnobserved` AND the previous log-odds $L_p(m_V | z_{1:t-1}) \le L_{min}$, return immediately [S1].
    d. Calculate the maximum approximation error for the voxel: $\epsilon_{max}(V) \leftarrow$ `ApproximationError(V, z_t)` [S1].
    e. If the voxel resolution $V_{res}$ equals $res_{max}$ OR $\epsilon_{max}(V) < \epsilon_{thresh}$, update the log-odds by adding the sensor likelihood $L_s(m_V | z_t)$ and return:
       $$L_p(m_V | z_{1:t}) \leftarrow L_p(m_V | z_{1:t-1}) + L_s(m_V | z_t)$$ [S1].
    f. Otherwise, iterate over all children $V_{child}$ of $V$ and recursively call `RecursiveAdaptiveUpdate(V_child, z_t)` [S1].

**Parameters:**
*   $L_{min}$: Lower log-odds threshold used to skip updates for unobserved or free cells with low confidence [S1].
*   $\epsilon_{thresh}$: Approximation error threshold used to terminate recursion early if the error is below this value [S1].
*   $res_{max}$: Maximum resolution of the octree; recursion stops if the current voxel resolution equals this value [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def RecursiveAdaptiveUpdate(V, z_t, L_min, epsilon_thresh, res_max):
    # Determine update type using Eq.11
    update_type = UpdateType(V, z_t)
    
    # Skip fully unobserved partitions
    if update_type == "FullyUnobserved":
        return

    # Skip free/unobserved cells with low confidence
    if (update_type == "FreeOrUnobserved" and 
        V.log_odds_prev <= L_min):
        return

    # Calculate max approximation error using Eq.13
    epsilon_max = ApproximationError(V, z_t)

    # Terminate early if at max resolution or error is low
    if (V.resolution == res_max or epsilon_max < epsilon_thresh):
        # Update log-odds with sensor likelihood
        V.log_odds_current = V.log_odds_prev + SensorLikelihood(V, z_t)
        return

    # Recurse into children if conditions not met
    for V_child in V.children:
        RecursiveAdaptiveUpdate(V_child, z_t, L_min, epsilon_thresh, res_max)

def UpdateMap(z_t, p_prev, L_min, epsilon_thresh, res_max, V_root):
    # Initialize map posterior
    p_current = p_prev
    
    # Start recursion from root
    RecursiveAdaptiveUpdate(V_root, z_t, L_min, epsilon_thresh, res_max)
    
    return p_current
```

**Edge Cases and Pitfalls:**
*   **Precision:** Log-odds arithmetic must handle small values carefully to avoid underflow when $L_p$ is near $L_{min}$ [S1].
*   **Boundaries:** The `FullyUnobserved` check must correctly identify partitions with no prior or current sensor data to prevent unnecessary computation [S1].
*   **Degenerate Input:** If $res_{max}$ is set too low, the approximation error $\epsilon_{max}$ may never drop below $\epsilon_{thresh}$, leading to excessive recursion or inaccurate maps [S1].

## Subtopics

### Priority Queue Based Field Update
Uses a priority queue to process updates based on importance, differing from the recursive error-bound approach. Choose when update order is critical for convergence speed. [7430-priority-queue-based-field-update.md]

### Sparse Voxel Octree Memory Management and Texture Upload
Focuses on memory allocation and GPU upload, whereas Wavemap Recursive Update focuses on the logical update traversal. Choose this for optimizing data transfer rather than update logic. [11147-sparse-voxel-octree-memory-management-and-texture-upload.md]

## Operating characteristics
*   **Time Complexity:** Depends on the octree depth and the distribution of sensor data. Early termination based on $\epsilon_{max}$ and $L_{min}$ reduces the number of visited nodes compared to a full traversal.
*   **Memory Complexity:** Proportional to the number of active octree nodes. Wavelet-based compression allows for efficient storage of the map posterior [S1].
*   **Measured Performance:** The method is evaluated in different range bands, including small negative values (thin objects), distances close to zero (surface reconstruction), and larger distances (obstacles) [S1]. Performance is quantified using the Area under the ROC Curve (AUC) to assess classification performance, avoiding dilution of challenging situations with easy free-space observations [S1].
*   **Strengths:** Adaptive resolution allows for high detail where needed and coarse representation elsewhere. Early termination improves efficiency.
*   **Limitations:** Requires accurate calculation of approximation error and update type. Performance may degrade in highly dynamic environments with frequent large-scale changes.
*   **When to use:** Suitable for volumetric mapping in multi-scale environments where adaptive resolution and efficient updates are required [S1].
*   **When not to use:** Not suitable for static maps where frequent updates are unnecessary, or when hardware lacks support for the required floating-point operations for log-odds and error calculations.

## Background
The Wavemap Recursive Update was introduced in the context of efficient volumetric mapping of multi-scale environments using wavelet-based compression [S1]. It builds upon octree structures and integrates wavelet compression to manage memory and update efficiency [S1]. The method is part of a broader effort to improve volumetric mapping performance, as evidenced by comparisons with other mapping systems like OctoMap, SuperEight, and Voxblox [S1].

## Sources
[S1] Efficient volumetric mapping of multi-scale environments using wavelet-based compression, https://zenodo.org/records/7993757, fetched 2026-10-27
