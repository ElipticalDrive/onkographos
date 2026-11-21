# Lifting-Based Temporal Wavelet Transform

Also called: Memory-Constrained Lifting-Based Temporal Transform

## Overview
This method computes 1-D temporal wavelet transforms on long image sequences without buffering all slices, thereby avoiding artificial Group of Pictures (GOP) boundaries and the severe quality degradations associated with lossy decoding at those boundaries [S1]. It sets itself apart from standard block-based temporal transforms by using a finite buffer to continuously process the sequence, creating the effect of infinite memory through an overlap-save-like mechanism [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of image slices along the temporal dimension [S1].
*   **Output:** Temporal wavelet coefficients (low-pass approximation and high-pass detail subbands) [S1].

**Data Structures:**
*   **Buffer:** A finite buffer storing intermediate coefficients across lifting steps [S1]. The buffer layout holds the low-pass and high-pass coefficients generated at each step [S1].
*   **Coefficients:** Output arrays for approximation and detail coefficients at each decomposition level [S1].

**Parameters:**
*   **Filters:** Daubechies 9/7 biorthogonal filters are specified [S1]. Shorter filters may be substituted to reduce memory requirements [S1].
*   **Decomposition Level ($L$):** The number of recursive decomposition levels applied to the temporal dimension [S1].

**Algorithm Steps:**
The implementation uses a standard five-step lifting process derived from the dual polyphase matrix factorization of the Daubechies 9/7 filters [S1].

1.  **Factorization:** The transform is defined by the following dual polyphase matrix factorization [S1]:
    $$
    \begin{bmatrix} H_0(z) & H_1(z) \end{bmatrix} = \begin{bmatrix} 1 & 0 \end{bmatrix} \begin{bmatrix} 1 & L_1(z) \end{bmatrix} \begin{bmatrix} 1 & 0 \end{bmatrix} \begin{bmatrix} 1 & L_2(z) \end{bmatrix} \begin{bmatrix} 1 & 0 \end{bmatrix} \begin{bmatrix} 1 & L_3(z) \end{bmatrix} \begin{bmatrix} 1 & 0 \end{bmatrix} \begin{bmatrix} 1 & L_4(z) \end{bmatrix} \begin{bmatrix} s_0 & 0 \end{bmatrix}
    $$
    where $L_1(z)$ through $L_4(z)$ are the lifting factors and $s_0$ is a scaling factor [S1].

2.  **Lifting Execution:** The algorithm executes the five lifting steps sequentially [S1]. Each output coefficient is related to at most the next four samples in the sequence [S1]. This implies that when current and past samples are available, only four future samples are needed to compute the current output [S1].

3.  **Buffering Strategy:** A finite buffer continuously processes one part of the sequence at a time [S1]. Coefficients at intermediate lifting steps are buffered toward the end of a GOP [S1]. Processing finishes until intermediate coefficients from the beginning of the next GOP are available [S1]. This seamless continuation avoids boundary effects [S1].

4.  **Scaling:** Scaling is applied to make the transform approximately unitary [S1]. A simple approach applies scaling by bit shifts, requiring each scaling factor to be a power of two [S1].

**Buffer Size Calculation:**
The minimal number of images $N_L$ to buffer for an $L$-level wavelet decomposition is calculated recursively [S1]:
$$
N_L = \lceil N_{L-1}/2 \rceil + 1
$$
with base case:
$$
N_0 = 1
$$
[S1] The source also notes a minimum buffer of five samples (current plus four future samples) is required for a one-level decomposition [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_min_buffer_size(levels):
    """Calculates N_L = ceil(N_{L-1}/2) + 1 with N_0 = 1."""
    n = 1
    for _ in range(levels):
        n = (n + 1) // 2 + 1
    return n

def lifting_based_temporal_transform(sequence, levels):
    """
    Computes the lifting-based temporal wavelet transform on a sequence of slices.
    Uses a finite buffer to process long sequences without artificial GOP boundaries.
    """
    min_buf = compute_min_buffer_size(levels)
    # Buffer stores intermediate coefficients across lifting steps
    buffer = [0.0] * min_buf
    coefficients = []

    for i in range(len(sequence)):
        # Overlap-save-like continuous processing
        buffer.pop(0)
        buffer.append(sequence[i])

        # Execute five lifting steps on the current buffer window
        # The source specifies the structure but does not provide the exact arithmetic for L1-L4
        # low_pass = buffer[0::2]
        # high_pass = buffer[1::2]
        # high_pass += L1(low_pass)
        # low_pass += L2(high_pass)
        # high_pass += L3(low_pass)
        # low_pass += L4(high_pass)
        # low_pass *= s0
        # coefficients.append((low_pass, high_pass))
        pass

    return coefficients
```

## Operating characteristics
*   **Memory:** Requires a finite buffer whose size grows with the decomposition level $L$ according to $N_L = \lceil N_{L-1}/2 \rceil + 1$ [S1]. Shorter filters require less memory [S1].
*   **Complexity:** The lifting scheme is computationally efficient, involving only additions and multiplications by lifting factors [S1].
*   **Quality:** Avoids severe quality degradations at GOP boundaries that occur with independent coding of GOPs [S1].
*   **Limitations:** The method is designed for 1-D temporal transforms [S1]. Extending to 3-D requires careful handling of the third dimension, as scaling factors for 3-D dyadic integer wavelet transforms may not be powers of two, complicating the simple bit-shift scaling approach [S1].

## Background
This method was proposed to address boundary effects in lossy decoding of medical volumetric data [S1]. It builds on the lifting scheme for wavelet transforms, which allows for integer-to-integer transforms and in-place computation [S1]. The specific implementation for memory-constrained temporal transforms was described in [S1], referencing prior work on lifting-based FIR filter banks [S1] and memory-constrained algorithms for long sequences [S1]. The Daubechies 9/7 biorthogonal filters are commonly used in JPEG 2000 and are chosen for their good approximation properties [S1].

## Sources
[S1] Lossy-to-lossless compression of medical volumetric data using three-dimensional integer wavelet transforms, https://doi.org/10.1109/tmi.2003.809585, fetched 2026-11-01
