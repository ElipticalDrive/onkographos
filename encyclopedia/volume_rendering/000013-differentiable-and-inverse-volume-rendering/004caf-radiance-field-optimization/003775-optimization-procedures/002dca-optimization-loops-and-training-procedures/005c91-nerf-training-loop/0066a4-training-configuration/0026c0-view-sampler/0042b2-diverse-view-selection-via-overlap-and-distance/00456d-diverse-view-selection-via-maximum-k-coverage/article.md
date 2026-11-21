# Diverse View Selection via Maximum K-Coverage

Also called: Camera Selection Algorithm, Camera Selection via Maximum K-Coverage, Greedy K-Coverage View Selection, K-Maximum Coverage Camera Selection, Maximum K-Coverage camera selection

## Overview
This method selects a subset of input cameras to maximize pixel coverage in a novel view, addressing GPU memory limitations in large-scale Implicit Brain Representation (IBR) and neural rendering pipelines [S1]. It builds on [Diverse View Selection via Overlap and Distance](10788-diverse-view-selection-via-overlap-and-distance.md) by replacing heuristic overlap/distance metrics with a greedy approximation of the Maximum K-Coverage problem from set theory [S1]. Unlike simple ULR-style weights or voxel grid approximations, this approach provides fast and accurate selection by explicitly maximizing the union of visible pixels from the chosen subset [S1].

## How it works
**Inputs:**
*   $M$: Number of input cameras.
*   $N$: Number of pixels in the novel view space $P$.
*   $k$: Target number of views to select ($|V_{sel}| = k$).
*   $Mesh$: A proxy mesh estimated from Multi-View Stereo (MVS) [S1].
*   $T_i$: Projection matrices for each input camera $i \in \{1, \dots, M\}$.
*   $T_{novel}$: Projection matrix for the novel view.

**Outputs:**
*   $V_{sel}$: A subset of $k$ input camera indices.

**Data Structures:**
*   $S_i$: A downscaled score map (array) for input camera $i$, with dimensions matching the resolution of the novel view space.
*   $C_{covered}$: A boolean mask or integer counter array of size $N$, tracking the coverage status of each pixel in the novel view.

**Algorithm Steps:**

1.  **Initialize Coverage:** Create $C_{covered}$ initialized to zero (or false) for all $p \in P$. Initialize $V_{sel} = \emptyset$. Initialize total score $V_0 = 0$.

2.  **Precompute Score Maps:** For each input camera $i \in \{1, \dots, M\}$:
    a.  Project the $Mesh$ onto the novel view space using $T_{novel}$ and $T_i$ to determine visibility.
    b.  Perform a depth test using the $Mesh$ to handle occlusions [S1].
    c.  Compute the visibility score $S_i(p)$ for each pixel $p \in P$.
        *   *Binary Mode:* $S_i(p) = 1$ if $p$ is visible in view $i$, else $0$ [S1].
        *   *Density Mode (Optional):* Replace binary visibility with a per-pixel score approximating point density: $S_i(p) = \frac{d_{novel}(p)}{d_{input}(p)}$, where $d_{novel}(p)$ is the distance from the novel camera to the visible point, and $d_{input}(p)$ is the distance from the input camera to the visible point [S1].
    d.  Store $S_i$ in memory.

3.  **Greedy Selection Loop:** Iterate $j$ from $1$ to $k$:
    a.  **Evaluate Candidates:** For each remaining candidate camera $c \notin V_{sel}$:
        i.  Calculate the marginal gain $G_c = \sum_{p \in P} \max(0, S_c(p) - \mathbb{I}(C_{covered}[p] > 0))$.
            *   Here, $\mathbb{I}(\cdot)$ is the indicator function. Effectively, sum the scores of pixels in $S_c$ that are not yet covered by any camera in $V_{sel}$.
    b.  **Select Best:** Choose $c^* = \arg\max_{c \notin V_{sel}} G_c$.
    c.  **Update State:**
        i.  Add $c^*$ to $V_{sel}$.
        ii. Update $C_{covered}$: For all $p \in P$, if $S_{c^*}(p) > 0$, mark $p$ as covered (increment counter or set boolean).
        iii. Calculate current total score $V_j = \sum_{p \in P} \mathbb{I}(C_{covered}[p] > 0)$ (or sum of max scores if using non-binary scores).

4.  **Degeneracy Check:** After each selection step $j$, check for the degenerate case where the novel view is identical to an already selected input view.
    a.  Compute ratio $R = V_j / V_{j-1}$.
    b.  If $R < \epsilon$ (where $\epsilon$ is a small threshold, e.g., $10^{-5}$), the algorithm has stalled because the first view covered all pixels, leaving zero marginal gain for others [S1].
    c.  **Fallback:** If stalled, discard the greedy selection for this step and instead select the remaining candidate with the absolute maximum total score: $\arg\max_{c \notin V_{sel}} \sum_{p \in P} S_c(p)$ [S1].

