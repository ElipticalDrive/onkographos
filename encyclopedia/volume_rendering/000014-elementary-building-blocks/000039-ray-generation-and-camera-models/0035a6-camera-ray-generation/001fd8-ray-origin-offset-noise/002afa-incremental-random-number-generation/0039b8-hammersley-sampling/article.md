# Hammersley Sampling
Also called: Hammersley Sequence
## Overview
Hammersley Sampling generates 2D low-discrepancy samples by combining a linear sequence for one coordinate with a radical inverse in base 2 for the other [S1, unofficial]. It differs from its parent [Incremental Random Number Generation](4898-incremental-random-number-generation.md) by providing structured 2D stratification rather than a 1D stream, and from [Rotated Halton Sampling](8922-rotated-halton-sampling.md) by using base 2 specifically for the second dimension, which simplifies implementation at the cost of potential correlation in higher dimensions [S1, unofficial].
## How it works
**Inputs:**
- $N$: Total number of samples (must be a perfect square, $N = A_{depth}^2$).
- Base offset: $(x_0, y_0)$, the base position for the sample grid.

**Outputs:**
- A set of $N$ 2D points $(x_k, y_k)$ distributed over the unit square relative to the base offset.

**Data Structures:**
- No persistent state is required beyond the index $k$. The method is stateless per sample, though often iterated sequentially.

**Steps:**
1. Calculate the depth $A_{depth} = \sqrt{N}$.
2. Iterate $k$ from $0$ to $N-1$.
3. Compute the first coordinate $x_k$ as the linear fraction:
   $$ x_k = \frac{k}{N} $$
4. Compute the second coordinate $y_k$ using the radical inverse in base 2, denoted $\phi_2(k)$:
   $$ y_k = \phi_2(k) $$
5. The final sample position is $(x_0 + x_k, y_0 + y_k)$.

**Radical Inverse in Base 2 ($\phi_2$):**
To compute $\phi_2(k)$:
1. Write $k$ in binary: $k = \sum_{i=0}^{m} b_i 2^i$, where $b_i \in \{0, 1\}$.
2. Reverse the bits after the binary point:
   $$ \phi_2(k) = \sum_{i=0}^{m} b_i 2^{-(i+1)} $$
   This is equivalent to bit-reversing the integer $k$ and dividing by $2^m$ (where $2^m$ is the smallest power of 2 greater than $k$).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def radical_inverse_base2(k: int) -> float:
    """
    Computes the radical inverse of k in base 2.
    """
    v = 0.0
    denom = 1.0
    n = k
    while n > 0:
        # Extract the least significant bit
        bit = n & 1
        # Add bit * 2^(-i-1) to the result
        v += bit * (denom / 2.0) # denom starts at 1, so first term is bit/2
        # Prepare for next bit: shift k right, update denominator
        n >>= 1
        denom *= 2.0
    return v

def hammersley_sample(k: int, N: int, x_offset: float, y_offset: float):
    """
    Generates a single Hammersley sample.
    k: index from 0 to N-1
    N: total number of samples (must be a perfect square)
    x_offset, y_offset: base coordinates
    """
    # Step 1: Linear coordinate
    x = k / N
    
    # Step 2: Radical inverse coordinate
    y = radical_inverse_base2(k)
    
    return (x_offset + x, y_offset + y)
```

**Parameters:**
- $N$: Typically a power of 4 or 16 for efficient bit manipulation, but any perfect square works.
- Base 2: Fixed for the second dimension.

**Edge Cases:**
- $N$ must be a perfect square to ensure uniform grid coverage in the linear dimension. If $N$ is not a square, the linear spacing $k/N$ does not align with the radical inverse structure optimally.
- Precision: Floating-point errors in bit reversal can occur for very large $k$ if not handled with integer arithmetic before conversion to float.

## Subtopics

### [Rotated Halton Sampling](8922-rotated-halton-sampling.md)
A stratified sampling technique that generates low-discrepancy Halton sequences for multiple dimensions while avoiding sequence correlation by rotating the sequence per stream and dimension. It falls back to white noise for high dimensions.

## Operating characteristics
**Time Complexity:** $O(\log N)$ per sample due to bit reversal for $\phi_2(k)$.
**Memory Complexity:** $O(1)$ per sample.
**Strengths:** Simple to implement; provides good 2D coverage; deterministic.
**Limitations:** Correlation between dimensions can occur for certain $N$; not easily extensible to $>2$ dimensions without modification (unlike Halton with different bases).
**When to use:** 2D stratified sampling in ray tracing or volume rendering where low discrepancy is needed and implementation simplicity is prioritized.
**When not to use:** High-dimensional sampling ($>2$D) or when strict independence between dimensions is required without rotation.

## Background
Originated from the Hammersley point set, a type of low-discrepancy sequence used in quasi-Monte Carlo methods. It is widely used in computer graphics for image sampling and path tracing to reduce variance compared to pure Monte Carlo. The specific implementation using base 2 for the radical inverse is common in real-time and offline renderers for its efficiency.

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
