# RDLS-SS-DWT+Pred

Also called: Hybrid Transform, Reversible Denoising Lifting Steps with Step Skipping and Prediction

## Overview
RDLS-SS-DWT+Pred is a lossless compression transform for 3D volumetric data that combines the Reversible Denoising Lifting Steps with Step Skipping (RDLS-SS-DWT) with predictive coding applied to the final subbands [S1]. It solves the problem of high entropy in wavelet coefficients by further decorrelating the data using spatial prediction, achieving significant bitrate reductions compared to standard DWT-based methods [S1]. It differs from its sibling [SS-DWT+Pred](5256-ss-dwt-pred.md) by incorporating denoising filters within the lifting steps, though sources indicate that using actual denoising filters may be avoided without sacrificing bitrate improvements, making it functionally similar to the simpler variant [S1].

## How it works
The method constructs a hybrid transform by applying RDLS-SS-DWT to the input volume $V$, followed by a prediction step on the resulting final subbands.

**Inputs and Outputs:**
*   Input: 3D volumetric data $V$ with dimensions $D_x \times D_y \times D_z$.
*   Output: A set of prediction errors for each final subband, ready for entropy encoding.

**Data Structures:**
The RDLS-SS-DWT produces a hierarchical decomposition. The "final subbands" refer to the detail subbands at the deepest decomposition level. For a 3-level decomposition, these are the subbands at level 3. The specific subbands depend on the RDLS configuration, but generally include high-frequency components in X, Y, and Z directions.

**Parameters:**
*   Decomposition level: 3 (fixed in experiments) [S1].
*   Candidate predictors: A set of spatial predictors defined by their neighborhood weights [S1].
*   NOP predictor: A fallback predictor that assumes the value is 0 [S1].

**Steps:**

1.  **Transform:** Apply RDLS-SS-DWT to the input volume $V$. This involves applying reversible denoising lifting steps and skipping specific lifting steps based on a heuristic (either based on actual JP3D bitrates or an entropy estimator) [S1].
2.  **Identify Final Subbands:** Extract the detail subbands at the final decomposition level (Level 3) [S1].
3.  **Predictor Selection:** For each final subband, evaluate a set of candidate predictors. The selection criterion is the memoryless entropy of the prediction errors.
    *   The memoryless entropy $H$ is calculated as:
        $$ H = -\sum_{i=\text{MinPE}}^{\text{MaxPE}} p_i \log_2(p_i) $$
        where $M$ is the number of samples in the subband, $\text{MinPE}$ and $\text{MaxPE}$ are the smallest and greatest prediction errors respectively, and $p_i$ is the probability of occurrence of prediction error $i$. By convention, $0 \log_2 0 = 0$ [S1].
    *   The predictor yielding the minimum $H$ is selected for that subband.
    *   If all actual predictors increase entropy compared to the NOP predictor, the NOP predictor is used [S1].
4.  **Prediction Application:** Apply the selected predictor to the samples in the subband.
    *   Prediction is performed in raster scan order: volume slices from front to back, slice rows from top to bottom, pixels in a row from left to right [S1].
    *   If a predictor requires neighbors that are out of bounds (e.g., for the front-top-left pixel), a simpler predictor (e.g., NOP) is used for that specific sample [S1].
5.  **Output:** The result is the set of prediction errors for all final subbands.

**Candidate Predictors:**
The source defines a set of candidate predictors based on neighboring samples $A, B, C, D, E, F, G, H$ in a 3D neighborhood. The notation $P_X, P_Y, P_Z$ refers to predictions along axes. The specific predictors include:
*   NOP: Predicts 0.
*   $P_X, P_Y, P_Z$: Simple directional predictors (exact formulas not explicitly given in text, but implied by context of "Axial", "Horizontal", "Vertical" labels in Table 1 of source) [S1].
*   AVG_XY, AVG_XZ, AVG_YZ: Averages of neighbors in respective planes.
*   MED_XY, MED_XZ, MED_YZ: Medians of neighbors in respective planes.
*   AVG3D: Average of 3D neighbors.
*   Specific weighted combinations like $(A+B)/2$, $(A+D)/2$, etc., and median-based predictors like $\text{median}(A, B, A+B-C)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rdls_ss_dwt_plus_pred(volume):
    """
    Applies RDLS-SS-DWT followed by predictive coding to the final subbands.
    volume: 3D numpy array of shape (Dx, Dy, Dz)
    """
    # Step 1: Apply RDLS-SS-DWT
    # The heuristic for RDLS filter selection and step skipping is assumed to be
    # handled by a separate function or pre-configured.
    # Returns a dictionary of subbands, e.g., {'L3': subband_3, ...}
    # Note: RDLS-SS-DWT implementation details are in [Parent](4393-predictive-and-differential-coding.md)
    subbands = rdls_ss_dwt(volume, decomposition_level=3)
    
    final_subbands = get_final_subbands(subbands, level=3)
    
    prediction_errors = {}
    
    # Candidate predictors definition
    # Note: Exact neighbor indices for A, B, C... depend on the specific subband orientation
    # and the raster scan order. The source Table 1 defines the logic.
    candidate_predictors = [
        PredictorNOP(),
        PredictorAXIAL(),
        PredictorHORIZONTAL(),
        PredictorVERTICAL(),
        PredictorAVG_XY(),
        PredictorAVG_XZ(),
        PredictorAVG_YZ(),
        PredictorMED_XY(),
        PredictorMED_XZ(),
        PredictorMED_YZ(),
        PredictorAVG3D(),
        PredictorWeightedSum1(),
        PredictorWeightedSum2(),
        PredictorWeightedSum3(),
        PredictorMedian1(),
        PredictorMedian2(),
        PredictorMedian3()
    ]
    
    for subband_name, subband_data in final_subbands.items():
        best_predictor = None
        min_entropy = float('inf')
        
        # Step 3: Evaluate predictors
        for predictor in candidate_predictors:
            # Step 4: Apply predictor and compute errors
            errors = apply_predictor(subband_data, predictor)
            
            # Compute memoryless entropy
            entropy = compute_memoryless_entropy(errors)
            
            if entropy < min_entropy:
                min_entropy = entropy
                best_predictor = predictor
        
        # Apply best predictor to get final errors
        final_errors = apply_predictor(subband_data, best_predictor)
        prediction_errors[subband_name] = final_errors
        
    return prediction_errors

