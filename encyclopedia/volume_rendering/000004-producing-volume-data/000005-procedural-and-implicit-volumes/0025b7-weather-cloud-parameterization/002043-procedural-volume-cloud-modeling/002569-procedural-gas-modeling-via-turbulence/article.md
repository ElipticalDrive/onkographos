# Procedural Gas Modeling via Turbulence

Also called: basic_gas procedural gas modeling, turbulence-based gas density function

## Overview
Procedural Gas Modeling via Turbulence is a method for generating volumetric gas density by evaluating a turbulence function and mapping the result through a precomputed power table [S1, unofficial]. It solves the problem of efficiently computing realistic gas density fields for volume rendering by replacing expensive per-sample power function calls with a lookup table [S1, unofficial]. Unlike sibling methods that may rely on 3D textures or distance fields, this approach computes density analytically at any point in space using mathematical shaping functions [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `pnt` (point in solid/world space), `parms` (parameter array), `POW_TABLE_SIZE` (table size constant).
- Output: `density` (float, gas density value).

**Data Structures:**
- `pow_table`: A static array of floats of size `POW_TABLE_SIZE`.
- `parms`: An array of floats. `parms[1]` is the maximum density value (range 0.0 to 1.0). `parms[2]` is the exponent for the power function [S1, unofficial].

**Steps:**
1. **Precompute Lookup Table:** Calculate `pow_table` once per image. For `i` from `POW_TABLE_SIZE-1` down to `0`:
   $$ \text{pow\_table}[i] = \left( \frac{i}{\text{POW\_TABLE\_SIZE}-1} \times \text{parms}[1] \times 2.0 \right)^{\text{parms}[2]} $$
   [S1, unofficial]
2. **Evaluate Turbulence:** Compute `turb = fast_turbulence(pnt)` [S1, unofficial].
3. **Index Calculation:** Compute the integer index:
   $$ \text{idx} = \left\lfloor \text{turb} \times 0.5 \times (\text{POW\_TABLE\_SIZE}-1) \right\rfloor $$
   [S1, unofficial]
4. **Output Density:** Set `*density = pow_table[idx]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def basic_gas(pnt, parms, pow_table, pow_table_size):
    # Precompute table if not done (logic from source)
    # In a real implementation, this would be handled by a static flag or initialization step
    # calcd = 1; if calcd: ... calcd = 0; ...
    
    # Evaluate turbulence
    turb = fast_turbulence(pnt)
    
    # Compute density index
    # Note: Source uses (int) cast which truncates towards zero. 
    # Assuming turb is in [0, 1] or similar range where this maps correctly.
    idx = int(turb * 0.5 * (pow_table_size - 1))
    
    # Output density
    density = pow_table[idx]
    return density
```

**Parameters:**
- `parms[1]`: Maximum density value. Range: 0.0 to 1.0 [S1, unofficial].
- `parms[2]`: Exponent for the power function. Typical value for steam: 6.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
- The turbulence function must return values that map correctly to the table indices. The source implies `turb` is scaled such that `turb * 0.5 * (POW_TABLE_SIZE-1)` yields a valid index [S1, unofficial].
- Precision issues may arise if `turb` is outside the expected range, leading to out-of-bounds access or incorrect density [G].

## Subtopics

### Sine-shaped gas veins
A procedural shaping technique that applies a sine function to turbulence output to create vein-like structures in the gas density [S1, unofficial]. Unlike the power function which increases contrast and separates plumes, the sine function creates continuous vein-like patterns similar to marble texturing [S1, unofficial].
**Code:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
turb = (1.0 + sin(fast_turbulence(pnt) * M_PI * 5)) * 0.5
```
[S1, unofficial]

## Operating characteristics
**Time and Memory Complexity:**
- Memory: Uses a precomputed table of size 10,000 (for `POW_TABLE_SIZE=10000`) [S1, unofficial].
- Time: Replaces per-sample `pow()` calls with a table lookup. For a 640x480 image with 100 volume samples per pixel, this saves approximately 30,710,000 `pow()` function calls [S1, unofficial].

**Strengths and Limitations:**
- Strength: High performance due to table lookup optimization [S1, unofficial].
- Limitation: Limited to simple shaping functions (power, sine) unless extended [S1, unofficial].

**When to Use:**
- When rendering gases with simple, turbulent structures and performance is critical [S1, unofficial].

## Background
Originated in David Ebert's work on procedural modeling of gases, fluids, and textures [S1, unofficial]. The method uses a visual simulation of turbulent flow similar to Ken Perlin’s approach [S1, unofficial]. It is part of a broader framework for procedural volume cloud modeling [S1, unofficial].

## Sources
[S1] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
