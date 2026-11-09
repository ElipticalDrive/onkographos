# Ray Origin Offset Noise

Also called: Noise Generation for Ray Offset, Ray Offset Noise

## Overview
Ray Origin Offset Noise is a technique used in volumetric ray casting to perturb the starting position of a ray within a voxel or grid cell, thereby avoiding banding artifacts caused by regular sampling patterns [S1, unofficial]. It differs from its sibling [Orthographic (Parallel) Camera Ray Generation](1319-orthographic-parallel-camera-ray-generation.md) by adding stochastic variation to the ray origin rather than strictly adhering to pixel centers, and from [Sample Streaming Payload Ray Computation](1509-sample-streaming-payload-ray-computation.md) by focusing on initialization jitter rather than payload management during traversal.

## How it works
**Inputs:**
- `uv`: A 2D vector representing the normalized screen or grid coordinates of the current sample (e.g., pixel coordinates or ray index).
- `unitVector`: A constant 2D vector used to break spatial correlation.

**Outputs:**
- `r`: A pseudo-random floating-point value in the range $[0, 1)$.

**Data Structures:**
- No persistent data structures are required. The function is stateless and computes the random value on demand based on input coordinates.

**Steps:**
1. Compute the dot product of the input coordinates `uv` and the scaled `unitVector`.
2. Apply the sine function to the result.
3. Multiply by the constant `43758.5453`.
4. Extract the fractional part of the result to obtain the final random value.

**Formula:**
$$ r = \text{fract}(\sin(\text{dot}(uv, \text{unitVector} \times 79.30408)) \times 43758.5453) $$

Where:
- $\text{dot}(a, b)$ is the dot product of vectors $a$ and $b$.
- $\text{fract}(x) = x - \lfloor x \rfloor$.
- $\text{unitVector}$ is a constant 2D vector. The source uses a specific scaling factor `79.30408` applied to this vector before the dot product.

**Parameters:**
- `unitVector`: A 2D vector. The specific components are not detailed in the source, but the scaling factor `79.30408` is applied to it.
- `43758.5453`: A constant multiplier.
- `79.30408`: A constant scaling factor for the unit vector.

**Edge Cases and Pitfalls:**
- **Precision:** The use of sine and large constants can lead to precision issues on some hardware. Ensure that the intermediate calculations maintain sufficient precision to avoid pattern repetition or banding.
- **Determinism:** The method is deterministic for the same input `uv`, ensuring consistency across frames and devices [S1, unofficial].
- **Correlation:** The choice of `unitVector` and the scaling factor is critical to minimize spatial correlation between adjacent samples.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def ray_origin_offset_noise(uv: tuple, unit_vector: tuple) -> float:
    """
    Generates a pseudo-random value to offset the ray origin.
    
    Args:
        uv: A tuple (x, y) representing the 2D coordinates.
        unit_vector: A tuple (x, y) representing the unit vector.
        
    Returns:
        A float in the range [0, 1).
    """
    # Scale the unit vector
    scaled_x = unit_vector[0] * 79.30408
    scaled_y = unit_vector[1] * 79.30408
    
    # Compute dot product
    dot_product = uv[0] * scaled_x + uv[1] * scaled_y
    
    # Apply sine
    sin_val = math.sin(dot_product)
    
    # Multiply by constant
    multiplied = sin_val * 43758.5453
    
    # Extract fractional part
    # math.modf returns (fractional, integer)
    fractional_part, _ = math.modf(multiplied)
    
    # Ensure the result is in [0, 1)
    if fractional_part < 0:
        fractional_part += 1.0
        
    return fractional_part
```

## Subtopics

### [Hash-based Pseudo-Random Number Generator](2896-hash-based-pseudo-random-number-generator.md)
A hash-based pseudo-random number generator maps integer or floating-point coordinates to a deterministic pseudo-random value in the range $[0, 1)$ or $[-1, 1]$ using transcendental functions (typically sine) or multiplicative mixing. It generates independent values based on spatial coordinates, ensuring that the same coordinate always yields the same random value across different frames and devices. Unlike [Incremental Random Number Generation](4898-incremental-random-number-generation.md), which generates a sequence of correlated random numbers for a single ray, this method computes values on demand from input coordinates.

### [Incremental Random Number Generation](4898-incremental-random-number-generation.md)
Incremental Random Number Generation is a method for producing pseudo-random numbers with low discrepancy properties, suitable for sampling in ray tracing and volume rendering. It differs from its parent [Ray Origin Offset Noise](2118-ray-origin-offset-noise.md) by using a deterministic additive sequence based on irrational constants rather than a hash function, and from its sibling [Hash-based Pseudo-Random Number Generator](2896-hash-based-pseudo-random-number-generator.md) by avoiding bitwise operations in favor of floating-point arithmetic. This approach ensures uniform distribution over iterations while maintaining statelessness across samples if the initial seed is managed externally.

### Deterministic LCG Noise Generation
A Linear Congruential Generator (LCG) producing deterministic pseudo-random floats in the range [-1, 1) using Numerical Recipes constants. Used for procedural sound synthesis [S2, unofficial].
**Steps:**
1. Update state: Multiply current state by 6,364,136,223,846,793,005 and add 1,442,695,040,888,963,407 using wrapping arithmetic (mod 2^64).
2. Extract top 24 bits: Shift state right by 40 bits and cast to f32.
**Formula:**
$$ x_{n+1} = (x_n \times 6364136223846793005 + 1442695040888963407) \mod 2^{64} $$
$$ v = \left(\frac{x_n \gg 40}{8388608.0}\right) - 1.0 $$
**Parameters:**
- Multiplier: 6,364,136,223,846,793,005 (u64).
- Increment: 1,442,695,040,888,963,407 (u64).
- State type: u64.
- Output range: [-1.0, 1.0).

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample. The computation involves a few arithmetic operations and a sine function.
- **Memory Complexity:** $O(1)$. No additional memory is required beyond the input coordinates.
- **Performance:** The method is efficient and suitable for real-time applications. The sine function is the most expensive operation, but modern GPUs optimize this well.
- **Strengths:** Simple to implement, deterministic, and effective at reducing banding artifacts.
- **Limitations:** The quality of the random numbers depends on the choice of constants. Poorly chosen constants can lead to visible patterns.
- **When to use:** When banding artifacts are visible in volumetric rendering due to regular sampling patterns.
- **When not to use:** When high-quality, statistically independent random numbers are required for Monte Carlo integration, as this method is not designed for such purposes.

## Background
The technique is commonly used in volumetric ray casting to improve the visual quality of rendered volumes by breaking up regular sampling patterns [S1, unofficial]. It is a simple yet effective method for reducing aliasing and banding artifacts.

## Sources
[S1] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S2] GitHub - NCrashed/roxlap: Rust portable reimplementation of Ken ..., https://github.com/NCrashed/roxlap, fetched 2026-10-29 (unofficial)
