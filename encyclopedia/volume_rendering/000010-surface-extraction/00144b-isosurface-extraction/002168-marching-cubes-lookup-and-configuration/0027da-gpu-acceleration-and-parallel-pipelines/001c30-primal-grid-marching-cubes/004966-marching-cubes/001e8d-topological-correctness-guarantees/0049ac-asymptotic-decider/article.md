# Asymptotic Decider

Also called: Nielson and Hamann decider, Nielson-Hamann Asymptotic Decider, Nielson-Hamann Face Disambiguation

## Overview
The Asymptotic Decider (AD) is a method for resolving topological ambiguities in Marching Cubes that arise from the trilinear interpolant's saddle points on cube faces [S1, S2, S4, S8]. It determines the correct connectivity of intersection points on ambiguous faces by comparing the isovalue to the saddle point value of the bilinear interpolant defined on that face [S8]. Unlike the standard lookup table which arbitrarily selects one of two valid topologies, AD uses the scalar field values to select the topology that best approximates the continuous isosurface, ensuring consistency across shared faces [S7, unofficial][S8]. It sets itself apart from sibling methods like [Topologically Correct Marching Cubes](1368-topologically-correct-marching-cubes.md) and [Marching Cubes with Hermite Data and Full Case Table](961-marching-cubes-with-hermite-data-and-full-case-table.md) by restricting its disambiguation to face-level bilinear tests rather than analyzing interior critical points or using higher-order Hermite data [S2][S5, unofficial].

## How it works
The Asymptotic Decider resolves the ambiguity present in specific Marching Cubes cases where a face has vertices with alternating signs (e.g., two diagonally opposite vertices are positive and the other two are negative) [S1][S7, unofficial]. In such cases, the standard algorithm cannot determine whether the surface passes through the face connecting the positive vertices or the negative vertices [S7, unofficial]. AD resolves this by computing a discriminant value based on the scalar values at the four corners of the ambiguous face [S7, unofficial][S8].

### Inputs and Outputs
*   **Input:**
    *   `I`: The isovalue (scalar threshold).
    *   `T_00, T_01, T_10, T_11`: The scalar field values at the four corners of the ambiguous face. The indices $(i,j)$ correspond to the grid coordinates $(0,0), (1,0), (0,1), (1,1)$ respectively.
*   **Output:**
    *   A boolean or topology indicator determining which pair of diagonally opposite edges is connected.

### Data Structures
No additional data structures are required beyond the scalar field values and the isovalue. The method is a pure function evaluation.

### Steps
1.  **Identify Ambiguous Faces:** A face is ambiguous if its four vertices have alternating signs. Specifically, if $(T_{00} > 0 \land T_{11} > 0 \land T_{01} < 0 \land T_{10} < 0)$ or $(T_{00} < 0 \land T_{11} < 0 \land T_{01} > 0 \land T_{10} > 0)$ [S7, unofficial].
2.  **Compute Discriminant $V$:** Calculate the value $V$ using the bilinear interpolation saddle point test [S4, S8]:
    $$ V = T_{00}T_{11} - T_{01}T_{10} - I(T_{00} + T_{11} - T_{01} - T_{10}) $$
    Here, $T_{ij}$ are the scalar values at the corners, and $I$ is the isovalue [S4].