def compute_memoryless_entropy(errors):
    """
    Computes H = -sum(p_i * log2(p_i))
    """
    # Count occurrences of each error value
    unique, counts = np.unique(errors, return_counts=True)
    total_samples = len(errors)
    
    entropy = 0.0
    for count in counts:
        if count > 0:
            p_i = count / total_samples
            if p_i > 0:
                entropy -= p_i * np.log2(p_i)
    return entropy

def apply_predictor(subband, predictor):
    """
    Applies a predictor to the subband in raster scan order.
    Handles boundary conditions by falling back to NOP if neighbors are invalid.
    """
    errors = np.zeros_like(subband)
    # Raster scan order: z, y, x
    for z in range(subband.shape[0]):
        for y in range(subband.shape[1]):
            for x in range(subband.shape[2]):
                try:
                    predicted_value = predictor.predict(subband, x, y, z)
                    errors[z, y, x] = subband[z, y, x] - predicted_value
                except BoundaryError:
                    # Fallback to NOP if predictor cannot be computed
                    errors[z, y, x] = subband[z, y, x] # NOP predicts 0, so error = value - 0
    return errors
```

## Subtopics

### [Adaptive Transform Construction Heuristic](11258-adaptive-transform-construction-heuristic.md)
A greedy parameter selection method for the RDLS-SS-DWT part of the transform. It selects denoising filters and steps to skip to minimize estimated bitrate, using a fast entropy estimator instead of actual JP3D bitrates to reduce computational cost.

## Operating characteristics
*   **Time Complexity:** The transform itself is linear in the number of voxels. The predictor selection adds a constant factor overhead (number of candidate predictors) per subband. The entropy estimation for heuristic selection can be costly if actual JP3D bitrates are used, but is low-cost if an entropy estimator is used [S1].
*   **Memory Complexity:** Requires storage for the volume and the subbands. No significant additional memory overhead compared to standard DWT, aside from temporary buffers for prediction errors [S1].
*   **Performance:** In experiments on medical images (CT, MRI, US), RDLS-SS-DWT+Pred achieved bitrate reductions ranging from -0.175% to -8.973% compared to standard DWT [S1]. The improvement was most significant for MRI1 (-8.973%) [S1].
*   **Strengths:** Significant lossless compression improvement, especially for medical images. Adaptive to data characteristics via predictor selection.
*   **Limitations:** The heuristic for RDLS filter selection can be slow if based on actual bitrates. The benefit of using actual denoising filters (RDLS) over simpler step skipping (SS-DWT) is marginal, suggesting the complexity of RDLS may not be justified [S1].
*   **When to use:** When lossless compression of 3D volumetric data is required and the data exhibits spatial correlation that can be exploited by prediction. Particularly effective for medical imaging data [S1].
*   **When not to use:** If computational speed is critical and the marginal gain over SS-DWT+Pred is not worth the complexity of RDLS. If the data is already highly decorrelated [S1].

## Background
The method was proposed in 2020 by researchers from Vrije Universiteit Brussel (VUB) and Interdisciplinary Institute for BroadBand Technology (IBBT) [S1]. It builds on previous work on 2D-DWT variants using step skipping and reversible denoising lifting steps [S1]. The implementation was provided as a patch to IRIS-JP3D [S1]. The initial phase of the research was presented in a conference report [S1].

## Sources
[S1] Employing New Hybrid Adaptive Wavelet-Based Transform and Histogram Packing to Improve JP3D Compression of Volumetric Medical Images, https://doi.org/10.3390/e22121385, fetched 2026-10-26