5.  **Return:** Output $V_{sel}$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_cameras(mesh, cameras, novel_view_matrix, k, epsilon=1e-5):
    """
    Select k cameras from 'cameras' to maximize coverage in the novel view.
    cameras: list of camera objects with projection matrices and mesh data
    novel_view_matrix: projection matrix for the target novel view
    """
    # 1. Precompute score maps
    # S[i] is an array of size N (pixels in novel view)
    S = []
    for cam in cameras:
        # Project mesh to novel view, handle occlusions via depth test
        # Returns array of visibility scores (binary or density-based)
        score_map = compute_visibility_score_map(mesh, cam.projection, novel_view_matrix)
        S.append(score_map)
    
    N = len(S[0])
    covered = [False] * N
    V_sel = []
    V_prev = 0.0
    
    for step in range(1, k + 1):
        best_gain = -1
        best_cam_idx = -1
        best_total_score = -1
        
        # 2. Evaluate candidates
        for i, cam in enumerate(cameras):
            if i in V_sel:
                continue
            
            # Calculate marginal gain
            gain = 0.0
            for p in range(N):
                if not covered[p]:
                    gain += S[i][p]
            
            # Check for degeneracy fallback condition
            # If gain is effectively zero but we haven't covered everything,
            # or if the ratio check triggers.
            # Note: The source checks V_k / V_{k-1} < epsilon.
            # We need to track total covered score or count.
            
            # For the fallback logic, we also need the total sum of the candidate
            total_score = sum(S[i])
            
            if gain > best_gain:
                best_gain = gain
                best_cam_idx = i
                best_total_score = total_score
        
        # 3. Degeneracy Check
        # Calculate current total coverage score V_step
        # In binary mode, this is the count of covered pixels.
        # In density mode, it's the sum of max scores.
        # Assuming binary for simplicity of the ratio check described:
        current_covered_count = sum(1 for c in covered if c)
        if V_prev > 0 and current_covered_count / V_prev < epsilon:
            # Degenerate case: switch to absolute max score
            # Re-evaluate to find the one with highest total sum
            best_cam_idx = -1
            max_total = -1
            for i, cam in enumerate(cameras):
                if i in V_sel:
                    continue
                total = sum(S[i])
                if total > max_total:
                    max_total = total
                    best_cam_idx = i
        
        # 4. Update
        if best_cam_idx == -1:
            break # No more candidates
            
        V_sel.append(best_cam_idx)
        
        # Update coverage mask
        for p in range(N):
            if S[best_cam_idx][p] > 0:
                covered[p] = True
                
        V_prev = sum(1 for c in covered if c) # Or sum of scores if non-binary
        
    return V_sel

def compute_visibility_score_map(mesh, cam_proj, novel_proj):
    """
    Projects mesh to novel view space.
    Returns array of scores.
    Binary: 1 if visible, 0 otherwise.
    Density: ratio of distances.
    """
    # Implementation details depend on specific rasterization engine.
    # Core logic:
    # 1. Transform mesh vertices using cam_proj to get 2D coords in input space?
    # No, we need visibility in NOVEL view.
    # 1. Project mesh vertices to novel view space using novel_proj.
    # 2. For each pixel in novel view, find the closest point on the mesh.
    # 3. Check if that point is visible from the input camera (cam_proj).
    #    This requires back-projecting the novel-view point to 3D, 
    #    then projecting to input camera to check depth/visibility.
    # 4. Compute score.
    pass
```

**Parameters:**
*   $k$: Number of views to select. Typical values depend on GPU memory limits; often small integers (e.g., 4–16) [S1].
*   $\epsilon$: Threshold for degeneracy detection. Not explicitly given in sources, but implied to be a small positive constant [S1].

**Edge Cases and Pitfalls:**
*   **Degeneracy:** If the novel view is identical to an input view, the first selected view covers all pixels, resulting in zero marginal gain for subsequent views. This is detected when $V_{k+1}/V_k < \epsilon$ [S1]. The algorithm switches to selecting the view with the absolute maximum sum of scores [S1].
*   **Precision:** The algorithm relies on accurate depth testing and occlusion handling. Using a proxy mesh instead of point splatting is chosen for speed, but mesh quality affects accuracy [S1].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels in the novel view and $M$ is the number of input cameras. This is tractable for interactive rendering [S1].
*   **Memory Complexity:** $O(N \cdot M)$ to store the score maps for all input cameras in the novel view space [S1].
*   **Performance:** Provides faster and more accurate selection than ULR-style weights, superpixel-based IBR, or voxel grid approximations [S1].
*   **Strengths:** Guarantees a lower bound of $1 - 1/e$ on the ratio of the score compared to the optimal solution (greedy approximation guarantee for Maximum K-Coverage) [S1].
*   **Limitations:** Requires a proxy mesh from MVS, which may not be available or accurate for all scenes [S1]. Memory usage scales linearly with the number of input cameras and novel view resolution.

## Background
This method was introduced by G. Kopanas, J. Philip, T. Leimkühler & G. Drettakis in "Point-Based Neural Rendering with Per-View Optimization" [S1]. It builds upon the K-Maximum coverage algorithm [HP98] to address the limitations of previous camera selection methods like those by Riegler and Koltun [RK20], which ranked views based on overlap but did not necessarily maximize the union of coverage [S1]. The approach is part of a broader trend in neural rendering that leverages MVS proxy geometry for efficient per-view optimization [S1].

## Sources
[S1] Point-Based Neural Rendering with Per-View Optimization, https://www.youtube.com/watch?v=TupOewenSAk, fetched 2026-10-31
