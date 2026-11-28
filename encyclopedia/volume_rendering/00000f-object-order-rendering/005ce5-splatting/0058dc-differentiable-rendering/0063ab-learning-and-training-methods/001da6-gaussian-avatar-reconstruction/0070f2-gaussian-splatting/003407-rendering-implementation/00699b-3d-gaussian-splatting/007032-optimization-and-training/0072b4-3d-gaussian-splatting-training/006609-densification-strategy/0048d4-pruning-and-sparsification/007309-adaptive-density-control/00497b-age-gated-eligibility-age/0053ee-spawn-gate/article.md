# Spawn Gate

Also called: Gaussian spawn gate, Learned gate, Leave-one-scene-out gate, Rank veto, Survival prediction gate

## Overview
The Spawn Gate is a mechanism that predicts the survival probability of a Gaussian spawn candidate at birth time to veto low-survival candidates, thereby controlling the peak population size during training [S1, unofficial]. It sets itself apart from heuristic densification rules by using a learned predictor to rank candidates, ensuring a deterministic peak-memory reduction without degrading final image quality [S1, unofficial].

## How it works
The method operates during the densification window, specifically monitoring spawn events to veto candidates before they are instantiated [S1, unofficial].

**Inputs and Outputs**
*   **Input:** A set of spawn candidates $C_t$ generated at densification step $t$ [S1, unofficial]. Each candidate $j$ is associated with a parent Gaussian $j$ [S1, unofficial].
*   **Output:** A subset of instantiated candidates $C'_t = C_t \setminus V_t$, where $V_t$ is the set of vetoed candidates [S1, unofficial].

**Data Structures**
For each candidate $j$ spawned from parent $j$ at step $t$, a feature vector $x_{j,t}$ is assembled [S1, unofficial]. The layout is:
$$ x_{j,t} = [\bar{g}_j, \alpha_j, \bar{s}_j, \tau_j, d_j, \ell_j, t, 1_{[split]}] $$
Where:
*   $\bar{g}_j$: View-space positional gradient magnitude averaged over $d_j$ views that observed the parent [S1, unofficial].
*   $\alpha_j$: Parent opacity [S1, unofficial].
*   $\bar{s}_j$: Mean of the parent’s three scale axes [S1, unofficial].
*   $\tau_j$: Parent’s age in iterations since its own birth [S1, unofficial].
*   $d_j$: Densification denominator, the number of views that accumulated gradient onto the parent [S1, unofficial].
*   $\ell_j$: Child’s LOD level (parent’s level for a clone, one level finer for a split) [S1, unofficial].
*   $t$: Current training iteration [S1, unofficial].
*   $1_{[split]}$: Binary indicator, 1 if the candidate is a split, 0 if it is a clone [S1, unofficial].

**Parameters**
*   $\rho$: Veto fraction. Typical value: 0.6 [S1, unofficial]. This value is chosen as the largest reclaim inside the quality-neutral flat region; $\rho \le 0.6$ matches ungated quality, while decline appears at $\rho \ge 0.75$ [S1, unofficial].
*   Onset iteration: The iteration at which the gate activates. Typical value: 24k [S1, unofficial]. This is fixed by the sharp drop in survival rates observed after iteration 24k [S1, unofficial].
*   Gate window: The gate is active during the interval $[24k, 32k)$ [S1, unofficial].

**Algorithm**
1.  **Monitor Spawns:** During the gate window $[24k, 32k)$, collect all new spawn candidates $C_t$ at each densification step $t$ [S1, unofficial].
2.  **Feature Extraction:** For each candidate $j \in C_t$, extract the features $\bar{g}_j, \alpha_j, \bar{s}_j, \tau_j, d_j, \ell_j$ from the parent Gaussian and the current iteration $t$ [S1, unofficial]. Determine if the candidate is a split or clone to set $1_{[split]}$ [S1, unofficial].
3.  **Prediction:** Query a pre-trained gradient-boosted tree predictor $f$ with the feature vector $x_{j,t}$ to obtain a survival score $p_j$:
    $$ p_j = f(x_{j,t}) \in [0, 1] $$
    The predictor $f$ is trained on lineage data from other scenes (leave-one-scene-out) to ensure transferability [S1, unofficial].
4.  **Ranking:** Rank all candidates in $C_t$ by their predicted survival score $p_j$ in ascending order [S1, unofficial].
5.  **Veto Selection:** Select the top $\lceil \rho |C_t| \rceil$ candidates with the lowest predicted survival scores to form the veto set $V_t$:
    $$ V_t = \arg \min_{S \subseteq C_t, |S|=\lceil \rho |C_t| \rceil} \sum_{j \in S} p_j $$
    Vetoing by rank ensures the gate depends only on the ordering of $f$, allowing a single predictor to transfer across scenes with different candidate volumes and score calibrations [S1, unofficial].
