# JPEG-LS
Also called: Lossless or Near-Lossless JPEG, JLS, JPEG-LS Standard
## Overview
JPEG-LS is a lossless or near-lossless image compression standard designed to achieve high compression ratios with low computational complexity, serving as an alternative to the computationally intensive JPEG2000 [S1]. It distinguishes itself from sibling methods like [Lorenz Predictor with Arithmetic Coding](8070-lorenz-predictor-with-arithmetic-coding.md) by employing a specific adaptive context modeling and Golomb-Rice entropy coding scheme that avoids the need for complex transform stages, making it particularly effective for medical data where wavelet-based methods may lack efficiency [S1].
## How it works
**Inputs and Outputs:**
Input: A 2D or 3D array of pixel/voxel values $V$.
Output: A compressed bitstream representing the residuals.

**Data Structures:**
The method relies on a context model for each pixel/voxel. The context is determined by the local gradient of the causal neighborhood.
Key parameters:
- $N$: Number of context bins (typically 256 or 512).
- $M$: Maximum value for the adaptation state (typically 32 or 64).
- $\sigma$: Quantization step for near-lossless mode (default 0 for lossless).

**Prediction:**
For a current sample $V$, the predicted value $V_{pred}$ is derived from causal neighbors. JPEG-LS defines seven prediction modes. The encoder selects the mode that minimizes the residual energy for the current block or context [S1]. The prediction is based on the local gradient direction. If the local gradient is small, the prediction is a weighted average of neighbors; otherwise, it follows the gradient direction.

**Residual Calculation:**
The residual is $R = V - V_{pred}$. In near-lossless mode, $R$ is quantized: $R_q = \text{round}(R / \sigma) \cdot \sigma$.

**Entropy Coding:**
The residuals are encoded using context-based Golomb-Rice coding. The context index $c$ is mapped to a parameter $k_c$ for the Golomb-Rice code. The parameter $k_c$ is adapted based on the history of residuals in that context. The coding uses a simple arithmetic-like scheme but specifically Golomb-Rice for efficiency.

**Steps:**
1. For each voxel/pixel $V$ in raster order:
2. Determine the causal neighborhood.
3. Calculate the local gradient from neighbors.
4. Select one of the seven prediction modes based on the gradient.
5. Compute $V_{pred}$ using the selected mode.
6. Compute residual $R = V - V_{pred}$.
7. If near-lossless, quantize $R$ to $R_q$.
8. Map the context of the current voxel to a context index $c$.
9. Encode $R_q$ using Golomb-Rice coding with parameter $k_c$ derived from context $c$.
10. Update the adaptation state for context $c$ based on $R_q$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jpeg_ls_encode(volume):
    # Initialize context states
    # Each context has a parameter k and a state variable
    contexts = initialize_contexts(num_contexts=256)
    
    output_bits = []
    
    for v in volume:
        # Get causal neighbors
        neighbors = get_causal_neighbors(v)
        
        # Calculate local gradient
        gradient = calculate_gradient(neighbors)
        
        # Select prediction mode (7 modes defined in standard)
        # Mode selection logic depends on gradient magnitude and direction
        mode = select_prediction_mode(gradient)
        
        # Predict value
        v_pred = predict_value(neighbors, mode)
        
        # Compute residual
        residual = v - v_pred
        
        # Near-lossless quantization (if enabled)
        if near_lossless_enabled:
            residual = round(residual / sigma) * sigma
            
        # Determine context index
        # Context is based on local gradient characteristics
        context_index = map_gradient_to_context(gradient)
        
        # Get Golomb-Rice parameter for this context
        k = contexts[context_index].k
        
        # Encode residual using Golomb-Rice
        # Golomb-Rice encodes integer x with parameter k
        bits = golomb_rice_encode(residual, k)
        output_bits.extend(bits)
        
        # Update context state
        update_context(contexts[context_index], residual)
        
    return output_bits
```

**Edge Cases:**
- Boundary pixels: Use available neighbors only.
- Degenerate gradients: Fallback to default prediction mode.
- Precision: Integer arithmetic is used to avoid floating-point errors.

## Subtopics

### [LSE-3D](22894-lse-3d.md)
LSE-3D is a lossless compression scheme for volumetric images that uses 3D adaptive least-square linear prediction for decorrelation and context-based Golomb-Rice entropy coding for residuals. It is designed to be simple but efficient with low computational requirements, specifically for volumetric medical datasets. Unlike wavelet-based schemes such as 3D-SPIHT and 3D QT-L, LSE-3D does not support resolution and quality scalability or progressive coding, but achieves similar compression efficiency with significantly lower computational cost (up to 50% less floating-point operations per voxel).

## Operating characteristics
JPEG-LS provides the best compression for ANGIO data among tested methods, but lacks quality scalability and simultaneous lossy/lossless support in the same codestream [S1]. It is computationally efficient, with lower complexity than JPEG2000. The coding gain of 3D over 2D systems depends on the correlation and smoothness along the axis, with LPL mode leading to rate savings of about 16% for DSR and 33% for MR-MRI over JPEG2000 [S1]. For ANGIO, LPL method provides an improvement of about 5% over JPEG2000 [S1].

## Background
JPEG-LS is an older JPEG standard variant tested against 3-D/2-D MLZC, featuring seven prediction modes [S1]. It was developed to provide a low-complexity alternative to JPEG2000 for lossless and near-lossless compression.

## Sources
[S1] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
