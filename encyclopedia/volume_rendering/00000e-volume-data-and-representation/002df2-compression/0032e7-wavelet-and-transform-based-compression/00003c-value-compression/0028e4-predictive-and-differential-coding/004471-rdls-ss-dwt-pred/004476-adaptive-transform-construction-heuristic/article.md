# Adaptive Transform Construction Heuristic

Also called: Greedy RDLS Filter Selection Heuristic, Image-Adaptive Parameter Selection Heuristic

## Overview
The Adaptive Transform Construction Heuristic is a greedy parameter selection method for the RDLS-SS-DWT+Pred transform [S1]. It selects denoising filters for RDLS subbands and predictors for the final subbands to minimize the estimated bitrate of the compressed volume [S1]. It differs from exhaustive search by using a fast entropy estimation to approximate JP3D bitrate, allowing image-adaptive configuration without prohibitive computational cost [S1].

## How it works
**Inputs:**
- 3D volumetric data $V$.
- Set of candidate RDLS denoising filters $\mathcal{F} = \{f_1, f_2, \dots, f_N\}$ (including Null) [S1].
- Set of candidate predictors $\mathcal{P}$ for final subbands [S1].
- Transform depth $L$ [S1].

**Outputs:**
- Optimal filter assignment $\Phi$: a mapping from each subband at each level to a filter $f \in \mathcal{F}$ [S1].
- Optimal predictor assignment $\Psi$: a mapping from each final subband to a predictor $p \in \mathcal{P}$ [S1].

**Data Structures:**
- `SubbandOrder`: A list of 14 entries defining the processing order for a single transform level (see Table 2 below) [S1].
- `BitrateEstimator`: A function that estimates the JP3D bitrate given a specific transform configuration (filters and predictors) applied to $V$ [S1].

**Table 2: Subband Analysis Order**
| Order | Subband | Step | Type | Final/Temp | Complementary |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | HLL | Prediction | Temporary | Varies* | L |
| 2 | HHL | Update | Temporary | Varies* | H |
| 3 | HLH | Prediction | Temporary | Varies* | LL |
| 4 | HHH | Prediction | Temporary | Varies* | LH |
| 5 | LLL | Update | Temporary | Varies* | HL |
| 6 | LHL | Update | Temporary | Varies* | HH |
| 7 | LLH | Update | Temporary | Varies* | LLL |
| 8 | LHH | Update | Temporary | Varies* | LHL |
| 9 | HLL | Prediction | Final | Final | L |
| 10 | HHL | Update | Final | Final | H |
| 11 | HLH | Prediction | Final | Final | LL |
| 12 | HHH | Prediction | Final | Final | LH |
| 13 | LLL | Update | Final | Final | HL |
| 14 | LHL | Update | Final | Final | HH |

