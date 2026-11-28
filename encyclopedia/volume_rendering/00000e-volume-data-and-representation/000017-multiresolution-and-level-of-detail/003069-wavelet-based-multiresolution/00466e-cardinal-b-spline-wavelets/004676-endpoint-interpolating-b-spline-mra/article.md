# Endpoint-Interpolating B-spline MRA

Also called: B-spline MRA with multiple knots at endpoints, B-spline boundary handling via multiple knots, Multiple Knots Boundary Construction, Multiple knots method

## Overview
Endpoint-Interpolating B-spline MRA constructs a multiresolution analysis on the bounded interval $[0, 1]$ by introducing multiple knots at the endpoints of the knot sequence, thereby avoiding the artificial discontinuities caused by zero-padding or the global dependencies of periodization [S1, S2, unofficial]. It distinguishes itself from sibling methods by explicitly defining boundary basis functions through knot multiplicity, ensuring that the resulting basis spans $L^2[0, 1]$ while maintaining the local support and smoothness properties of cardinal B-splines [S2, unofficial].

## How it works
**Inputs:**
- B-spline order $j$ (integer, e.g., $j=4$ for cubic).
- Target resolution level $m$ (integer, typically $m \le 0$).
- Minimum resolution level $m_0$ such that $j < 2^{-m_0}$ for scaling functions and $(2j - 1) < 2^{-m_0}$ for wavelets [S1, unofficial].

**Outputs:**
- Sets of scaling functions $\mathbf{V}_m$ and wavelet functions $\mathbf{U}_m$ (detail space $W_m$) defined on $[0, 1]$.

**Data Structures:**
The basis functions are classified into three disjoint sets based on their support intersection with $[0, 1]$:
1.  **Left Boundary (LB):** Functions whose support includes 0.
2.  **Inner (I):** Functions whose support is strictly within $(0, 1)$.
3.  **Right Boundary (RB):** Functions whose support includes 1.

**Knot Sequence:**
Define the knot sequence $t_m^j = \{t_m^k\}_{k=-j+1}^{2^m+j-1}$ on $[0, 1]$ as [S2, unofficial]:
$$
t_m^k = \begin{cases} 
0 & \text{for } -j+1 \le k \le 0 \\
k 2^{-m} & \text{for } 1 \le k \le 2^m - 1 \\
1 & \text{for } 2^m \le k \le 2^m + j - 1 
\end{cases}
$$

**Index Sets:**
Compute the integer indices $p$ for the basis functions $\phi_{mp}^{[j]}$ and $\psi_{mp}^{[j]}$ [S1, unofficial]:
-   **LB Scaling/Wavelet Indices:** $S_m^{LB} = \{p \in \mathbf{Z} \mid -j + 1 \le p \le -1\}$
-   **Inner Scaling/Wavelet Indices:** $S_m^I = \{p \in \mathbf{Z} \mid 0 \le p \le 2^{-m} - j\}$
-   **RB Scaling/Wavelet Indices:** $S_m^{RB} = \{p \in \mathbf{Z} \mid 2^{-m} - j + 1 \le p \le 2^{-m} - 1\}$

**Basis Function Definitions:**
The scaling functions are defined using the cardinal B-spline $N^j(x)$ [S2, unofficial]:
$$
B_{mp}^j(x) = N^j(2^m x - p), \quad p \in S_m^{LB} \cup S_m^I \cup S_m^{RB}
$$
Note: For $p \in S_m^{LB}$, the function $B_{mp}^j$ contains a multiple knot at 0. For $p \in S_m^{RB}$, it contains a multiple knot at 1.

The supports are [S1, unofficial]:
$$
\text{supp}\left(\phi_{mp}^{[j]}\right) = [2^m p, 2^m(j+p)]
$$
$$
\text{supp}\left(\psi_{mp}^{[j]}\right) = [2^m p, 2^m(2j-1+p)]
$$

