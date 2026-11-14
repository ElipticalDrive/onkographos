# Diverse View Selection via Overlap and Distance

Also called: Alg.1, Camera View Selection Algorithm, Iterative Diverse View Sampling

## Overview
This method selects a diverse subset of $N_v$ camera views from a 3D scene by iteratively sampling candidates and accepting them only if they satisfy minimum thresholds for spatial overlap and geometric distance relative to already selected views [S1]. It differs from simple random sampling or uniform grid sampling by explicitly enforcing diversity constraints to ensure broad coverage of the scene geometry, which is critical for training generalizable neural radiance fields [S1].

## How it works
**Inputs:** A pool of candidate camera poses (extrinsics $[R, t]$ and intrinsics); target number of views $N_v$; overlap threshold $T_o$; distance threshold $T_d$.
**Outputs:** A list $E$ of $N_v$ selected camera views.

**Data Structures:**
- $E$: A list of selected camera extrinsics $E_i = [R_i, t_i]$, where $R_i \in \mathbb{R}^{3 \times 3}$ is the rotation matrix and $t_i \in \mathbb{R}^3$ is the translation vector.
- $n$: Integer counter of selected views, initialized to 0.

**Formulas:**
1.  **Frustum Overlap ($O$):** The Intersection over Union (IoU) of the viewing frustums $A$ and $B$ of two cameras $E_1$ and $E_2$:
    $$ O(E_1, E_2) = \frac{|A \cap B|}{|A \cup B|} $$
    [S1]

2.  **Geometric Distance ($D$):** The sum of Euclidean distance between camera centers and the angular difference between orientations:
    $$ D(E_1, E_2) = \|t_1 - t_2\|_2 + \arccos\left(\frac{\text{Tr}(R_2^\top R_1) - 1}{2}\right) $$
    [S1]
    *Note:* The term $\arccos\left(\frac{\text{Tr}(R_2^\top R_1) - 1}{2}\right)$ computes the rotation angle in radians between the two coordinate frames.

3.  **Aggregate Metrics:** For a new candidate view $E$ against the set of existing views $E = \{E_1, \dots, E_n\}$:
    $$ \text{Overlap}(E, E) = \max_{i=1 \dots n} O(E, E_i) $$
    $$ \text{Distance}(E, E) = \min_{i=1 \dots n} D(E, E_i) $$
    [S1]

**Parameters:**
- $N_v = 200$: Target number of views [S1].
- $T_o = 0.3$: Minimum overlap threshold [S1].
- $T_d = 0.2$: Minimum distance threshold [S1].

**Algorithm Steps:**
1.  Initialize $E \leftarrow \emptyset$, $n \leftarrow 0$.
2.  Set $N_v = 200$, $T_o = 0.3$, $T_d = 0.2$ [S1].
3.  Repeat the following loop:
    a. Sample a candidate camera view $E_{cand}$ from the available pool [S1].
    b. If $n = 0$:
       i. Append $E_{cand}$ to $E$.
       ii. Increment $n \leftarrow n + 1$.
    c. Else ($n > 0$):
       i. Calculate $O_{max} = \text{Overlap}(E_{cand}, E)$.
       ii. Calculate $D_{min} = \text{Distance}(E_{cand}, E)$.
       iii. If $O_{max} \ge T_o$ AND $D_{min} \ge T_d$:
           - Append $E_{cand}$ to $E$.
           - Increment $n \leftarrow n + 1$.
    d. Stop when $n \ge N_v$ [S1].
4.  Output $E$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_views(candidate_pool, Nv=200, To=0.3, Td=0.2):
    E = []
    n = 0
    
    while n < Nv:
        # Sample a candidate view from the pool
        # [G] Implementation detail: candidate_pool should be shuffled or iterated
        E_cand = sample_candidate(candidate_pool)
        
        if n == 0:
            # First view is always accepted
            E.append(E_cand)
            n += 1
        else:
            # Calculate max overlap with any existing view
            max_overlap = 0.0
            for existing in E:
                overlap = compute_overlap(E_cand, existing)
                if overlap > max_overlap:
                    max_overlap = overlap
            
            # Calculate min distance with any existing view
            min_distance = float('inf')
            for existing in E:
                distance = compute_distance(E_cand, existing)
                if distance < min_distance:
                    min_distance = distance
            
            # Check thresholds
            if max_overlap >= To and min_distance >= Td:
                E.append(E_cand)
                n += 1
                
    return E

def compute_overlap(E1, E2):
    # [G] Frustum intersection volume calculation is non-trivial.
    # The source defines this as IoU of frustums A and B.
    # Exact geometric implementation depends on frustum definition (near/far planes).
    # Not given in the sources.
    pass

def compute_distance(E1, E2):
    t1 = E1.t
    t2 = E2.t
    R1 = E1.R
    R2 = E2.R
    
    # Euclidean distance between centers
    pos_dist = norm(t1 - t2)
    
    # Angular distance between orientations
    # Tr(R2^T R1) is the trace of the relative rotation matrix
    trace = trace(matrix_multiply(transpose(R2), R1))
    # Clamp value to [-1, 1] for numerical stability in arccos
    cos_angle = (trace - 1.0) / 2.0
    cos_angle = max(-1.0, min(1.0, cos_angle))
    angle_dist = arccos(cos_angle)
    
    return pos_dist + angle_dist
```

**Edge Cases and Pitfalls:**
- **Frustum Calculation:** The source defines overlap via frustum IoU but does not provide the geometric algorithm for computing the intersection volume of two arbitrary frustums [S1]. Implementers must define the frustum geometry (e.g., using near/far planes and FOV) and compute the voxelized or analytical intersection [G].
- **Degenerate Rotation:** If $R_1 = R_2$, the trace term becomes 3, leading to $\arccos(1) = 0$, which is correct. Numerical precision near $\pm 1$ should be handled via clamping [G].
- **Infinite Loop:** If the candidate pool is exhausted or too small to satisfy the diversity constraints, the loop may not terminate. The source assumes a sufficiently large pool [S1].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N_v \cdot K \cdot C)$, where $K$ is the number of candidates sampled per accepted view and $C$ is the cost of computing overlap/distance. The overlap calculation (frustum IoU) is computationally expensive compared to the distance calculation [S1].
- **Memory Complexity:** $O(N_v)$ to store the selected views.
- **Strengths:** Ensures spatial diversity, reducing redundancy in training data for generalizable NeRFs [S1].
- **Limitations:** The frustum overlap calculation is complex to implement efficiently. The method is heuristic and does not guarantee global optimality of coverage [S1].

## Background
The method was introduced in **ContraNeRF** (2023) by the authors of the ContraNeRF paper [S1]. It is used in the context of generalizable neural radiance fields for synthetic-to-real novel view synthesis, specifically to select input views for contrastive learning [S1]. The algorithm is labeled as "Algorithm 1" in the source [S1].

## Sources
[S1] ContraNeRF: Generalizable Neural Radiance Fields for Synthetic-to-real Novel View Synthesis via Contrastive Learning, https://doi.org/10.1109/cvpr52729.2023.01584, fetched 2026-10-29