*Note: "Varies" indicates the subband is temporary at lower levels and final at the highest level [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize all subband filters to Null or a default filter [G].
2.  **Step A (Global Best Filter):**
    For each filter $f \in \mathcal{F}$:
    - Apply $f$ to all subbands at all transform levels.
    - Estimate the total bitrate $B_f$ using `BitrateEstimator` [S1].
    Select the filter $f_{best}$ that yields the minimum $B_f$.
    - Assign $f_{best}$ to all subbands at all levels [S1].
3.  **Step B (Local Refinement):**
    Repeat for $I$ iterations (where $I$ is a small integer, e.g., 2-5) [S1]:
    For each transform level $a$ from 1 to $L$:
    For each subband index $k$ from 1 to 14 (following `SubbandOrder`):
    - Let $S$ be the subband identified by level $a$ and order $k$.
    - Let $f_{current}$ be the filter currently assigned to $S$ [S1].
    - Let $B_{current}$ be the bitrate estimated with the current global configuration [S1].
    - For each candidate filter $f_{cand} \in \mathcal{F}$ where $f_{cand} \neq f_{current}$:
        - Temporarily assign $f_{cand}$ to $S$.
        - **Constraint Check:** If $S$ is a prediction step and $f_{cand}$ is Null, also assign Null to its complementary update subband (as per Table 2) [S1]. If $S$ is an update step and its complementary prediction step is already Null, $f_{cand}$ must be Null [G].
        - Estimate the new bitrate $B_{new}$ using `BitrateEstimator` [S1].
        - If $B_{new} < B_{current}$:
            - Update $B_{current} \leftarrow B_{new}$.
            - Update the filter assignment for $S$ (and its complement if applicable) to $f_{cand}$ [S1].
4.  **Predictor Selection:**
    For each final subband, select the predictor from $\mathcal{P}$ that minimizes the estimated bitrate, given the fixed RDLS filters [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_transform_heuristic(volume, filters, predictors, levels, iterations):
    # SubbandOrder defines the sequence of subbands to process at each level
    # Each entry: (subband_name, step_type, complementary_subband_name)
    subband_order = [
        ("HLL", "prediction", "L"), ("HHL", "update", "H"),
        ("HLH", "prediction", "LL"), ("HHH", "prediction", "LH"),
        ("LLL", "update", "HL"), ("LHL", "update", "HH"),
        ("LLH", "update", "LLL"), ("LHH", "update", "LHL"),
        ("HLL", "prediction", "L"), ("HHL", "update", "H"),
        ("HLH", "prediction", "LL"), ("HHH", "prediction", "LH"),
        ("LLL", "update", "HL"), ("LHL", "update", "HH")
    ]
    
    # Initialize filters for all subbands at all levels
    # filters_map[level][subband_name] = filter_index
    filters_map = initialize_filters(levels, subband_order)
    
    # Step A: Global Best Filter
    best_global_bitrate = infinity
    best_global_filter = None
    
    for f in filters:
        # Apply f to ALL subbands at ALL levels
        temp_filters_map = copy(filters_map)
        for level in range(1, levels + 1):
            for subband_name, _, _ in subband_order:
                temp_filters_map[level][subband_name] = f
        
        bitrate = estimate_jp3d_bitrate(volume, temp_filters_map, predictors)
        if bitrate < best_global_bitrate:
            best_global_bitrate = bitrate
            best_global_filter = f
            
    # Set all subbands to the best global filter
    for level in range(1, levels + 1):
        for subband_name, _, _ in subband_order:
            filters_map[level][subband_name] = best_global_filter
            
    # Step B: Local Refinement
    for iteration in range(iterations):
        for level in range(1, levels + 1):
            for order_idx, (subband_name, step_type, comp_subband) in enumerate(subband_order):
                current_filter = filters_map[level][subband_name]
                current_bitrate = estimate_jp3d_bitrate(volume, filters_map, predictors)
                best_local_filter = current_filter
                best_local_bitrate = current_bitrate
                
                for f_cand in filters:
                    if f_cand == current_filter:
                        continue
                        
                    # Temporarily apply candidate
                    filters_map[level][subband_name] = f_cand
                    
                    # Handle Null constraint for prediction/update pairs
                    if step_type == "prediction" and f_cand == "Null":
                        # Also set complementary update subband to Null
                        # Note: Complementary subband name in table refers to the 
                        # other subband in the pair at the same level.
                        # We must find the index of the complementary subband in the order
                        comp_idx = find_subband_index(subband_order, comp_subband)
                        filters_map[level][subband_order[comp_idx][0]] = "Null"
                    elif step_type == "update":
                        # If complementary prediction is Null, this must be Null
                        # Find complementary prediction subband
                        # This logic is implicit in the source's description of the pitfall
                        # and the order. The source says: "if Null is selected for a 
                        # prediction step, it is also selected for the complementary update step"
                        # It does not explicitly force update->prediction, but the pitfall 
                        # notes that update may be changed later. We assume the constraint 
                        # is one-way as per Step B description.
                        pass
                        
                    new_bitrate = estimate_jp3d_bitrate(volume, filters_map, predictors)
                    
                    if new_bitrate < best_local_bitrate:
                        best_local_bitrate = new_bitrate
                        best_local_filter = f_cand
                        
                # Commit best local filter
                filters_map[level][subband_name] = best_local_filter
                
    # Final Predictor Selection for final subbands
    for level in range(1, levels + 1):
        for subband_name, step_type, _ in subband_order:
            if step_type == "final": # Or check if it's the highest level
                # Select best predictor for this subband given fixed filters
                best_pred = select_best_predictor(volume, filters_map[level][subband_name], predictors)
                # Store predictor assignment
                pass
                
    return filters_map
```

**Edge Cases and Pitfalls:**
- **Null Constraint:** Selecting Null for a prediction step forces Null for the complementary update step [S1]. However, because prediction steps are processed before their complementary update steps in the heuristic order (Table 2), the update step's filter may be changed later in the iteration if a better non-Null filter is found, potentially violating the intended symmetry if not handled carefully [S1]. The implementation must ensure that if a prediction step is set to Null, the complementary update step is also set to Null and not overwritten by a subsequent local optimization step for the update subband unless the prediction step is also revisited [G].
- **Sparse Histograms:** For images with sparse intensity histograms, standard entropy estimation may be ineffective without Histogram Packing (HP) [S1]. HP should be applied before bitrate estimation to ensure accurate heuristic evaluation [S1].

## Operating characteristics
- **Time Complexity:** The heuristic is greedy and iterative. Step A requires $N$ full transform and bitrate estimations. Step B requires $I \times L \times 14 \times (N-1)$ estimations. Each estimation involves applying RDLS filters and estimating JP3D bitrate [S1]. This is significantly faster than exhaustive search ($N^{14L}$) but slower than a single transform [S1].
- **Memory Complexity:** Requires storing the volume data and intermediate transformed coefficients for bitrate estimation. No additional large data structures [S1].
- **Performance:** Improves compression ratio by up to 2% on average for 3D medical images compared to competitive methods like DA-DWT and JP3D+BP, which improved by less than 1% [S1].
- **Strengths:** Image-adaptive; reduces bitrate by optimizing filters per subband; computationally feasible compared to exhaustive search [S1].
- **Limitations:** Slows down the JP3D compressor due to repeated transform and estimation steps; sensitive to the quality of the fast entropy estimation [S1].

## Background
The heuristic was proposed in the context of improving JP3D compression of volumetric medical images using hybrid adaptive wavelet-based transforms [S1]. It builds on the NH heuristic used for 2D images [S1]. The method is part of a broader effort to combine RDLS with step skipping and prediction (RDLS-SS-DWT+Pred) to achieve better compression ratios than traditional DWT methods [S1].

## Sources
[S1] Employing New Hybrid Adaptive Wavelet-Based Transform and Histogram Packing to Improve JP3D Compression of Volumetric Medical Images, https://doi.org/10.3390/e22121385, fetched 2026-10-26
