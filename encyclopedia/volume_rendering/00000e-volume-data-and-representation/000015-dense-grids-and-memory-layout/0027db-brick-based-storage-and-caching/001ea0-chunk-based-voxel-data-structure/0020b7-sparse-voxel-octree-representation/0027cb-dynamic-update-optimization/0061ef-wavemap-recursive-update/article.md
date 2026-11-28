# Wavemap Recursive Update

Also called: RecursiveAdaptiveUpdate

## Overview
The Wavemap Recursive Update is an algorithm for updating a volumetric map posterior given a current measurement, utilizing an octree structure with adaptive resolution and early termination based on approximation error [S1, S2]. It distinguishes itself from sibling methods by recursively traversing the octree and pruning branches based on sensor likelihood and approximation bounds, rather than relying on priority queues or static memory management [S1, S2].

## How it works
**Inputs:**
*   Current measurement $z_t$ [S1].
*   Previous map posterior $p(m | z_{1:t-1})$ [S1].
*   Lower log-odds threshold $L_{min}$ [S1].
*   Approximation error threshold $\epsilon_{thresh}$ [S1].
*   Maximum resolution $res_{max}$ [S1].
*   Octree’s root partition $V_{root}$ [S1].

**Outputs:**
*   Updated map posterior $p(m | z_{1:t})$ [S1].

**Data Structures:**
The method operates on an octree where each node $V$ represents a voxel partition [S1, S2]. Each node stores:
*   Log-odds occupancy value $L_p(m_V | z_{1:t-1})$ [S1].
*   Current resolution $V_{res}$ [S1].
*   Children pointers for non-leaf nodes [S1].

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
*   $L_{min}$: Lower log-odds threshold used to skip updates for unobserved or free cells with low confidence [S2].
*   $\epsilon_{thresh}$: Approximation error threshold used to terminate recursion early if the error is below this value [S2].
*   $res_{max}$: Maximum resolution of the octree; recursion stops if the current voxel resolution equals this value [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def UpdateType(V, z_t):
    # Determines if measurement z_t intersects partition V.
    # Returns "FullyUnobserved", "FreeOrUnobserved", or "Occupied" [G].
    if not intersects(V, z_t):
        return "FullyUnobserved"
    if is_free_space(z_t):
        return "FreeOrUnobserved"
    return "Occupied"

def ApproximationError(V, z_t):
    # Computes the maximum approximation error for voxel V given measurement z_t [G].
    return compute_error(V, z_t)

def SensorLikelihood(V, z_t):
    # Computes the sensor likelihood L_s(m_V | z_t) for voxel V [G].
    return compute_likelihood(V, z_t)

def RecursiveAdaptiveUpdate(V, z_t, L_min, epsilon_thresh, res_max):
    update_type = UpdateType(V, z_t)
    
    if update_type == "FullyUnobserved":
        return

    if update_type == "FreeOrUnobserved" and V.log_odds <= L_min:
        return

    epsilon_max = ApproximationError(V, z_t)

    if V.resolution == res_max or epsilon_max < epsilon_thresh:
        V.log_odds += SensorLikelihood(V, z_t)
        return

    for V_child in V.children:
        RecursiveAdaptiveUpdate(V_child, z_t, L_min, epsilon_thresh, res_max)

def UpdateMap(z_t, p_prev, L_min, epsilon_thresh, res_max, V_root):
    p_current = p_prev
    RecursiveAdaptiveUpdate(V_root, z_t, L_min, epsilon_thresh, res_max)
    return p_current
```

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### Priority Queue Based Field Update
Uses a priority queue to process updates based on importance, differing from the recursive error-bound approach. Choose when update order is critical for convergence speed. [7430-priority-queue-based-field-update.md]

### Sparse Voxel Octree Memory Management and Texture Upload
Focuses on memory allocation and GPU upload, whereas Wavemap Recursive Update focuses on the logical update traversal. Choose this for optimizing data transfer rather than update logic. [11147-sparse-voxel-octree-memory-management-and-texture-upload.md]

### Coarse-to-Fine Octree Update Strategy
Updates the octree starting from the root and refining nodes progressively, differing from the adaptive error-bound recursion. Choose when hierarchical refinement order is preferred. [15715-coarse-to-fine-octree-update-strategy.md]

## Operating characteristics
*   **Time Complexity:** Depends on the octree depth and the distribution of sensor data [S1]. Early termination based on $\epsilon_{max}$ and $L_{min}$ reduces the number of visited nodes compared to a full traversal [S1].
*   **Memory Complexity:** Proportional to the number of active octree nodes [S1].
*   **Measured Performance:** The method is evaluated in different range bands, including small negative values (thin objects), distances close to zero (surface reconstruction), and larger distances (obstacles) [S1]. Performance is quantified using the Area under the ROC Curve (AUC) to assess classification performance, avoiding dilution of challenging situations with easy free-space observations [S1].
*   **Strengths:** Adaptive resolution allows for high detail where needed and coarse representation elsewhere [S1]. Early termination improves efficiency [S1].
*   **Limitations:** Requires accurate calculation of approximation error and update type [S1].
*   **When to use:** Suitable for volumetric mapping in multi-scale environments where adaptive resolution and efficient updates are required [S1].
*   **When not to use:** Not suitable for static maps where frequent updates are unnecessary [G].

## Background
The algorithm is presented as Algorithm 1 in the source literature [S1, S2]. It is designed for efficient volumetric mapping using an octree structure with adaptive resolution and early termination based on approximation error [S1, S2].

## Sources
[S1] Efficient volumetric mapping of multi-scale environments ..., https://www.roboticsproceedings.org/rss19/p065.pdf, fetched 2026-10-26
[S2] Efficient volumetric mapping of multi-scale environments using wavelet-based compression, https://zenodo.org/records/7993757, fetched 2026-10-27
