# Difference encoding for time-variant volumes

Also called: Shen and Johnson difference encoding

## Overview
A method by Shen and Johnson using difference encoding of time-variant volumes to reduce storage and rendering time [S1, unofficial]. It addresses the problem of temporal redundancy in time-varying volumetric datasets by transforming the data in the temporal domain to expose redundancy for compression [S2, unofficial]. It sets itself apart from spatial predictive methods by operating on sequences of values at a single spatial location across time steps, rather than on spatial neighbors [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A time-variant volume dataset $D$ with spatial dimensions $(X, Y, Z)$ and $n$ time steps.
- Output: Compressed coefficients suitable for entropy coding.

**Data Structures:**
- The method processes data in the temporal domain. For each spatial voxel $(x, y, z)$, a sequence of values $D(i)$ is extracted for $i = 0 \dots n-1$.
- The output consists of transformed coefficients $TC(i)$ and $TL(i)$ for each sequence.

**Steps:**
1. Iterate over every spatial voxel $(x, y, z)$ in the volume.
2. Extract the temporal sequence $D(0), D(1), \dots, D(n-1)$ for that voxel.
3. Pass the first value $D(0)$ unchanged to the output.
4. Compute the first-order temporal difference $TC(i)$ for $i > 0$:
   $$ TC(i) = D(i) - D(i-1) $$
5. Compute the second-order temporal difference $TL(i)$ for $i > 1$:
   $$ TL(i) = TC(i) - TC(i-1) $$
   Note: The source formula (5.2) in [S2, unofficial] defines $TL(i)$ with a conditional for $i=1$, but the text describes "double differencing" which implies applying the difference transform to the coefficients resulting from the first difference transform. The standard double difference for $i>1$ is the difference of the first differences. The source explicitly states: "repeating the difference transform on the coefficients resulting from the first difference transform."
6. Encode the resulting coefficients ($D(0)$, $TC(i)$, $TL(i)$) using entropy coding (e.g., lead-1 encoding as suggested by the histogram properties [S2, unofficial]).

**Formulas:**
- First difference: $TC(i) = D(i) - D(i-1)$ for $0 < i < n$ [S2, unofficial].
- Second difference: $TL(i) = TC(i) - TC(i-1)$ for $1 < i < n$ [S2, unofficial].
- The source formula (5.2) in [S2, unofficial] is written as:
  $$ TL(i) = \begin{cases} TC(i) & i = 1 \\ T(i) - T(i-1) & 1 < i \end{cases} $$
  However, the text clarifies that $TC$ is the result of the first transform, and $TL$ is the result of applying the difference transform to $TC$. The notation $T(i)$ in the source likely refers to the first-order difference sequence (equivalent to $TC$). Thus, for $i>1$, $TL(i) = TC(i) - TC(i-1)$. The case $i=1$ is handled by passing $TC(1)$ unchanged or as defined by the specific implementation detail in the source. The source states "The first data value at i = 0 is passed along unchanged." It does not explicitly state how $TC(1)$ is handled in the second transform, but the conditional $i=1$ suggests $TL(1) = TC(1)$.

**Parameters:**
- Dataset dimensions: Example given is $256^3$ volume with 274 time steps [S2, unofficial].

**Edge Cases and Pitfalls:**
- The range of coefficient magnitudes may expand after transformation, although more coefficients cluster around zero, improving compressibility [S2, unofficial].
- Precision issues may arise if the difference values exceed the storage capacity of the encoding format, though the method aims to pull coefficients toward zero to mitigate this [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def difference_encoding_time_variant(volume):
    """
    volume: 4D array [x, y, z, t]
    Returns: list of transformed coefficients for each voxel
    """
    transformed_data = []
    num_time_steps = volume.shape[3]
    
    for x in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for z in range(volume.shape[2]):
                # Extract temporal sequence for this voxel
                D = [volume[x, y, z, t] for t in range(num_time_steps)]
                
                # Output list for this voxel
                voxel_coeffs = []
                
                # First value passed unchanged
                voxel_coeffs.append(D[0])
                
                # Compute first-order differences TC
                TC = []
                for i in range(1, num_time_steps):
                    tc_val = D[i] - D[i-1]
                    TC.append(tc_val)
                    voxel_coeffs.append(tc_val)
                
                # Compute second-order differences TL
                # For i > 1, TL(i) = TC(i) - TC(i-1)
                # The source formula (5.2) has a conditional for i=1.
                # Assuming TC is 0-indexed corresponding to time steps 1..n-1
                # TC[0] corresponds to i=1, TC[1] to i=2, etc.
                
                for i in range(1, len(TC)):
                    # i in loop is index in TC, corresponding to time step i+1
                    # TL(i+1) = TC(i+1) - TC(i)
                    tl_val = TC[i] - TC[i-1]
                    voxel_coeffs.append(tl_val)
                
                # Note: The source formula (5.2) defines TL(1) = TC(1).
                # In our 0-indexed TC list, TC[0] is TC(1).
                # The loop above starts from i=1 (which is TC(2)), so TC(1) is not added as TL.
                # We need to add TL(1) = TC(1) if the source implies it is part of the output sequence.
                # The source says "TL(i) = ... : 1 < i". This implies TL(1) is defined separately.
                # Let's re-read carefully: "TL(i) = TC(i) if i=1, else T(i) - T(i-1) for 1 < i".
                # This suggests TL(1) is included in the output.
                
                # Re-adjusting to match source formula exactly:
                # Output sequence: D(0), TC(1), TL(1), TC(2), TL(2), ...
                # But the source formula (5.2) is for TL.
                # Let's stick to the explicit formula provided.
                
                # Resetting to match source formula (5.2) precisely:
                voxel_coeffs_precise = []
                voxel_coeffs_precise.append(D[0]) # i=0
                
                # i=1: TC(1) = D(1) - D(0)
                tc_1 = D[1] - D[0]
                voxel_coeffs_precise.append(tc_1)
                
                # i=1: TL(1) = TC(1)
                tl_1 = tc_1
                voxel_coeffs_precise.append(tl_1)
                
                for i in range(2, num_time_steps):
                    tc_i = D[i] - D[i-1]
                    tl_i = tc_i - TC[i-2] # TC[i-2] is TC(i-1)
                    voxel_coeffs_precise.append(tc_i)
                    voxel_coeffs_precise.append(tl_i)
                    
                transformed_data.append(voxel_coeffs_precise)
                
    return transformed_data
```
*Note: The pseudocode above attempts to reconcile the text description of "double differencing" with the specific formula (5.2) in [S2, unofficial]. The formula (5.2) explicitly defines $TL(1) = TC(1)$. The text says "repeating the difference transform on the coefficients resulting from the first difference transform." This implies the output of the first transform ($TC$) is the input to the second. The standard double difference would be $TL(i) = TC(i) - TC(i-1)$. The source's conditional for $i=1$ is an edge case handling.*

## Subtopics

### Double Difference Transform
A transform applied to time-varying volumetric data to expose redundancy by computing differences between temporal samples, specifically using double differencing to pull coefficients toward zero [S2, unofficial]. It is chosen when data varies smoothly over time, as it creates a histogram with values clustered around zero, which compresses better [S2, unofficial]. It differs from simple delta encoding by applying the difference operation twice [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels and time steps, $O(X \cdot Y \cdot Z \cdot n)$, as each voxel's temporal sequence is processed independently.
- **Memory Complexity:** Low, as it can be processed voxel-by-voxel or in small chunks. The decoder must be fast and small to fit in cache [S2, unofficial].
- **Performance:** Effective for data that varies smoothly over time. The histogram of coefficients shows more values grouped around zero, leading to better compression ratios [S2, unofficial].
- **Strengths:** Reduces storage and rendering time by exploiting temporal coherence [S1, unofficial]. Simple to implement [S2, unofficial].
- **Limitations:** May expand the range of coefficient magnitudes, though entropy is reduced [S2, unofficial]. Not suitable for data with high temporal noise or abrupt changes without additional handling.

## Background
Originated from Shen and Johnson's work on difference encoding of time-variant volumes [S1, unofficial]. The method is discussed in the context of reducing storage and rendering time for time-varying volumetric data [S1, unofficial]. The specific double difference transform details are found in a report on length-limited data transformation and compression [S2, unofficial].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] Length-Limited Data Transformation and Compression, https://digital.library.unt.edu/ark:/67531/metadc874691/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