**Algorithm:**
1.  **Initialize** $m_0$ such that supports are disjoint at the coarsest level.
2.  **Compute Inner Functions:** Generate $\phi_{mp}^{[j]}$ and $\psi_{mp}^{[j]}$ for all $p \in S_m^I$ using the standard cardinal B-spline and wavelet formulas from [Cardinal B-spline wavelets](11756-cardinal-b-spline-wavelets.md).
3.  **Compute Left Boundary Functions:** Generate $\phi_{mp}^{[j]}$ and $\psi_{mp}^{[j]}$ for all $p \in S_m^{LB}$. These are constructed by evaluating the B-spline/wavelet definitions with the multiple knots at 0.
4.  **Generate Right Boundary Functions:** Use symmetry and reindexing of the LB functions to generate the RB functions for $p \in S_m^{RB}$. Specifically, map $p_{RB}$ to the corresponding $p_{LB}$ via reflection across the interval center [S1, unofficial].
5.  **Assemble Spaces:**
    $$
    \mathbf{V}_m = \mathbf{V}_m^{LB} \oplus \mathbf{V}_m^I \oplus \mathbf{V}_m^{RB}
    $$
    $$
    \mathbf{U}_m = \mathbf{U}_m^{LB} \oplus \mathbf{U}_m^I \oplus \mathbf{U}_m^{RB}
    $$
    such that $\mathbf{V}_m = \mathbf{V}_{m+1} \oplus \mathbf{U}_{m+1}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Precision:** At very high resolutions (large $m$), floating-point errors in knot placement $k 2^{-m}$ may cause support overlaps or gaps.
-   **Degenerate Input:** If $m$ is too large such that $2^{-m} < j$, the inner set $S_m^I$ becomes empty or invalid. The condition $j < 2^{-m_0}$ prevents this [S1, unofficial].
-   **Orthogonality:** The wavelets must be orthogonal to the scaling functions. This constrains the choice of wavelet coefficients $q_{j,k}$ for the boundary wavelets, which are not identical to the interior wavelets [S2, unofficial].

## Subtopics

### [Boundary Conditions for Wavelet Transform on Intervals](23726-boundary-conditions-for-wavelet-transform-on-intervals.md)
This method defines how to apply wavelet transforms, originally formulated for the infinite real line $\mathbb{R}$, to finite datasets confined to the interval $[0, 1]$, which is a prerequisite for volume rendering applications using bounded data. It solves the problem of artificial discontinuities and basis function count inflation that arise when naively truncating the signal. It distinguishes itself from sibling methods by offering specific extension strategies—Spatial Windowing, Reflection, Repetition, and Multiple Knots—each imposing different continuity and symmetry constraints on the extended function $\tilde{f}(x)$.

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Strengths:** Eliminates artificial discontinuities at boundaries without the global coupling of periodization [S2, unofficial]. Maintains local support, allowing for efficient sparse storage and fast decomposition/reconstruction [S1, unofficial].
-   **Limitations:** Requires pre-computation of special boundary basis functions [S1, unofficial]. The construction of orthogonal wavelets for the boundary spaces is non-trivial and depends on the specific order $j$ [S2, unofficial].
-   **When to use:** When rendering or processing bounded volume data where boundary artifacts from zero-padding are unacceptable and periodization is inappropriate (e.g., non-periodic physical phenomena) [S2, unofficial].

## Background
Endpoint-Interpolating B-spline MRA was introduced by Chui and Quak [14] (cited in [S2, unofficial]) to define B-spline wavelets intrinsically on the bounded interval $L^2[0, 1]$. Efficient decomposition and reconstruction algorithms based on these bases were proposed by [30] (cited in [S2, unofficial]). The method is part of a broader effort to adapt wavelet theory, originally for $\mathbb{R}$, to bounded domains for computer graphics applications like surface and volume representation [S2, unofficial].

## Sources
[S1] Ray–tracing of multiresolution B–Spline volumes, https://doi.org/10.3929/ethz-a-006651431, fetched 2026-10-26 (unofficial)
[S2] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