3.  **Determine Topology:**
    *   If $V < 0$, the isocontour on the face is a rectangular hyperbola in the first and third quadrants (relative to the saddle point). This implies the surface connects the vertices $(T_{00}, T_{11})$ [S4].
    *   If $V > 0$, the isocontour is a rectangular hyperbola in the second and fourth quadrants. This implies the surface connects the vertices $(T_{01}, T_{10})$ [S4].
    *   If $V = 0$, the saddle point lies exactly on the isovalue. This is a degenerate case; typically, the standard topology or a fallback rule is used [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def asymptotic_decider(T_00, T_01, T_10, T_11, I):
    """
    Resolves face ambiguity for Marching Cubes.
    
    Args:
        T_00: Scalar value at corner (0,0)
        T_01: Scalar value at corner (0,1)
        T_10: Scalar value at corner (1,0)
        T_11: Scalar value at corner (1,1)
        I:    Isovalue
        
    Returns:
        True if the surface connects (T_00, T_11), False if it connects (T_01, T_10).
        Assumes the face is already identified as ambiguous (alternating signs).
    """
    # Compute the discriminant V
    # Formula: V = T00*T11 - T01*T10 - I*(T00 + T11 - T01 - T10)
    V = (T_00 * T_11) - (T_01 * T_10) - I * (T_00 + T_11 - T_01 - T_10)
    
    # If V < 0, the hyperbola is in the 1st/3rd quadrants -> connects T_00 and T_11
    # If V > 0, the hyperbola is in the 2nd/4th quadrants -> connects T_01 and T_10
    if V < 0:
        return True  # Connect (0,0) and (1,1)
    else:
        return False # Connect (0,1) and (1,0)
```

### Parameters
*   **Isovalue ($I$):** The threshold scalar value defining the surface.
*   **Corner Values ($T_{ij}$):** The scalar field samples at the grid vertices.

### Edge Cases and Pitfalls
*   **Precision:** Floating-point precision errors can cause $V$ to be slightly non-zero when it should be zero, or vice versa. Robust implementations should handle $V \approx 0$ carefully, potentially by checking the sign of the gradient or using a small epsilon tolerance [G].
*   **Consistency:** The test must be applied consistently to shared faces. Since the test depends only on the four corner values and the isovalue, it is inherently consistent across adjacent voxels sharing that face [S8].
*   **Interior Ambiguity:** The Asymptotic Decider only resolves face ambiguities. It does not resolve interior ambiguities (saddle points inside the cube) [S2][S5, unofficial]. Methods like [Topologically Correct Marching Cubes](1368-topologically-correct-marching-cubes.md) are required for full topological correctness in all cases [S2].

## Subtopics

### [Ambiguous Face Triangulation for Regular Grid](10105-ambiguous-face-triangulation-for-regular-grid.md)
Ambiguous Face Triangulation is a method for resolving topological ambiguities in isosurface extraction on regular grids when a single grid face has sign changes along all four of its edges, resulting in four intersection vertices on that face. It sets itself apart from the parent [Asymptotic Decider](790-asymptotic-decider.md) by using a geometric connectivity rule based on gradient orientation rather than a scalar interpolation test to determine the triangulation, ensuring that adjacent cells fit together topologically.

### Lopes and Brodlie Extension
An extension of Natarajan's tests to all ambiguous cases of Marching Cubes. Proposed by Lopes and Brodlie in 2003 [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per ambiguous face. The test involves a constant number of arithmetic operations.
*   **Memory Complexity:** $O(1)$ additional memory. No extra data structures are needed.
*   **Performance:** The test is computationally inexpensive, adding minimal overhead to the Marching Cubes algorithm [S6, unofficial].
*   **Strengths:** Simple to implement; resolves the most common source of topological errors (face ambiguities); ensures consistency across shared faces [S8].
*   **Limitations:** Does not resolve interior ambiguities; requires scalar field values (not just binary signs); adds complexity to the implementation compared to the standard 15-case table [S6, unofficial].
*   **When to use:** When topological correctness on faces is required and scalar field values are available.
*   **When not to use:** When interior ambiguities are present and full topological correctness is required (use [Topologically Correct Marching Cubes](1368-topologically-correct-marching-cubes.md) or [Marching Cubes with Hermite Data and Full Case Table](961-marching-cubes-with-hermite-data-and-full-case-table.md) instead) [S2][S5, unofficial].

## Background
The Asymptotic Decider was proposed by Nielson and Hamann in 1991 in the paper "The Asymptotic Decider: Resolving the Ambiguity in Marching Cubes" [S1, S2][S5, S7, unofficial]. It was developed to address the face ambiguities in the original Marching Cubes algorithm, which were caused by the behavior of the trilinear interpolant on cube faces [S1, S2]. The method adds 11 new cases to the original 14-case table, creating subcases for original cases 3, 6, 7, 10, 12, and 13, which exhibit face ambiguity [S1, S8]. Natarajan later observed in 1994 that the ambiguity problem also occurs inside the cube, leading to further extensions like Marching Cubes 33 [S2][S5, unofficial].

## Sources
[S1] A Evolucao do Algoritmo Marching Cubes, https://doi.org/10.5540/03.2017.005.01.0107, fetched 2026-10-26
[S2] Marching Cubes algorithm, https://en.wikipedia.org/wiki/Marching_cubes, fetched 2026-10-26
[S3] Isosurface, https://en.wikipedia.org/wiki/Isosurface, fetched 2026-10-26
[S4] Topologically Accurate Dual Isosurfacing Using Ray Intersection, https://doi.org/10.20385/1860-2037/4.2007.4, fetched 2026-10-26
[S5] RobertBoettcherSF/Ada-Marching-Cubes, https://github.com/RobertBoettcherSF/Ada-Marching-Cubes, fetched 2026-10-27 (unofficial)
[S6] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
[S7] Resolving Ambiguities in Marching Squares - BorisTheBrave.Com, https://www.boristhebrave.com/2022/01/03/resolving-ambiguities-in-marching-squares/, fetched 2026-10-28 (unofficial)
[S8] New metric for evaluating the accuracy of marching isosurfacing algorithms, https://doi.org/10.1145/2638404.2638484, fetched 2026-11-05
