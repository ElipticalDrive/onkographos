# Lifting-Based Temporal Wavelet Transform

Also called: Memory-Constrained Lifting-Based Temporal Transform

## Overview
This method computes 1-D temporal wavelet transforms on long image sequences without buffering all slices, thereby avoiding artificial Group of Pictures (GOP) boundaries and the severe quality degradations associated with lossy decoding at those boundaries [S1]. It sets itself apart from standard block-based temporal transforms by using a finite buffer to continuously process the sequence, creating the effect of infinite memory through an overlap-save-like mechanism [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of image slices (temporal dimension).
*   **Output:** Wavelet coefficients for the temporal dimension.

**Data Structures:**
*   **Buffer:** A finite buffer storing intermediate coefficients. The size depends on the decomposition level.
*   **Coefficients:** Low-pass (approximation) and high-pass (detail) coefficients generated at each lifting step.

**Parameters:**
*   **Filters:** Daubechies 9/7 biorthogonal filters are specified [S1]. Shorter filters may be used to reduce memory requirements, though specific alternative filter coefficients are not provided in the sources [S1].
*   **Decomposition Level ($L$):** The number of recursive levels.

**Algorithm Steps:**
The implementation uses a standard five-step lifting process derived from the dual polyphase matrix factorization of the Daubechies 9/7 filters [S1].

1.  **Factorization:** The transform is defined by the following dual polyphase matrix factorization [S1]:
    $$
    \begin{bmatrix} H_0(z) & H_1(z) \end{bmatrix} = \begin{bmatrix} 1 & 0 \end{bmatrix} \begin{bmatrix} 1 & L_1(z) \end{bmatrix} \begin{bmatrix} 1 & 0 \end{bmatrix} \begin{bmatrix} 1 & L_2(z) \end{bmatrix} \begin{bmatrix} 1 & 0 \end{bmatrix} \begin{bmatrix} 1 & L_3(z) \end{bmatrix} \begin{bmatrix} 1 & 0 \end{bmatrix} \begin{bmatrix} 1 & L_4(z) \end{bmatrix} \begin{bmatrix} s_0 & 0 \end{bmatrix}
    $$
    where $L_1(z)$ through $L_4(z)$ are the lifting factors and $s_0$ is a scaling factor [S1].

2.  **Lifting Execution:** Execute the five lifting steps. Each output coefficient is related to at most the next four samples in the sequence [S1]. This implies that when current and past samples are available, only four future samples are needed to compute the current output [S1].

3.  **Buffering Strategy:**
    *   Use a finite buffer to continuously process the sequence [S1].
    *   Buffer coefficients at intermediate lifting steps toward the end of a GOP [S1].
    *   Finish processing until intermediate coefficients from the beginning of the next GOP are available [S1]. This seamless continuation avoids boundary effects [S1].

4.  **Scaling:** Apply scaling to make the transform approximately unitary. A simple approach is to apply scaling by bit shifts, requiring scaling factors to be powers of two [S1].

**Buffer Size Calculation:**
The minimal number of images $N_L$ to buffer for an $L$-level wavelet decomposition is calculated recursively [S1]:
$$
N_L = \lceil N_{L-1}/2 \rceil + 1
$$
with base case:
$$
N_0 = 1
$$
[S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lifting_based_temporal_transform(sequence, levels, buffer_size):
    """
    Computes the lifting-based temporal wavelet transform on a sequence of slices.
    
    Args:
        sequence: List of image slices (1D arrays or 2D images flattened/processed per pixel).
        levels: Number of decomposition levels L.
        buffer_size: Size of the buffer, must be >= N_L calculated by get_min_buffer_size.
        
    Returns:
        Coefficients: List of lists of coefficients for each level.
    """
    # Calculate minimum required buffer size
    min_buffer = get_min_buffer_size(levels)
    if buffer_size < min_buffer:
        raise ValueError(f"Buffer size {buffer_size} is less than minimum required {min_buffer}")

    # Initialize buffer with zeros or appropriate padding
    # The buffer must hold intermediate coefficients for the lifting steps
    buffer = [0.0] * buffer_size
    
    # Initialize coefficients storage
    coefficients = []
    
    # Process sequence in chunks or continuously
    # Note: The source describes an overlap-save-like approach.
    # For implementation simplicity, we assume a sliding window over the sequence
    # that respects the buffer constraints.
    
    current_idx = 0
    while current_idx < len(sequence):
        # Determine the window of samples to process
        # Based on the 5-sample dependency for one level
        # For L levels, the dependency is more complex, handled by the buffer
        
        # Extract samples for the current step
        # This part requires careful handling of the "infinite memory" effect
        # by buffering intermediate results from the end of one GOP 
        # and continuing into the next.
        
        # Placeholder for the actual lifting step execution
        # which updates the buffer and produces coefficients
        # The source does not provide the explicit arithmetic for L1-L4
        # but specifies the structure is a 5-step lifting.
        
        # Update buffer and compute coefficients
        # ... (Implementation of the 5 lifting steps using Daubechies 9/7 factors) ...
        
        current_idx += 1
        
    return coefficients

def get_min_buffer_size(levels):
    """
    Calculates the minimal number of images to buffer for an L-level decomposition.
    N_L = ceil(N_{L-1} / 2) + 1, with N_0 = 1
    """
    n = 1
    for l in range(1, levels + 1):
        n = (n // 2) + 1 if n % 2 == 0 else (n + 1) // 2 + 1
        # Equivalent to ceil(n/2) + 1
        # Using integer arithmetic: ceil(a/b) = (a + b - 1) // b
        # n = (n + 2 - 1) // 2 + 1 ? No.
        # ceil(n/2) = (n + 1) // 2 for integer n
        # So n = (n + 1) // 2 + 1
        pass
    
    # Recalculate correctly
    n = 1
    for l in range(1, levels + 1):
        n = (n + 1) // 2 + 1
    return n
```

## Operating characteristics
*   **Memory:** Requires a finite buffer. The size grows with the decomposition level $L$ according to $N_L = \lceil N_{L-1}/2 \rceil + 1$ [S1]. Shorter filters require less memory [S1].
*   **Complexity:** The lifting scheme is computationally efficient, involving only additions and multiplications by lifting factors [S1].
*   **Quality:** Avoids severe quality degradations at GOP boundaries that occur with independent coding of GOPs [S1].
*   **Limitations:** The method is designed for 1-D temporal transforms. Extending to 3-D requires careful handling of the third dimension, as scaling factors for 3-D dyadic integer wavelet transforms may not be powers of two, complicating the simple bit-shift scaling approach [S1].

## Background
This method was proposed to address boundary effects in lossy decoding of medical volumetric data [S1]. It builds on the lifting scheme for wavelet transforms, which allows for integer-to-integer transforms and in-place computation [S1]. The specific implementation for memory-constrained temporal transforms was described in [S1], referencing prior work on lifting-based FIR filter banks [S1] and memory-constrained algorithms for long sequences [S1]. The Daubechies 9/7 biorthogonal filters are commonly used in JPEG 2000 and are chosen for their good approximation properties [S1].

## Sources
[S1] Lossy-to-lossless compression of medical volumetric data using three-dimensional integer wavelet transforms, https://doi.org/10.1109/tmi.2003.809585, fetched 2026-11-01
