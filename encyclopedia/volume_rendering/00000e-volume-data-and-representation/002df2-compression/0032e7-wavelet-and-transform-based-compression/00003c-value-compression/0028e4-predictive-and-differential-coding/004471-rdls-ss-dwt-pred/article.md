# RDLS-SS-DWT+Pred

Also called: Hybrid Transform, Reversible Denoising Lifting Steps with Step Skipping and Prediction

## Overview
RDLS-SS-DWT+Pred is a lossless volumetric compression method that constructs a hybrid transform by applying predictive coding to the final subbands of a Reversible Denoising Lifting Steps with Step Skipping Discrete Wavelet Transform (RDLS-SS-DWT) [S1]. It solves the problem of high entropy in transformed volumetric data by reducing residual variance through spatial prediction, specifically leveraging the characteristics of subbands generated when Null filters are used in the RDLS stage [S1]. It sets itself apart from its parent [Predictive and Differential Coding](4393-predictive-and-differential-coding.md) by integrating prediction into a multi-level 3D wavelet hierarchy rather than applying it directly to raw voxels, and from its sibling [SS-DWT+Pred](5256-residual-updates-in-transform-domain.md) by optionally including reversible denoising lifting steps, though sources indicate the denoising steps may be omitted without significant bitrate loss [S1].

## How it works
The method operates in two phases: transform construction and predictive coding.

**Inputs and Outputs:**
*   **Input:** A 3D volumetric dataset $V$ of dimensions $W \times H \times D$.
*   **Output:** A compressed bitstream containing entropy-coded prediction residuals.

**Data Structures:**
*   **Subbands:** The RDLS-SS-DWT decomposes $V$ into multiple frequency subbands. The method focuses on the "final subbands" (highest frequency detail coefficients) produced by the transform.
*   **Candidate Predictors:** A fixed set of spatial predictors defined by their neighbor dependencies (see Table 1 below).
*   **Entropy Estimator:** A histogram of prediction errors used to calculate memoryless entropy.

**Parameters:**
*   **Decomposition Level:** Set to 3 levels for all experiments [S1].
*   **RDLS Filters:** Can be Null, None, or actual denoising filters. Sources suggest Null/None filters are sufficient for comparable performance [S1].
*   **Step Skipping:** Determined by a heuristic based on either actual JP3D bitrates or an entropy estimator [S1].

**Candidate Predictors:**
The following predictors are available. $A, B, C, D, E, F, G, H$ represent voxel values at specific relative coordinates in the raster scan order (front-to-back, top-to-bottom, left-to-right). $P_X, P_Y, P_Z$ represent axial, horizontal, and vertical neighbors respectively.

| Predictor | Formula | Neighbors Used |
| :--- | :--- | :--- |
| NOP | $0$ | None |
| $P_X$ | $A$ | Left neighbor |
| $P_Y$ | $B$ | Top neighbor |
| $P_Z$ | $C$ | Front neighbor (previous slice) |
| $AVG_{XY}$ | $(A + B) / 2$ | Left, Top |
| $AVG_{XZ}$ | $(A + C) / 2$ | Left, Front |
| $AVG_{YZ}$ | $(B + C) / 2$ | Top, Front |
| $MED_{XY}$ | $\text{median}(A, B, A+B-C)$ | Left, Top, Front |
| $MED_{XZ}$ | $\text{median}(A, C, A+C-B)$ | Left, Front, Top |
| $MED_{YZ}$ | $\text{median}(B, C, B+C-A)$ | Top, Front, Left |
| $AVG3D$ | $(A + B + C) / 3$ | Left, Top, Front |

*Note: The specific geometric layout of neighbors A-H corresponds to Figure 2 in [S1], where A is the left neighbor, B is the top neighbor, C is the front neighbor (slice $i-1$), and D, E, F, G, H are further neighbors in the current and previous slices. For implementation, standard 3D spatial neighbors (Left, Top, Front) are sufficient for the primary predictors listed.*

**Algorithm Steps:**

1.  **Transform Construction:**
    *   Apply RDLS-SS-DWT to $V$ with 3 decomposition levels.
    *   Select RDLS filters and step-skipping patterns using a heuristic. The heuristic minimizes either the actual JP3D bitrate or an estimated entropy [S1].
    *   *Heuristic Note:* If using entropy estimation, calculate the memoryless entropy for candidate configurations. If using actual bitrate, encode with JP3D and measure [S1].

