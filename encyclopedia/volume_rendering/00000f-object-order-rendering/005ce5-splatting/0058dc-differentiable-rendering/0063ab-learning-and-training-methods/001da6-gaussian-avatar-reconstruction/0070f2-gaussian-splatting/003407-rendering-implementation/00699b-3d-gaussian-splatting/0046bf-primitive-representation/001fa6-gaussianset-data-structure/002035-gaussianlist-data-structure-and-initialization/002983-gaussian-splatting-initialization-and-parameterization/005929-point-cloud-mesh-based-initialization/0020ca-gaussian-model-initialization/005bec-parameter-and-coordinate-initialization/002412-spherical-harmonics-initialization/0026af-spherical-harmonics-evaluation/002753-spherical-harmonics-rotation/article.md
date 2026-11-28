# Spherical Harmonics Rotation

Also called: Analytic SH Rotation, Analytic Spherical Harmonics Rotation, SH X-axis rotation tables

## Overview
Spherical Harmonics Rotation computes the new SH coefficients of a function after rotating the viewing coordinate system by $\pm \pi/2$ around the X-axis [S1, unofficial]. It sets itself apart from general rotation methods by using precomputed analytic tables for specific $\pm 90^\circ$ rotations, avoiding expensive matrix multiplications or numerical integration [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An array of SH coefficients $P$ for a specific band $l \in \{1, 2, 3, 4\}$ [S1, unofficial].
- Output: A new array of coefficients $P'$ rotated by $+\pi/2$ or $-\pi/2$ around the X-axis [S1, unofficial].

**Data Structures:**
The rotation is implemented using precomputed constant arrays (tables) indexed by the band level $l$ [S1, unofficial].
- `kAnalyticRotatePlusPiHalfAboutXTable`: Stores the transformation rules for $+\pi/2$ rotation [S1, unofficial].
- `kAnalyticRotateMinusPiHalfAboutXTable`: Stores the transformation rules for $-\pi/2$ rotation [S1, unofficial]. This table is the transpose (inverse) of the plus table [S1, unofficial].
- Indexing: The tables are indexed by $l-1$ (i.e., band 1 is index 0, band 2 is index 1, etc.) [S1, unofficial].

**Steps:**
1. Select the appropriate rotation table based on the desired rotation direction ($+\pi/2$ or $-\pi/2$) [S1, unofficial].
2. Index the table using $l-1$ to retrieve the transformation rules for the target band [S1, unofficial].
3. Apply the linear combination rules to the input coefficient array $P$ to produce the output array $P'$ [S1, unofficial].

**Formulas:**
Let $p_i$ denote the input coefficient at index $i$ (corresponding to $m = i - l$) and $p_i'$ denote the output coefficient [S1, unofficial]. The source provides explicit linear combinations for bands $l=1$ to $l=4$ for the $+\pi/2$ rotation [S1, unofficial].

**Band $l=1$ (3 coefficients):**
$$
\begin{aligned}
p_0' &= p_1 \\
p_1' &= -p_0 \\
p_2' &= p_2
\end{aligned}
$$

**Band $l=2$ (5 coefficients):**
Let $s_i = p_i$.
$$
\begin{aligned}
p_0' &= s_3 \\
p_1' &= -s_1 \\
p_2' &= -0.5 s_2 - \frac{\sqrt{3}}{2} s_4 \\
p_3' &= -s_0 \\
p_4' &= -\frac{\sqrt{3}}{2} s_2 + 0.5 s_4
\end{aligned}
$$

**Band $l=3$ (7 coefficients):**
$$
\begin{aligned}
p_0' &= \sqrt{5/8}s_3 - \sqrt{3/8}s_5 \\
p_1' &= -s_1 \\
p_2' &= \sqrt{3/8}s_3 + \sqrt{5/8}s_5 \\
p_3' &= -\sqrt{5/8}s_0 - \sqrt{3/8}s_2 \\
p_4' &= -0.25 s_4 - \frac{\sqrt{15}}{4} s_6 \\
p_5' &= \sqrt{3/8}s_0 - \sqrt{5/8}s_2 \\
p_6' &= -\frac{\sqrt{15}}{4} s_4 + 0.25 s_6
\end{aligned}
$$

**Band $l=4$ (9 coefficients):**
$$
\begin{aligned}
p_0' &= -\frac{\sqrt{14}}{4}s_5 + \frac{\sqrt{2}}{4}s_7 \\
p_1' &= -0.75 s_1 + \frac{\sqrt{7}}{4}s_3 \\
p_2' &= -\frac{\sqrt{2}}{4}s_5 - \frac{\sqrt{14}}{4}s_7 \\
p_3' &= \frac{\sqrt{7}}{4}s_1 + 0.75 s_3 \\
p_4' &= \frac{3}{8}s_4 + \frac{\sqrt{5}}{4}s_6 + \frac{\sqrt{35}}{8}s_8 \\
p_5' &= \frac{\sqrt{14}}{4}s_0 + \frac{\sqrt{2}}{4}s_2 \\
p_6' &= \frac{\sqrt{5}}{4}s_4 + 0.5 s_6 - \frac{\sqrt{7}}{4}s_8 \\
p_7' &= -\frac{\sqrt{2}}{4}s_0 + \frac{\sqrt{14}}{4}s_2 \\
p_8' &= \frac{\sqrt{35}}{8}s_4 - \frac{\sqrt{7}}{4}s_6 + 0.125 s_8
\end{aligned}
$$

**Pseudocode:**
The following Python-like pseudocode implements the $+\pi/2$ rotation for bands 1-4 [S1, unofficial]. The $-\pi/2$ rotation uses the inverse transformation (transpose of the plus table) [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def rotate_sh_plus_pi_half_x(coeffs, band_l):
    """
    Rotates SH coefficients by +pi/2 around the X-axis.
    
    Args:
        coeffs: List of float coefficients for band l, length 2*l + 1.
                Indices correspond to m = -l, ..., l.
        band_l: Integer band level (1, 2, 3, or 4).
        
    Returns:
        List of rotated coefficients.
    """
    if band_l < 1 or band_l > 4:
        raise ValueError("Only bands 1-4 are supported by analytic tables.")
    
    n = 2 * band_l + 1
    p = coeffs
    p_new = [0.0] * n
    
    if band_l == 1:
        p_new[0] = p[1]
        p_new[1] = -p[0]
        p_new[2] = p[2]
        
    elif band_l == 2:
        p_new[0] = p[3]
        p_new[1] = -p[1]
        p_new[2] = -0.5 * p[2] - (math.sqrt(3) / 2) * p[4]
        p_new[3] = -p[0]
        p_new[4] = -(math.sqrt(3) / 2) * p[2] + 0.5 * p[4]
        
    elif band_l == 3:
        p_new[0] = math.sqrt(5/8) * p[3] - math.sqrt(3/8) * p[5]
        p_new[1] = -p[1]
        p_new[2] = math.sqrt(3/8) * p[3] + math.sqrt(5/8) * p[5]
        p_new[3] = -math.sqrt(5/8) * p[0] - math.sqrt(3/8) * p[2]
        p_new[4] = -0.25 * p[4] - (math.sqrt(15) / 4) * p[6]
        p_new[5] = math.sqrt(3/8) * p[0] - math.sqrt(5/8) * p[2]
        p_new[6] = -(math.sqrt(15) / 4) * p[4] + 0.25 * p[6]
        
    elif band_l == 4:
        p_new[0] = -(math.sqrt(14) / 4) * p[5] + (math.sqrt(2) / 4) * p[7]
        p_new[1] = -0.75 * p[1] + (math.sqrt(7) / 4) * p[3]
        p_new[2] = -(math.sqrt(2) / 4) * p[5] - (math.sqrt(14) / 4) * p[7]
        p_new[3] = (math.sqrt(7) / 4) * p[1] + 0.75 * p[3]
        p_new[4] = (3/8) * p[4] + (math.sqrt(5) / 4) * p[6] + (math.sqrt(35) / 8) * p[8]
        p_new[5] = (math.sqrt(14) / 4) * p[0] + (math.sqrt(2) / 4) * p[2]
        p_new[6] = (math.sqrt(5) / 4) * p[4] + 0.5 * p[6] - (math.sqrt(7) / 4) * p[8]
        p_new[7] = -(math.sqrt(2) / 4) * p[0] + (math.sqrt(14) / 4) * p[2]
        p_new[8] = (math.sqrt(35) / 8) * p[4] - (math.sqrt(7) / 4) * p[6] + 0.125 * p[8]
        
    return p_new

def rotate_sh_minus_pi_half_x(coeffs, band_l):
    """
    Rotates SH coefficients by -pi/2 around the X-axis.
    Uses the inverse (transpose) of the +pi/2 table.
    """
    # [S1, unofficial] The source states the minus table is the transpose (inverse) of the plus table.
    # Explicit coefficients for the minus table are not provided in the source.
    # Implementation requires constructing the plus table matrix, transposing it, and applying it.
    pass
```

**Parameters:**
- `band_l`: Must be 1, 2, 3, or 4 [S1, unofficial]. Higher bands are not covered by the analytic tables [S1, unofficial].
- Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Band Limit:** The analytic tables only cover bands 1-4 [S1, unofficial]. For band 0, no rotation is needed (constant) [G]. For bands > 4, this method is not applicable [S1, unofficial].
- **Precision:** The coefficients involve square roots and fractions that should be computed in double precision before casting to float to avoid accumulation errors [G].
- **Boundaries:** The method assumes coefficients are ordered by $m = -l, \dots, l$ [G].

## Subtopics

### [CoordinateConverter Composition](4720-coordinateconverter-composition.md)
A composite transformation pipeline used in the SPZ format to convert Gaussian splatting parameters between coordinate systems. It handles the rotation of SH coefficients alongside position and quaternion flips, solving the problem of maintaining visual consistency when the underlying 3D coordinate frame changes. It is distinct from simple affine transforms because it explicitly manages the coupling between geometric flips and SH band rotations.

## Operating characteristics
- **Time Complexity:** Not given in the sources [S1, unofficial].
- **Memory Complexity:** Not given in the sources [S1, unofficial].
- **Performance:** Not given in the sources [S1, unofficial].
- **Strengths:** Exact analytic solution for specific rotations; no iterative approximation; very low computational overhead [S1, unofficial].
- **Limitations:** Only supports rotations of $\pm \pi/2$ around the X-axis [S1, unofficial]. General rotations require different methods [G]. Only supports bands 1-4 [S1, unofficial].

## Background
The analytic rotation tables for $\pm \pi/2$ around the X-axis are implemented in the `spz` library by Niantic Labs and Adobe Inc. [S1, unofficial].

## Sources
[S1] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
