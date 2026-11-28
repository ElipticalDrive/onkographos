# Selective Refinement with Active Front

Also called: Active Front Representation, Current Mesh $\Sigma$

## Overview
Selective Refinement with Active Front is a method for maintaining a conforming tetrahedral mesh $\Sigma$ (the "current mesh") during the incremental extraction of a partial multiresolution volume dataset [S1]. It solves the problem of efficiently traversing a diamond hierarchy to select diamonds based on error thresholds and isovalue criteria without reconstructing the mesh from scratch [S1]. It sets itself apart from static mesh representations by maintaining a dynamic "cut" of the refinement DAG that separates applied modifications from those not yet applied, allowing for incremental updates [S1].

## How it works
**Inputs:** A Diamond Multiresolution Volume (DMV) hierarchy; an approximation error threshold $\varepsilon$; a target isovalue $\kappa$ [S1].
**Outputs:** A conforming tetrahedral mesh $\Sigma$ representing the current state of selective refinement [S1].

**Data Structures:**
The active front can be represented in two primary ways, both using hash tables to map spatial locations to metadata [S1].

1.  **Diamond-based Active Front ($\Sigma_d$):**
    *   **Index:** Hash table keyed by the central vertex $v_c$ of each diamond [S1].
    *   **Entry Layout:**
        *   Coordinates of $v_c$: 6 bytes [S1].
        *   Bookkeeping: 1 byte (8 bits) containing bitflags [S1].
    *   **Bitflag Allocation:** The bitflags track which parents of a diamond have been subdivided. The number of bits required depends on the diamond type (defined by the number of parents $k$):
        *   0-diamonds ($k=3$ parents): 3 bits [S1].
        *   1-diamonds ($k=2$ parents): 2 bits [S1].
        *   2-diamonds ($k=4$ parents): 4 bits [S1].
    *   **Total Overhead:** 7 bytes per diamond [S1].

2.  **Supercube-based Active Front ($\Sigma_s$):**
    *   **Index:** Separate hash table for each level of supercubes, keyed by the supercube origin [S1].
    *   **Entry Layout:** Metadata associated with the supercube [S1].
    *   **Total Overhead:** 27 bytes per supercube [S1].