2.  **Predictor Selection (Per Subband):**
    *   For each final subband of the transformed volume:
        *   Iterate through the set of candidate predictors.
        *   For each predictor, compute the prediction errors $e = v - \hat{v}$ for all samples $v$ in the subband.
        *   Compute the memoryless entropy $H$ of the errors:
            $$ H = -\sum_{i=\text{MinPE}}^{\text{MaxPE}} p_i \log_2(p_i) $$
            where $M$ is the number of samples, $\text{MinPE}$ and $\text{MaxPE}$ are the minimum and maximum prediction errors, $p_i$ is the probability of error $i$, and $0 \log_2 0 = 0$ [S1].
        *   Select the predictor that yields the minimum $H$.
        *   If all predictors increase entropy compared to NOP, select NOP [S1].

3.  **Prediction and Residual Generation:**
    *   Process the subband in raster scan order: volume slices from front to back, slice rows from top to bottom, pixels in a row from left to right [S1].
    *   For each voxel $v$:
        *   Compute $\hat{v}$ using the selected predictor.
        *   If the required neighbors are out of bounds (e.g., front-top-left pixel for $P_Z$), fallback to a simpler predictor (e.g., NOP) [S1].
        *   Compute residual $d = v - \hat{v}$.

4.  **Entropy Encoding:**
    *   Encode the residuals $d$ using an entropy coder (e.g., Huffman, Arithmetic, or JP3D's specific coder) [S1].

5.  **Reconstruction:**
    *   Decode residuals $d$.
    *   Apply the inverse predictor: $v_{rec} = \hat{v} + d$.
    *   Apply the inverse RDLS-SS-DWT to reconstruct $V$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rdls_ss_dwt_plus_pred(volume, decomposition_level=3):
    """
    Compress a 3D volume using RDLS-SS-DWT+Pred.
    volume: 3D numpy array or similar structure [D, H, W]
    Returns: compressed bitstream (bytes)
    """
    # 1. Transform Construction
    # Apply RDLS-SS-DWT. The heuristic for filter selection and step skipping
    # is assumed to be handled by a helper function that returns the transform
    # parameters or the transformed subbands directly.
    # For implementation, we assume the transform produces a list of subbands.
    subbands = apply_rdls_ss_dwt(volume, decomposition_level)
    
    compressed_data = []
    
    # 2. Process each final subband
    for subband in subbands:
        # Determine optimal predictor for this subband
        best_predictor = select_best_predictor(subband)
        
        # 3. Prediction and Residual Generation
        residuals = []
        D, H, W = subband.shape
        
        for d in range(D):
            for h in range(H):
                for w in range(W):
                    # Get current voxel value
                    v = subband[d, h, w]
                    
                    # Compute prediction
                    pred_val = compute_prediction(subband, d, h, w, best_predictor)
                    
                    # Compute residual
                    residual = v - pred_val
                    residuals.append(residual)
        
        # 4. Entropy Encoding
        # Encode residuals using the chosen entropy coder (e.g., JP3D context)
        # This step depends on the specific entropy coding implementation.
        # Here we assume a generic entropy encoder.
        encoded_residuals = entropy_encode(residuals)
        compressed_data.append(encoded_residuals)
        
    return combine_bitstreams(compressed_data)

def select_best_predictor(subband):
    """
    Select the predictor that minimizes memoryless entropy of prediction errors.
    """
    candidates = [NOP, PX, PY, PZ, AVG_XY, AVG_XZ, AVG_YZ, MED_XY, MED_XZ, MED_YZ, AVG3D]
    best_entropy = float('inf')
    best_pred = NOP
    
    for pred_func in candidates:
        errors = []
        D, H, W = subband.shape
        for d in range(D):
            for h in range(H):
                for w in range(W):
                    v = subband[d, h, w]
                    pred_val = compute_prediction(subband, d, h, w, pred_func)
                    errors.append(v - pred_val)
        
        # Calculate memoryless entropy
        entropy = calculate_memoryless_entropy(errors)
        
        if entropy < best_entropy:
            best_entropy = entropy
            best_pred = pred_func
            
    return best_pred

def compute_prediction(subband, d, h, w, predictor):
    """
    Compute predicted value based on spatial neighbors.
    Fallback to NOP if neighbors are out of bounds.
    """
    # Define neighbor access with bounds checking
    def get_val(d, h, w):
        if 0 <= d < subband.shape[0] and 0 <= h < subband.shape[1] and 0 <= w < subband.shape[2]:
            return subband[d, h, w]
        return None

    # Neighbor coordinates (assuming raster order: d=depth/slice, h=height/row, w=width/col)
    # A: Left (w-1), B: Top (h-1), C: Front (d-1)
    # Note: The exact mapping of A-H to coordinates depends on Figure 2 in [S1].
    # Standard 3D raster order:
    # Left: (d, h, w-1)
    # Top: (d, h-1, w)
    # Front: (d-1, h, w)
    
    A = get_val(d, h, w-1)
    B = get_val(d, h-1, w)
    C = get_val(d-1, h, w)
    
    # Additional neighbors for more complex predictors (D, E, F, G, H)
    # D: Top-Left (d, h-1, w-1)
    # E: Front-Left (d-1, h, w-1)
    # F: Front-Top (d-1, h-1, w)
    # G: Front-Top-Left (d-1, h-1, w-1)
    # H: Back-Right (d+1, h+1, w+1) - usually not needed for causal prediction
    
    D_val = get_val(d, h-1, w-1)
    E_val = get_val(d-1, h, w-1)
    F_val = get_val(d-1, h-1, w)
    G_val = get_val(d-1, h-1, w-1)
    
    # Check if required neighbors are available
    if predictor == NOP:
        return 0
    elif predictor == PX:
        if A is not None: return A
        return 0 # Fallback
    elif predictor == PY:
        if B is not None: return B
        return 0
    elif predictor == PZ:
        if C is not None: return C
        return 0
    elif predictor == AVG_XY:
        if A is not None and B is not None: return (A + B) / 2
        return 0
    elif predictor == AVG_XZ:
        if A is not None and C is not None: return (A + C) / 2
        return 0
    elif predictor == AVG_YZ:
        if B is not None and C is not None: return (B + C) / 2
        return 0
    elif predictor == MED_XY:
        if A is not None and B is not None and C is not None:
            return median([A, B, A + B - C])
        return 0
    elif predictor == MED_XZ:
        if A is not None and C is not None and B is not None:
            return median([A, C, A + C - B])
        return 0
    elif predictor == MED_YZ:
        if B is not None and C is not None and A is not None:
            return median([B, C, B + C - A])
        return 0
    elif predictor == AVG3D:
        if A is not None and B is not None and C is not None:
            return (A + B + C) / 3
        return 0
    else:
        return 0

def calculate_memoryless_entropy(errors):
    """
    Calculate H = -sum(p_i * log2(p_i))
    """
    from collections import Counter
    import math
    
    if not errors:
        return 0
    
    M = len(errors)
    counts = Counter(errors)
    entropy = 0.0
    
    for error, count in counts.items():
        p_i = count / M
        if p_i > 0:
            entropy -= p_i * math.log2(p_i)
            
    return entropy

def median(values):
    """Helper for median calculation."""
    sorted_vals = sorted(values)
    n = len(sorted_vals)
    if n % 2 == 1:
        return sorted_vals[n // 2]
    else:
        return (sorted_vals[n // 2 - 1] + sorted_vals[n // 2]) / 2
```

## Subtopics

### [Adaptive Transform Construction Heuristic](11258-adaptive-transform-construction-heuristic.md)
A greedy heuristic for image-adaptive selection of RDLS filters for each subband and predictors for final subbands to minimize bitrate.

## Operating characteristics
*   **Time Complexity:** The method involves a 3-level 3D DWT, which is $O(N)$ where $N$ is the number of voxels. The predictor selection phase requires computing entropy for multiple predictors across all subbands, which adds a constant factor overhead proportional to the number of predictors and subbands. Entropy encoding is typically $O(N)$ or $O(N \log N)$ depending on the coder.
*   **Memory Complexity:** Requires storage for the transformed subbands and temporary buffers for prediction errors. The RDLS-SS-DWT can be implemented in-place or with minimal auxiliary storage.
*   **Measured Performance:** In experiments on medical images (CT, MRI, US), RDLS-SS-DWT+Pred showed bitrate improvements over standard DWT. For example, on MRI1, it achieved an 8.973% bitrate reduction compared to DWT [S1]. The improvement over SS-DWT+Pred was very small, suggesting RDLS steps may be optional [S1].
*   **Strengths:** Effective for volumetric data with smooth regions; leverages spatial correlation in the transform domain; adaptive predictor selection minimizes entropy.
*   **Limitations:** Heuristic selection can be computationally expensive if based on actual bitrate; prediction accuracy depends on data coherence; boundary handling requires fallbacks.
*   **When to use:** Lossless compression of volumetric medical images or other 3D data with spatial coherence.
*   **When not to use:** Real-time applications where heuristic selection overhead is prohibitive; data with high-frequency noise or lack of spatial correlation.

## Background
The method was proposed in 2020 by researchers from Vrije Universiteit Brussel (VUB) and Interdisciplinary Institute for BroadBand Technology (IBBT) [S1]. It builds on previous work on 2D image compression using hybrid transforms and extends it to 3D volumetric data [S1]. The research was presented in a conference report prior to the journal publication [S1]. The implementation is available as a patch to IRIS-JP3D [S1].

## Sources
[S1] Employing New Hybrid Adaptive Wavelet-Based Transform and Histogram Packing to Improve JP3D Compression of Volumetric Medical Images, https://doi.org/10.3390/e22121385, fetched 2026-10-26
