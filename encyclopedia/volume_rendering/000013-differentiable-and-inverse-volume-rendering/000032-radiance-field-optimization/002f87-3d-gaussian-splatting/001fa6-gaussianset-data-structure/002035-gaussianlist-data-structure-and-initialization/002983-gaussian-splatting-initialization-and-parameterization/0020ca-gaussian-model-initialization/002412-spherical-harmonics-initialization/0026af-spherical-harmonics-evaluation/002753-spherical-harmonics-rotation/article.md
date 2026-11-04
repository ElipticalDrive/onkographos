# Spherical Harmonics Rotation

Also called: Analytic SH Rotation, Analytic Spherical Harmonics Rotation, SH X-axis rotation tables

## Overview
Spherical Harmonics Rotation computes the new SH coefficients of a function after rotating the viewing coordinate system by $\pm \pi/2$ around the X-axis. It solves the problem of efficiently updating view-dependent color representations in 3D Gaussian Splatting when the Gaussian's orientation changes or when projecting from a local frame to a global frame. It sets itself apart from general rotation methods by using precomputed analytic tables for specific $\pm 90^\circ$ rotations, avoiding expensive matrix multiplications or numerical integration [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: An array of SH coefficients $P$ for a specific band $l$ (where $l \in \{1, 2, 3, 4\}$). The coefficients are stored in standard order $m = -l, \dots, l$.
- Output: A new array of coefficients $P'$ rotated by $+\pi/2$ or $-\pi/2$ around the X-axis.

**Data Structures:**
The rotation is implemented using precomputed constant arrays (tables) indexed by the band level $l$.
- `kAnalyticRotatePlusPiHalfAboutXTable`: Stores the transformation rules for $+\pi/2$ rotation.
- `kAnalyticRotateMinusPiHalfAboutXTable`: Stores the transformation rules for $-\pi/2$ rotation. This table is the transpose (inverse) of the plus table [S1, unofficial].
- Indexing: The tables are indexed by $l-1$ (i.e., band 1 is index 0, band 2 is index 1, etc.) [S1, unofficial].

**Formulas:**
Let $p_i$ denote the input coefficient at index $i$ (corresponding to $m = i - l$) and $p_i'$ denote the output coefficient. The source provides explicit linear combinations for bands $l=1$ to $l=4$ for the $+\pi/2$ rotation [S1, unofficial].

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
The following Python-like pseudocode implements the $+\pi/2$ rotation for bands 1-4. The $-\pi/2$ rotation uses the inverse transformation (transpose of the plus table).

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
        # s_i = p_i
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
    Note: The exact inverse formulas are not explicitly listed in the source,
    but the source states the minus table is the transpose of the plus table.
    For implementation, one would typically precompute the inverse matrix
    or derive it from the plus table properties.
    """
    # [G] The source states the minus table is the transpose of the plus table.
    # To implement this without the explicit inverse formulas, one would construct
    # the rotation matrix R_plus, transpose it to get R_minus, and apply it.
    # However, since the source only gives the forward formulas, we assume
    # the existence of a corresponding table or function for the inverse.
    # For a complete implementation from text alone, the inverse coefficients
    # must be derived or provided. Here we note the dependency.
    raise NotImplementedError("Inverse table formulas not explicitly provided in source; use transpose of plus table.")
```

**Parameters:**
- `band_l`: Must be 1, 2, 3, or 4. Higher bands are not covered by the analytic tables [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Band Limit:** The analytic tables only cover bands 1-4. For band 0, no rotation is needed (constant). For bands > 4, this method is not applicable [S1, unofficial].
- **Precision:** The coefficients involve square roots of rational numbers (e.g., $\sqrt{3}/2$, $\sqrt{15}/4$). Use standard floating-point precision (float32 or float64) as required by the rendering pipeline [S1, unofficial].
- **Indexing:** The input array must be ordered by $m$ from $-l$ to $l$. Misalignment of indices will produce incorrect results [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ per band, as it involves a fixed number of arithmetic operations (linear combinations) regardless of the number of Gaussians. The cost is dominated by the constant number of multiplications and additions defined in the tables.
- **Memory Complexity:** $O(1)$ auxiliary memory for the rotation logic. The precomputed tables are stored in `constexpr` arrays, consuming minimal static memory [S1, unofficial].
- **Performance:** Significantly faster than general rotation methods that require computing Wigner D-matrices or numerical integration, especially for the specific $\pm 90^\circ$ case.
- **Strengths:** Exact analytic solution for specific rotations; no iterative approximation; very low computational overhead.
- **Limitations:** Only supports rotations of $\pm \pi/2$ around the X-axis. General rotations require different methods. Only supports bands 1-4 [S1, unofficial].

## Background
The analytic rotation tables for $\pm \pi/2$ around the X-axis are implemented in the `spz` library by Niantic Labs and Adobe Inc. [S1, unofficial]. The method is part of the Gaussian Splatting initialization and parameterization pipeline, specifically used in conjunction with Spherical Harmonics Evaluation to handle view-dependent color updates efficiently [S1, unofficial].

## Sources
[S1] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