6.  **Instantiation:** Instantiate only the remaining candidates:
    $$ C'_t = C_t \setminus V_t $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def spawn_gate(candidates, predictor, rho, current_iteration):
    """
    Veto low-survival spawn candidates based on a learned predictor.
    
    Args:
        candidates: List of spawn candidate objects. Each object has attributes:
            parent: The parent Gaussian object.
            is_split: Boolean, True if split, False if clone.
        predictor: A gradient-boosted tree model with a predict() method.
        rho: Float, fraction of candidates to veto (e.g., 0.6).
        current_iteration: Integer, current training step.
        
    Returns:
        List of instantiated candidates.
    """
    scored_candidates = []
    
    for cand in candidates:
        parent = cand.parent
        
        # Extract features
        g_bar = parent.avg_view_space_gradient_magnitude
        alpha = parent.opacity
        s_bar = (parent.scale_x + parent.scale_y + parent.scale_z) / 3.0
        tau = current_iteration - parent.birth_iteration
        d = parent.densification_denominator
        l = parent.lod_level + 1 if cand.is_split else parent.lod_level
        
        # Assemble feature vector
        is_split_val = 1.0 if cand.is_split else 0.0
        x = [g_bar, alpha, s_bar, tau, d, l, current_iteration, is_split_val]
        
        # Predict survival probability
        p = predictor.predict(x)
        
        scored_candidates.append({
            'candidate': cand,
            'score': p
        })
    
    # Sort by predicted survival score ascending (lowest first)
    scored_candidates.sort(key=lambda item: item['score'])
    
    # Determine number of candidates to veto: ceil(rho * |C_t|)
    num_to_veto = math.ceil(rho * len(scored_candidates))
    num_to_veto = max(0, min(num_to_veto, len(scored_candidates)))
    
    # Veto the bottom candidates
    vetoed_indices = set(range(num_to_veto))
    
    instantiated = []
    for i, item in enumerate(scored_candidates):
        if i not in vetoed_indices:
            instantiated.append(item['candidate'])
            
    return instantiated
```

**Edge Cases and Pitfalls**
*   **Threshold vs. Rank:** Using a fixed probability threshold instead of rank-based vetoing causes the actual veto rate to vary per scene because base survival rates differ across scenes [S1, unofficial]. Rank vetoing ensures a deterministic peak-population cut [S1, unofficial].
*   **Timing:** Delaying the veto decision beyond spawn time forfeits the peak-memory benefit because non-surviving spawns are allocated for the delay duration $\Delta$ [S1, unofficial]. Waiting until simplification recovers no peak [S1, unofficial].
*   **Predictor Transfer:** The predictor must be trained on scenes other than the evaluation scene (leave-one-scene-out) to ensure it generalizes [S1, unofficial]. Transfer is harder across domain gaps (e.g., synthetic scenes like MatrixCity) [S1, unofficial].

## Operating characteristics
*   **Memory Reduction:** On the Building scene with $\rho=0.6$ and onset 24k, peak memory is reduced from 10.89M (ungated) to 9.23M Gaussians, with PSAR remaining flat at 23.46 dB [S1, unofficial]. On Rubble, the learned gate captures 98% of the peak-population cut achievable by a perfect-foresight oracle, while matching the oracle's quality [S1, unofficial].
*   **Quality Impact:** All veto rates $\rho \le 0.6$ match the ungated quality, so vetoing up to 60% of late candidates is quality-neutral; a measurable decline appears only at $\rho \ge 0.75$ [S1, unofficial].
*   **Performance:** The predictor achieves a held-out AUC of 0.748–0.787 when trained and tested on the same scene's lineage [S1, unofficial]. Leave-one-scene-out predictors achieve an AUC of 0.750–0.770 on unseen scenes, dropping to 0.702 on the synthetic MatrixCity scene [S1, unofficial].
*   **Comparison:** The learned gate (AUC 0.750–0.770) significantly outperforms random vetoing [S1, unofficial]. Random vetoing discards true survivors at the 21.2% base rate, while the learned gate discards them at 11.1%, preserving quality [S1, unofficial].
*   **Strengths:** Deterministic peak-population control; transferable across scenes; quality-neutral at optimal settings [S1, unofficial].
*   **Limitations:** Requires a pre-trained predictor; performance degrades on out-of-distribution scenes (e.g., synthetic vs. real); sensitive to the onset iteration choice [S1, unofficial].

## Background
The Spawn Gate was introduced in the BlitzGS paper [S1, unofficial]. It addresses the issue of transient peak population sizes during 3D Gaussian Splatting training, which dictate per-GPU memory limits [S1, unofficial]. The method leverages the observation that survival to the final model is predictable from spawn-time features, particularly in the late densification window where survival rates drop sharply [S1, unofficial].

## Sources
[S1] BlitzGS: City-Scale Gaussian Splatting at Lightning Speed, https://doi.org/10.48550/arXiv.2605.13794, fetched 2026-10-29 (unofficial)
