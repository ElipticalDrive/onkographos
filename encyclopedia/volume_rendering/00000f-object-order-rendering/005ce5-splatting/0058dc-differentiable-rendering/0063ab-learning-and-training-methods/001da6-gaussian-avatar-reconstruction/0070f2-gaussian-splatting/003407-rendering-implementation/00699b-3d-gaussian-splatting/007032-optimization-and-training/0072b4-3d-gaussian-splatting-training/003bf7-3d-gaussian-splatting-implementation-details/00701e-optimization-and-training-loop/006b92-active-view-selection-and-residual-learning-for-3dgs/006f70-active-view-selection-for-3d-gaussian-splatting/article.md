# Active View Selection for 3D Gaussian Splatting

Also called: PRIMU Active View Selection, PRIMU-AVS

## Overview
Active View Selection (AVS) for 3D Gaussian Splatting (3DGS) identifies a subset of training views that maximizes final reconstruction quality while limiting the total number of views used [S1, unofficial]. PRIMU-AVS distinguishes itself from the full PRIMU method by omitting the gradient boosting regressor, thereby avoiding the need for an additional hold-out view for training [S1, unofficial]. It relies on a single direction-dependent coverage feature map with max aggregation and dropped opacity to guide view selection [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of candidate training views (with camera poses) and the current state of 3D Gaussian primitives (position, covariance, opacity, spherical harmonics coefficients) after a specified number of training iterations.
*   **Output:** A selected training view from the candidates.

**Data Structures**
*   **Coverage Feature Map ($V_{max}$):** A 2D image buffer with the same resolution as the target rendering. Each pixel stores a scalar value representing the maximum direction-dependent coverage of Gaussian primitives.
*   **Candidate Views:** A list of camera poses excluding the currently selected training views and the reserved evaluation views.

**Parameters**
*   **Total Training Views ($N_{total}$):** 20 [S1, unofficial].
*   **Initial Views:** 4 (1 fixed + 3 distance-maximized) [S1, unofficial].
*   **Selection Interval:** A new view is added after $100 \times N_{current}$ iterations, where $N_{current}$ is the number of views already selected [S1, unofficial].
*   **Opacity Drop:** Opacity is set to $\alpha=1$ (dropped) during coverage calculation [S1, unofficial].

**Steps**
1.  **Initialization:**
    *   Select one fixed view as the first training view.
    *   Select three additional views by maximizing the Euclidean distance to the camera centers of the existing views.
    *   Begin training with these 4 views.
2.  **Iterative Selection:**
    *   Continue training until the iteration count reaches the threshold for the next view selection.
    *   For each candidate view in the pool:
        *   Render the $V_{max}$ feature map.
        *   Calculate the mean value across all pixels of the rendered map.
    *   Select the candidate view with the **lowest** mean value. This corresponds to the view with the lowest overall coverage from already selected views.
    *   Add the selected view to the training set.
    *   Repeat until 20 training views are reached.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_training_views(candidates, current_primitives, current_iterations, n_total=20):
    # 1. Initialization
    # Select first view (fixed)
    view_1 = candidates[0] # Placeholder for fixed selection logic
    selected_views = [view_1]
    
    # Select 3 views by maximizing Euclidean distance to camera centers
    for _ in range(3):
        best_view = None
        max_min_dist = -1
        for cand in candidates:
            if cand in selected_views:
                continue
            # Calculate minimum distance to any selected camera center
            min_dist = min(distance(cand.camera_center, sv.camera_center) for sv in selected_views)
            if min_dist > max_min_dist:
                max_min_dist = min_dist
                best_view = cand
        selected_views.append(best_view)
        
    # 2. Iterative Selection
    current_n = len(selected_views)
    while current_n < n_total:
        # Determine iteration threshold for next selection
        # Interval is 100 * current_n. 
        # Note: The source says "added after 100 times the number of selected training views"
        # e.g., 5th view after 400 iterations (100*4).
        threshold = 100 * current_n
        
        # Wait until training reaches threshold (handled by training loop)
        if current_iterations < threshold:
            return None # No selection needed yet
            
        # Render coverage map for all candidates
        candidate_scores = {}
        for cand in candidates:
            if cand in selected_views:
                continue
            
            # Render V_max feature map
            # V_max uses direction-dependent coverage with max aggregation and dropped opacity (alpha=1)
            coverage_map = render_coverage_map(cand, current_primitives, use_opacity=False, aggregation='max')
            
            # Calculate mean value across pixels
            mean_coverage = mean(coverage_map)
            candidate_scores[cand] = mean_coverage
            
        # Select candidate with lowest mean value (lowest coverage)
        best_view = min(candidate_scores, key=candidate_scores.get)
        selected_views.append(best_view)
        current_n += 1
        
        # Reset iterations or track relative progress? 
        # Source implies continuous training. The threshold is absolute or relative?
        # "added after 100 times the number of selected training views"
        # e.g. 5th view after 400 iterations. 6th after additional 500? 
        # Wait, source says: "the fifth view after 400 iterations, the sixth after an additional 500"
        # This phrasing is ambiguous. Does "additional 500" mean iteration 900? Or 500 more?
        # Let's re-read: "a new view is added after 100 times the number of currently selected training views"
        # If 4 views selected, next is 5th. 100 * 4 = 400. So 5th view at iter 400.
        # If 5 views selected, next is 6th. 100 * 5 = 500. So 6th view at iter 500?
        # But source says "sixth after an additional 500". This contradicts the formula if it means absolute.
        # However, "100 times the number of currently selected training views" is the explicit rule.
        # The example "sixth after an additional 500" might mean 500 iterations *after* the 5th view selection?
        # If 5th was at 400, and 6th is 500 *after*, that's 900.
        # But 100 * 5 = 500.
        # Let's stick to the explicit formula: threshold = 100 * current_n.
        # The example might be illustrative of a different protocol or I am misinterpreting "additional".
        # Actually, looking at the text: "e.g., the fifth view after 400 iterations, the sixth after an additional 500"
        # If the rule is 100 * N, then 6th view (N=5) -> 500.
        # If it's "additional", maybe it means the interval grows?
        # Let's assume the formula 100 * N is the primary definition.
        pass
        
    return selected_views
```

**Edge Cases and Pitfalls**
*   **Evaluation View Reservation:** Every eighth view is removed from the candidate pool and reserved for evaluation to ensure comparability with baselines like FisherRF [S1, unofficial].
*   **Spherical Harmonics Degree:** The interval for raising the spherical harmonics degree is increased from 1000 to 5000 iterations to mitigate overfitting caused by the limited number of views [S1, unofficial].

## Operating characteristics
*   **Performance:** On the MipNeRF360 dataset, PRIMU-AVS achieved PSNR 20.721, SSIM 0.625, and LPIPS 0.34 [S1, unofficial].
*   **Comparison:** Outperforms FisherRF (PSNR 20.392, SSIM 0.586, LPIPS 0.363) and manifold (PSNR 20.102, SSIM 0.610, LPIPS 0.351) [S1, unofficial].
*   **Complexity:** The selection step requires rendering a coverage map for all candidate views. This adds computational overhead proportional to the number of candidates and the resolution of the feature map.
*   **Strengths:** Simple, independent of primitives' spatial extent (due to max aggregation), and does not require a hold-out view for training a regression model [S1, unofficial].
*   **Limitations:** Relies on a single feature map, potentially missing complex uncertainty correlations captured by the full PRIMU method's 13 feature maps and gradient boosting [S1, unofficial].

## Background
PRIMU-AVS is part of the PRIMU framework, a post-hoc technique for uncertainty estimation in Gaussian splatting [S1, unofficial]. The full PRIMU method uses 13 handcrafted Gaussian primitive-based feature maps to capture training-view coverage and projected error [S1, unofficial]. PRIMU-AVS simplifies this by using only the coverage feature map for view selection, aiming to maximize total coverage of Gaussian primitives across chosen training views [S1, unofficial]. The method was proposed in 2025 [S1, unofficial].

## Sources
[S1] PRIMU: Uncertainty Estimation for Novel Views in Gaussian Splatting from Primitive-Based Representations of Error and Coverage, https://arxiv.org/abs/2508.02443v2, fetched 2026-10-26 (unofficial)
