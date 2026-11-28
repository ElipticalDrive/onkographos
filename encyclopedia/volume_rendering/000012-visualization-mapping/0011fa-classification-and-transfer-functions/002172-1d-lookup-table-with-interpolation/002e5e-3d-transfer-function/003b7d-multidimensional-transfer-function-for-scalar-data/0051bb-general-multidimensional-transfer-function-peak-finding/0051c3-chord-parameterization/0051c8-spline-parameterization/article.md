# Spline Parameterization
Also called: Catmull-Rom Spline Parameterization, Cubic Hermite Spline Parameterization

## Overview
Spline parameterization improves upon piecewise linear (chord) parameterization by approximating the transfer function curve $\rho(f(t))$ using a cubic Hermite spline (specifically Catmull-Rom) formulation [S1]. It is designed to provide smoother adherence to the image $\rho(f(t))$ with fewer world-space samples, particularly when the underlying data is extremely noisy and samples are far apart [S1]. Unlike its parent [Chord Parameterization](14542-chord-parameterization.md), which assumes a linear path between samples, this method estimates tangents to create a curved interpolation [S1].

## How it works
**Inputs:**
- Four consecutive world-space samples from the volume data: $f_{-1}, f_0, f_1, f_2$ [S1].
- The chord length $L_f$ between the current segment endpoints $f_0$ and $f_1$ [S1].
- A step increment $ds$ for sampling along the parameterized segment [S1].

**Outputs:**
- A parameterized curve $F(s)$ for $s \in [0, 1]$ representing the segment between $f_0$ and $f_1$ [S1].
- The parameter $t$ corresponding to the peak density $\rho$ found along the curve [S1].

**Data Structures:**
- A 1D texture (lookup table) storing the precomputed Hermite basis coefficients $h_{ij}(s)$ for efficient GPU access [S1].

**Steps:**
1. Maintain a stencil of four world-space samples $f_{-1}, f_0, f_1, f_2$ [S1].
2. Precompute the cubic Hermite basis coefficients $h_{ij}(s)$ and store them in a 1D texture [S1].
3. Parameterize the segment between $f_0$ and $f_1$ using the chord length $L_f$ to define the parameter $s \in [0, 1]$ [S1].
4. Choose a suitable increment $ds$ for sampling along the curve [S1].
5. Evaluate the Catmull-Rom spline $F(s)$ for each $s$ to approximate the curve [S1].
6. Find the parameter $s_\rho$ corresponding to the maximum density $\rho(F(s))$ along the segment [S1].
7. Interpolate to find the final peak parameter $t_\rho$ using linear interpolation between the segment endpoints' $t$ values:
   $$ t_\rho = t_0 + \frac{s_\rho}{1} (t_1 - t_0) $$
   [S1]

**Formulas:**
The Catmull-Rom spline parameterization is defined as [S1]:
$$ F(s) = f_0 h_{00}(s) + (f_1 - f_{-1})h_{10}(s) + f_1 h_{01}(s) + (f_2 - f_0)h_{11}(s) $$
where $h_{ij}(s)$ are the cubic Hermite basis functions [S1].

**Parameters:**
- $ds$: The sampling increment along the curve. Value: "Not given in the sources." [S1].

**Edge Cases and Pitfalls:**
- The method is impractical for most 2D transfer functions due to the computational cost of maintaining a 4-point stencil and evaluating the spline [S1].
- Arc-length parameterization is noted as too costly; chord length is used as an imperfect but efficient metric [S1].
- Precision issues may arise if the data is not extremely noisy or if samples are too close, where simpler chordal parameterization with more samples would be more efficient [S1].

## Subtopics

### Catmull-Rom Spline Interpolation
A method to generate smooth 3D curves for yarn trajectories from discrete contact points (knots) using Catmull-Rom splines, ensuring local influence and no aliasing artifacts [S2]. It is chosen for its algorithmic simplicity, low computational cost (usually below 2-3 seconds for a million crossings), and good results with initial sampling [S2].

**Definition:**
Organizes sampling points $\{P_1 \dots P_n\}$ into sequential subsets containing two knots ($P_i, P_{i+1}$) and two extra samples ($P_{i-1}, P_{i+2}$) to estimate tangents [S2]. Each initial sample is shared in four overlapping subsets, where the sample changes its role (as a clue for the first knot tangent, the first knot, the second knot, and a clue for the second knot tangent) [S2]. This ensures the yarn trajectory is only influenced by simulation, not by interpolation side effects; if one control point is moved, it affects the curve locally [S2].

**Steps:**
1. Identify sequential subsets of points: $P_{i-1}, P_i, P_{i+1}, P_{i+2}$ [S2].
2. Estimate the tangent at each knot using the neighboring points [S2].
3. Construct the cubic curve segment between $P_i$ and $P_{i+1}$ using the matrix formulation [S2].

**Formulas:**
The tangent at each control point is defined as [S2]:
$$ T_i = \frac{P_{i+1} - P_{i-1}}{2} $$
$$ T_{i+1} = \frac{P_{i+2} - P_i}{2} $$
respectively [S2]. The curve segment is represented by the cubic matrix formulation [Fol96] (not explicitly detailed in the provided text beyond the tangent definition) [S2].

**When to choose:**
Useful for visualizing yarn trajectories in cloth simulation where contact points are located at non-uniform distances and capture points with maximum variability, preventing aliasing artifacts [S2].

## Operating characteristics
- **Time Complexity:** Higher than chord parameterization due to the cost of maintaining a 4-point stencil and evaluating the spline [S1].
- **Memory Complexity:** Requires storage for the 1D texture of precomputed coefficients $h_{ij}(s)$ [S1].
- **Measured Performance:** Not given in the sources for volume rendering. In the context of yarn visualization, computational cost is usually below 2-3 seconds for a million crossings [S2].
- **Strengths:** Improves adherence to the image $\rho(f(t))$, providing smoother results with fewer world-space samples [S1].
- **Limitations:** Impractical for most 2D transfer functions compared to simpler chordal parameterization with more samples [S1].
- **When to use:** When the data itself is extremely noisy and world-space samples are farther apart [S1].
- **When not to use:** For most standard 2D transfer functions where computational efficiency is prioritized over smoothness with sparse samples [S1].

## Background
Spline parameterization was introduced as a logical improvement over piecewise linear parameterization in the context of volume rendering with multidimensional peak finding [S1]. The specific Catmull-Rom formulation is based on the classic Catmull-Rom splines [CR74] [S2]. The technique is described in the context of finding peaks in transfer functions where arc-length parameterization is too costly [S1].

## Sources
[S1] Volume Rendering with Multidimensional Peak Finding, https://www.sci.utah.edu/~knolla/multipeak_pv.pdf, fetched 2026-10-29
[S2] GPU Visualization and Voxelization of Yarn-Level Cloth, https://doi.org/10.2312/ceig.20141115, fetched 2026-10-26
