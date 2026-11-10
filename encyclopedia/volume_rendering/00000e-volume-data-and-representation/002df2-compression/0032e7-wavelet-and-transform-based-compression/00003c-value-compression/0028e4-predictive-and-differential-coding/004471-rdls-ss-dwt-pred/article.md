# RDLS-SS-DWT+Pred

Also called: Hybrid Transform, Reversible Denoising Lifting Steps with Step Skipping and Prediction

## Overview
RDLS-SS-DWT+Pred is a lossless compression transform for volumetric data that combines the 3D-DWT with Reversible Denoising Lifting Steps (RDLS) and Step Skipping (SS), followed by predictive coding on the final subbands [S1]. It solves the problem of residual entropy remaining in wavelet subbands by applying spatial prediction, similar to techniques used in 2D image compression [S1]. It differs from its sibling [SS-DWT+Pred](5256-residual-updates-in-transform-domain.md) by incorporating RDLS filters to reduce noise artifacts, though sources indicate that using actual denoising filters may be avoided without sacrificing bitrate improvements, making it functionally similar to [SS-DWT+Pred](5256-residual-updates-in-transform-domain.md) in practice [S1].

## How it works
**Inputs:** A 3D volumetric image $V$ of dimensions $D_x \times D_y \times D_z$.
**Outputs:** A compressed bitstream representing the transformed and predicted coefficients.

**Data Structures:**
*   **Subbands:** The output of the 3-level 3D-DWT decomposition. Each subband is a 3D array of coefficients.
*   **Prediction Errors:** Residuals computed as $r = v - \hat{v}$.
*   **Entropy Model:** A histogram of prediction errors used to calculate memoryless entropy.

**Parameters:**
*   **Decomposition Level:** Fixed at 3 levels for all experiments [S1].
*   **RDLS Filters:** Selected from a set of candidates (including Null and None) [S1].
*   **Step Skipping:** Selected from a set of candidates [S1].
*   **Predictor Set:** A set of candidate predictors (see Table 1 below) [S1].
*   **NOP Predictor:** A candidate predictor that predicts each sample is 0, used if actual predictors increase subband entropy [S1].

**Candidate Predictors:**
The following predictors are defined based on spatial neighbors. Let $A, B, C, D, E, F, G, H$ be the values of neighboring voxels relative to the current voxel $P$ being predicted. The prediction order is raster scan: volume slices from front to back, slice rows from top to bottom, pixels in a row from left to right [S1].

| Predictor | Prediction Formula | Context Neighbors |
| :--- | :--- | :--- |
| NOP | $0$ | None |
| $P_X$ | $0$ | None (Horizontal) |
| $P_Y$ | $0$ | None (Vertical) |
| $P_Z$ | $0$ | None (Axial) |
| $AVG_{XY}$ | $(A + B) / 2$ | $A, B$ in current slice |
| $AVG_{XZ}$ | $(A + D) / 2$ | $A, D$ in adjacent slices |
| $AVG_{YZ}$ | $(B + D) / 2$ | $B, D$ in adjacent slices |
| $MED_{XY}$ | $\text{median}(A, B, A + B - C)$ | $A, B, C$ in current slice |
| $MED_{XZ}$ | $\text{median}(A, D, A + D - E)$ | $A, D, E$ in adjacent slices |
| $MED_{YZ}$ | $\text{median}(B, D, B + D - F)$ | $B, D, F$ in adjacent slices |
| $AVG3D$ | $(A + B + D) / 3$ | $A, B, D$ in current and adjacent slices |

*Note: The specific geometric arrangement of neighbors $A-H$ corresponds to Figure 2 in [S1], where $A$ is the left neighbor, $B$ is the top neighbor, $C$ is the top-left neighbor, $D$ is the neighbor in the previous slice (same row/col), etc. If a neighbor is out of bounds, a simpler predictor (e.g., NOP) is used [S1].*

**Steps:**
1.  **Transform Construction:** Apply the 3D-DWT with RDLS and Step Skipping to the input volume $V$. The RDLS filters and step skipping patterns are selected using a heuristic (see [Adaptive Transform Construction Heuristic](11258-adaptive-transform-construction-heuristic.md)) to minimize estimated bitrate [S1].
2.  **Subband Processing:** For each final subband produced by the transform:
    a.  Iterate through all samples in raster scan order [S1].
    b.  For each sample, compute the prediction $\hat{v}$ using each candidate predictor from the set. If a predictor cannot be computed due to boundary conditions, use NOP [S1].
    c.  Compute the prediction error $r_i = v_i - \hat{v}_i$ for each candidate predictor.
    d.  Calculate the memoryless entropy $H$ for each candidate predictor's error distribution using the formula:
        $$ H = -\sum_{i=\text{MinPE}}^{\text{MaxPE}} p_i \log_2(p_i) $$
        where $M$ is the number of samples in the subband, $\text{MinPE}$ and $\text{MaxPE}$ are the smallest and greatest prediction errors, $p_i$ is the probability of occurrence of prediction error $i$, and $0 \log_2 0 = 0$ [S1].
    e.  Select the predictor that results in the smallest $H$. If the NOP predictor yields the smallest entropy, use NOP (effectively skipping prediction) [S1].
    f.  Generate the final residuals using the selected predictor.
