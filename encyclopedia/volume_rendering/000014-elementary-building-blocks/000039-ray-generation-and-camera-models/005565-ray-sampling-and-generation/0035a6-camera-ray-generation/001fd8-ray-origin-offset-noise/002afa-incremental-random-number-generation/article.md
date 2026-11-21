# Incremental Random Number Generation
Also called: Golden Ratio Random, Quasi-Monte Carlo Random

## Overview
Incremental Random Number Generation is a method for producing pseudo-random numbers with low discrepancy properties by iteratively adding a constant irrational number to the previous value and taking the fractional part. It solves the problem of generating uniformly distributed samples for ray tracing origins or directions while avoiding the computational overhead of hash functions or lookup tables. It differs from [Hash-based Pseudo-Random Number Generator](2896-hash-based-pseudo-random-number-generator.md) by relying on arithmetic progression rather than bitwise mixing, offering a stateless, memory-free approach suitable for GPU fragment shaders or compute kernels where texture fetches are expensive.

## How it works
**Inputs and Outputs:**
- Input: A scalar or vector `rand` representing the current random state/value.
- Output: A scalar or vector `rand_next` representing the next value in the sequence.

**Data Structures:**
No persistent data structures are required. The state is implicit in the previous output value.

**Parameters:**
- `GOLDEN_RATIO`: The golden ratio constant, approximately $0.6180339887$.
- `PI`: The mathematical constant $\pi$, approximately $3.1415926538$.
- `SCALE_1D`: A multiplier for the 1D case, typically $4.0$ in common implementations to spread the sequence across the unit interval [S1, unofficial].

**Formulas:**
The core operation is the fractional part function $\{ x \} = x - \lfloor x \rfloor$.

1. **1D Incremental Random:**
   $$ r_{i+1} = \{ r_i + 4 \phi \} $$
   where $\phi$ is the golden ratio.

2. **2D Incremental Random:**
   $$ r_{i+1} = \{ r_i + (\phi, \pi) \} $$
   where $r_i$ is a 2D vector $(x_i, y_i)$.

**Steps:**
1. Initialize the starting value $r_0$. A common choice is $0.0$ or a seed derived from pixel coordinates.
2. For each subsequent sample $i$, compute the next value using the appropriate formula (1D or 2D).
3. Use the resulting value(s) for sampling.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
GOLDEN_RATIO = 0.6180339887
PI = 3.1415926538

def fract(x):
    """Returns the fractional part of x."""
    return x - floor(x)

def incremental_rand_1d(rand: float) -> float:
    """
    Generates the next 1D random number.
    :param rand: Current random value.
    :return: Next random value.
    """
    return fract(rand + GOLDEN_RATIO * 4.0)

def incremental_rand_2d(rand: vec2) -> vec2:
    """
    Generates the next 2D random number.
    :param rand: Current 2D random vector.
    :return: Next 2D random vector.
    """
    return fract(rand + vec2(GOLDEN_RATIO, PI))
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision limits can cause the sequence to cycle or exhibit patterns if the same value is reused without incrementing. Ensure the input changes every call.
- **Correlation:** While low discrepancy, simple linear congruential generators can show correlations in higher dimensions. The 2D variant uses $\pi$ to mitigate this, but for strict quasi-Monte Carlo requirements, more complex sequences may be needed.
- **Initialization:** The starting value $r_0$ affects the phase of the sequence. Using a fixed seed may lead to visible patterns if the scene geometry aligns with the sequence period.

## Subtopics

### [Blue Noise Jitter Generator](5093-blue-noise-jitter-generator.md)
The Blue Noise Jitter Generator produces temporally coherent, spatially blue-noise-distributed offsets for ray origins or sampling positions. It combines a low-discrepancy spatial lookup table with a golden-ratio-based temporal shift and a Halton sequence for subpixel jitter, ensuring that jitter values change smoothly across frames while maintaining blue-noise spectral properties. It differs from [Hammersley Sampling](8569-hammersley-sampling.md) by explicitly decoupling spatial structure (via a precomputed table) from temporal evolution (via modular golden-ratio shifts), which reduces temporal aliasing artifacts in animated scenes.

### [Hammersley Sampling](8569-hammersley-sampling.md)
Hammersley Sampling generates 2D low-discrepancy samples by combining a linear sequence for one coordinate with a radical inverse in base 2 for the other. It differs from its parent [Incremental Random Number Generation](4898-incremental-random-number-generation.md) by providing structured 2D stratification rather than a 1D stream, and from [Rotated Halton Sampling](8922-rotated-halton-sampling.md) by using base 2 specifically for the second dimension, which simplifies implementation at the cost of potential correlation in higher dimensions.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. Only basic arithmetic operations (addition, multiplication, floor) are required.
- **Memory Complexity:** $O(1)$. No additional memory is needed beyond the current state.
- **Performance:** Extremely fast on GPUs due to lack of memory access. Suitable for real-time rendering.
- **Strengths:** Low overhead, easy to implement, good distribution properties for 1D and 2D.
- **Limitations:** Not suitable for high-dimensional sampling without modification. Potential for visible patterns if not initialized properly.
- **When to use:** When performance is critical and 1D/2D sampling is sufficient.
- **When not to use:** When high-dimensional stratification or strict blue noise properties are required.

## Background
Incremental Random Number Generation is a simple form of quasi-random number generation. It leverages the properties of irrational numbers to ensure that the generated sequence does not repeat and covers the unit interval uniformly. The use of the golden ratio is particularly effective because it is the "most irrational" number, minimizing the likelihood of resonance with integer multiples. This method is often used in ray tracing for generating random directions or offsets, as seen in various voxel ray tracing demos [S1, unofficial].

## Sources
[S1] newDINO/voxel_ray_demo, https://github.com/newDINO/voxel_ray_demo, fetched 2026-10-24 (unofficial)