**Algorithm Steps:**
1.  **Initialization:** Start with the base mesh or an existing partial mesh $\Sigma$ [S1].
2.  **Selection:** Traverse the DMV hierarchy to identify diamonds $\delta$ that satisfy the selection criteria: approximation error $> \varepsilon$ and containing the isovalue $\kappa$ [S1].
3.  **Validation (Parent Constraint):** Before subdividing a diamond $\delta$, verify that all of its parents have already been subdivided. This ensures that all tetrahedra belonging to $\delta$ are present in the current mesh $\Sigma$. A diamond cannot be subdivided until this condition is met [S1].
4.  **Refinement:** Subdivide the selected diamonds. Update the active front representation ($\Sigma_d$ or $\Sigma_s$) to reflect the new mesh state. In $\Sigma_d$, this involves adding/removing diamonds from the hash table, which incurs memory allocation/deallocation [S1]. In $\Sigma_s$, clustering reduces the frequency of these allocations/deallocations [S1].
5.  **Iteration:** Repeat selection and refinement until no diamonds meet the criteria or a target mesh complexity is reached [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def selective_refinement(dm_hierarchy, epsilon, kappa, active_front_type='diamond'):
    """
    Maintains the active front Σ during selective refinement.
    dm_hierarchy: Iterable of diamond objects. Each has: id, vc, parents (list), error, contains_isovalue(kappa), num_parents.
    epsilon: Approximation error threshold.
    kappa: Target isovalue.
    active_front_type: 'diamond' for Σd, 'supercube' for Σs.
    """
    # Hash table for active front: key = vc (or supercube origin), value = {bitflags, subdivided_parents_count}
    active_front = {}
    current_mesh_tets = []

    # Selection phase
    candidates = [d for d in dm_hierarchy if d.contains_isovalue(kappa) and d.error > epsilon]

    # Process candidates
    for diamond in candidates:
        vc = diamond.vc
        if vc not in active_front:
            active_front[vc] = {'bitflags': 0, 'subdivided_parents': 0}

        state = active_front[vc]
        # Validation: Check parent constraint
        if state['subdivided_parents'] < diamond.num_parents:
            continue

        # Refinement: Update bitflags to track subdivided parents and tetrahedra presence
        for _ in range(diamond.num_parents):
            state['bitflags'] |= (1 << state['subdivided_parents'])
            state['subdivided_parents'] += 1

        # Extract tetrahedra present in Σ based on bitflags
        # A diamond with k parents has 2^k tetrahedra
        for i in range(2**diamond.num_parents):
            if state['bitflags'] & (1 << i):
                current_mesh_tets.append(f"tet_{diamond.id}_{i}")

        # Remove from active front after processing
        del active_front[vc]

    return current_mesh_tets
```

**Edge Cases and Pitfalls:**
*   **Memory Allocation:** Adding or removing a diamond from $\Sigma_d$ requires dynamic memory allocation and deallocation, which can be costly [S1].
*   **Hash Table Load Factor:** Memory overhead is inversely proportional to the hash table load factor [S1].
*   **Conforming Mesh Constraint:** The method enforces a conforming mesh by ensuring a diamond is only subdivided after all its parents are subdivided, guaranteeing all its tetrahedra are present in $\Sigma$ [S1].

## Operating characteristics
**Time Complexity:**
Performance is measured by the number of diamonds visited per second. All three DMV representations ($\Delta_f, \Delta_d, \Delta_s$) yield similar performance, processing approximately 300,000 diamonds per second on the tested hardware [S1].
*   $\Delta_f$ (array-based) is the fastest, allowing direct access via array indices [S1].
*   $\Delta_d$ (hash-based diamond) is 5-6% slower than $\Delta_f$ due to indirect hashing [S1].
*   $\Delta_s$ (hash-based supercube) is ~7.5% slower than $\Delta_f$ but within 2% of $\Delta_d$ [S1].
*   The supercube-based active front $\Sigma_s$ is on average 1-2% more efficient than the diamond-based $\Sigma_d$ [S1].

**Memory Complexity:**
*   **$\Sigma_d$:** Requires 7 bytes overhead per diamond. Across tested datasets, the average load factor was 74% (std dev 15%), resulting in a 36% memory overhead compared to the ideal representation [S1]. It averages 3.3 tetrahedra per diamond, leading to ~2.16 bytes overhead per tetrahedron [S1].
*   **$\Sigma_s$:** Requires 27 bytes overhead per supercube. The average load factor was 72% (std dev 11%), resulting in a 39% memory overhead [S1]. It averages 26.5 tetrahedra per supercube, leading to ~1 byte overhead per tetrahedron [S1].
*   **Comparison:** $\Sigma_s$ uses less than half the storage of $\Sigma_d$ for equivalent meshes due to reduced overhead per tetrahedron [S1].

**Strengths and Limitations:**
*   **Strengths:** $\Sigma_s$ offers significant storage savings and slightly better performance than $\Sigma_d$, with fewer memory allocations due to supercube clustering [S1].
*   **Limitations:** $\Sigma_d$ incurs frequent memory allocations/deallocations [S1]. $\Sigma_d$ overhead is inversely proportional to load factor, while $\Sigma_s$ overhead depends on supercube clustering concentration [S1].

## Background
Selective refinement with an active front was introduced in the context of Diamond Multiresolution Volumes (DMV) and Supercubes. The method is described in the paper "Supercubes: A High-Level Primitive for Diamond Hierarchies" [S1]. It builds on the concept of maintaining a "cut" in the refinement DAG to represent the current mesh state during isosurface extraction or direct volume rendering [S1].

## Sources
[S1] Supercubes: A High-Level Primitive for Diamond Hierarchies, https://doi.org/10.1109/tvcg.2009.186, fetched 2026-10-28