3.  **Entropy Coding:** Entropy-code the residuals from all subbands using JP3D entropy coding [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rdls_ss_dwt_plus_pred(volume, rdls_filters, step_skipping, predictor_set):
    # Step 1: Apply Transform
    # Note: RDLS and Step Skipping selection is handled by the heuristic
    # outside this core transform function.
    subbands = apply_3d_dwt_with_rdls_and_ss(volume, rdls_filters, step_skipping)
    
    all_residuals = []
    
    # Step 2: Process each subband
    for subband in subbands:
        subband_residuals = []
        # Raster scan order: z (front to back), y (top to bottom), x (left to right)
        for z in range(subband.shape[0]):
            for y in range(subband.shape[1]):
                for x in range(subband.shape[2]):
                    current_value = subband[z, y, x]
                    
                    best_predictor = None
                    min_entropy = infinity
                    
                    # Evaluate all candidate predictors
                    for predictor in predictor_set:
                        try:
                            # Get neighbors based on predictor type
                            neighbors = get_neighbors(subband, x, y, z, predictor)
                            
                            # Compute prediction
                            prediction = predictor.predict(neighbors)
                            
                            # Compute residual
                            residual = current_value - prediction
                            
                            # Store residual for this predictor
                            # (In practice, we might accumulate errors to compute entropy)
                            # Here we assume we compute entropy over the whole subband
                            # for each predictor candidate.
                        except BoundaryError:
                            # If predictor cannot be computed, use NOP
                            prediction = 0
                            residual = current_value - prediction
                            
                        # Accumulate residuals for entropy calculation
                        # (Implementation detail: collect all residuals for the subband
                        # for each predictor before selecting the best one)
                        pass
                    
                    # After collecting residuals for all predictors for the whole subband:
                    # Calculate entropy for each predictor's error distribution
                    # Select predictor with minimum entropy
                    # Apply selected predictor to generate final residuals for this subband
                    pass
        
        all_residuals.append(subband_residuals)
        
    # Step 3: Entropy Code
    bitstream = jp3d_entropy_encode(all_residuals)
    return bitstream
```

## Subtopics

### [Adaptive Transform Construction Heuristic](11258-adaptive-transform-construction-heuristic.md)
A greedy parameter selection method for the RDLS-SS-DWT+Pred transform. It selects denoising filters for RDLS subbands and predictors for the final subbands to minimize the estimated bitrate of the compressed volume. It differs from exhaustive search by using a fast entropy estimation to approximate JP3D bitrate, allowing image-adaptive configuration without prohibitive computational cost.

## Operating characteristics
*   **Time Complexity:** The transform itself is $O(N)$ where $N$ is the number of voxels. However, the heuristic selection of RDLS filters and step skipping patterns can be computationally expensive if based on actual JP3D bitrates, as entropy coding is the slowest part of JP3D [S1]. Using entropy estimation reduces this cost [S1].
*   **Memory Complexity:** Requires storage for the volume and all subbands. The heuristic may require temporary storage for entropy calculations across candidate predictors [S1].
*   **Performance:** In experiments on medical images (CT, MRI, US), RDLS-SS-DWT+Pred showed bitrate improvements ranging from -0.175% to -8.973% compared to standard DWT [S1]. The improvement is most significant for MRI1 (-8.973%) [S1].
*   **Strengths:** Effective at reducing entropy in final subbands, especially when Null filters are used in RDLS-SS-DWT [S1]. Adaptive to image characteristics via the heuristic [S1].
*   **Limitations:** The heuristic selection can be slow if based on actual bitrates [S1]. The benefit of RDLS filters over no filters (SS-DWT+Pred) is negligible in many cases, suggesting the added complexity of RDLS may not be justified [S1].
*   **When to use:** When lossless compression of volumetric medical images is required and the computational cost of the heuristic is acceptable. Particularly effective when combined with histogram packing [S1].
*   **When not to use:** When real-time compression is required and the heuristic selection time is prohibitive. If the bitrate improvement from RDLS is negligible, [SS-DWT+Pred](5256-residual-updates-in-transform-domain.md) may be preferred for its simplicity [S1].

## Background
The method was proposed in 2020 by researchers from Vrije Universiteit Brussel (VUB) and Interdisciplinary Institute for BroadBand Technology (IBBT) [S1]. It builds on previous work on 2D-DWT variants using step skipping and reversible denoising lifting steps [S1]. The initial phase of the research was presented in a conference report [S1]. The implementation is available as a patch to IRIS-JP3D [S1].

## Sources
[S1] Employing New Hybrid Adaptive Wavelet-Based Transform and Histogram Packing to Improve JP3D Compression of Volumetric Medical Images, https://doi.org/10.3390/e22121385, fetched 2026-10-26
