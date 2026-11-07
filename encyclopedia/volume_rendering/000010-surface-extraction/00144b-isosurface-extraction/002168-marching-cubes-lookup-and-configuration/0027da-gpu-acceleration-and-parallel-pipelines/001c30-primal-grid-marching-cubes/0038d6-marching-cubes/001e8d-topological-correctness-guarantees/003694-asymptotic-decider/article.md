# Asymptotic Decider

Also called: NH91 test, Nielson and Hamann decider, Nielson-Hamann Face Disambiguation, Resolving Ambiguities in Marching Squares, bilinear interpolation test

## Overview
The Asymptotic Decider is a topological disambiguation test used in Marching Cubes to resolve face ambiguities where the sign pattern of vertex values is insufficient to determine the unique connectivity of the isosurface intersection on a cell face [S1, S2][S6, unofficial]. It distinguishes itself from sibling methods like [Extended Trilinear Triangulation](1618-extended-trilinear-triangulation.md) by focusing exclusively on the bilinear interpolant on the 2D faces of the cube, rather than analyzing the full 3D trilinear interior critical points [S2][S5, unofficial]. While it guarantees manifold consistency on faces, it does not resolve interior ambiguities (such as those in Case 13 or 14) without additional tests like Natarajan’s or Chernyaev’s [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field value $S_v$ for each of the four vertices of a square face (2D cell) or a hexahedral face (3D cell). The vertices are indexed $0, 1, 2, 3$.
*   **Output:** A boolean or integer indicating the correct triangulation configuration for the ambiguous face, or a decision on which diagonal to connect.

**Data Structures:**
The method operates on the four scalar values associated with the corners of the face. Let the corners be indexed such that $0$ and $2$ are diagonally opposite, and $1$ and $3$ are diagonally opposite.
*   $S_0, S_1, S_2, S_3$: Scalar values at vertices $0, 1, 2, 3$.

**Ambiguity Detection:**
A face is considered ambiguous if the signs of the vertices alternate around the perimeter. Specifically, the diagonally opposite pairs have the same sign, but the pairs are opposite to each other [S6, unofficial]:
1.  $S_0 > 0$ and $S_2 > 0$, while $S_1 < 0$ and $S_3 < 0$.
2.  $S_0 < 0$ and $S_2 < 0$, while $S_1 > 0$ and $S_3 > 0$.

If the signs do not match this alternating pattern, the topology is determined by the standard lookup table, and the decider is not invoked [S1][S6, unofficial].

**The Test (Formula):**
The test computes a discriminant value $V$ (also referred to as $Q$ in 2D contexts) using bilinear interpolation coefficients [S4][S6, unofficial].

For a face with corners $0, 1, 2, 3$ (where $0$ is bottom-left, $1$ is bottom-right, $2$ is top-right, $3$ is top-left in a standard 2D grid, or mapped appropriately for 3D faces), the formula is:

$$ V = S_0 S_2 - S_1 S_3 $$

*Note: Source [S4] provides a more general form involving an isovalue $I$: $V = T_{00}T_{11} - T_{01}T_{10} - I(T_{00} + T_{11} - T_{01} - T_{10})$. However, for the standard binary sign-based ambiguity resolution described in [S6, unofficial] and commonly implemented in the NH91 test for face disambiguation, the simplified product difference $S_0 S_2 - S_1 S_3$ is the core discriminator derived from the bilinear saddle point analysis [S4][S6, unofficial]. If the isovalue is zero, the terms involving $I$ vanish, leaving the product difference. If the isovalue is non-zero, the full bilinear test should be used [S4]. We provide the general bilinear form below as it is more robust.*

Let the four corners of the face be $C_{00}, C_{10}, C_{01}, C_{11}$ corresponding to indices $(0,0), (1,0), (0,1), (1,1)$ in local face coordinates. Let $I$ be the isovalue.
$$ V = C_{00}C_{11} - C_{10}C_{01} - I(C_{00} + C_{11} - C_{10} - C_{01}) $$

**Decision Logic:**
1.  If $V < 0$: The isocontour forms a rectangular hyperbola in the first and third quadrants relative to the saddle point. The connectivity connects the vertices with the same sign as $C_{00}$ and $C_{11}$ (diagonal 0-2) [S4].
2.  If $V > 0$: The isocontour forms a rectangular hyperbola in the second and fourth quadrants. The connectivity connects the vertices with the same sign as $C_{10}$ and $C_{01}$ (diagonal 1-3) [S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def asymptotic_decider_face(scalar_00, scalar_10, scalar_01, scalar_11, isovalue):
    """
    Resolves face ambiguity for Marching Cubes.
    
    Args:
        scalar_00: Scalar value at corner (0,0)
        scalar_10: Scalar value at corner (1,0)
        scalar_01: Scalar value at corner (0,1)
        scalar_11: Scalar value at corner (1,1)
        isovalue: The threshold value for the isosurface
        
    Returns:
        int: 0 if diagonal 00-11 is chosen, 1 if diagonal 10-01 is chosen.
             Returns None if the face is not ambiguous.
    """
    # Check for ambiguity: diagonally opposite corners must have the same sign,
    # and the two diagonals must have opposite signs.
    # We check if (00 and 11 are positive) and (10 and 01 are negative)
    # OR (00 and 11 are negative) and (10 and 01 are positive).
    
    pos_00 = scalar_00 > isovalue
    pos_10 = scalar_10 > isovalue
    pos_01 = scalar_01 > isovalue
    pos_11 = scalar_11 > isovalue
    
    # Ambiguity condition: (00 == 11) != (10 == 01) is not quite right.
    # Correct condition: (00 and 11 have same sign) AND (10 and 01 have same sign)
    # AND (sign(00) != sign(10))
    
    diag1_same = (pos_00 == pos_11)
    diag2_same = (pos_10 == pos_01)
    diag_diff = (pos_00 != pos_10)
    
    if not (diag1_same and diag2_same and diag_diff):
        return None  # Not ambiguous, use standard table
        
    # Compute the discriminant V using bilinear interpolation coefficients
    # V = T00*T11 - T10*T01 - I*(T00 + T11 - T10 - T01)
    # Note: Using raw scalar values. If scalars are signed distances, 
    # isovalue is typically 0. If scalars are raw intensity, isovalue is explicit.
    
    T00 = scalar_00
    T10 = scalar_10
    T01 = scalar_01
    T11 = scalar_11
    
    V = (T00 * T11) - (T10 * T01) - isovalue * (T00 + T11 - T10 - T01)
    
    if V < 0:
        # Hyperbola in 1st and 3rd quadrants (relative to saddle)
        # Connects 00 and 11
        return 0
    else:
        # Hyperbola in 2nd and 4th quadrants
        # Connects 10 and 01
        return 1
```

**Edge Cases and Pitfalls:**
*   **Precision:** If $V$ is extremely close to 0, the saddle point lies very near the center of the face. Floating-point errors may cause instability. A small epsilon tolerance may be required, though the topology remains consistent as long as the sign is stable [S4].
*   **Boundary Conditions:** The test assumes the face is internal to the volume. Faces on the domain boundary do not require disambiguation for mesh manifoldness relative to neighbors, though they still require correct local topology [S1].
*   **Isovalue:** The formula depends on the isovalue $I$. If the scalar field is a signed distance field (SDF), $I=0$, simplifying the test to $V = T_{00}T_{11} - T_{10}T_{01}$ [S6, unofficial].

## Subtopics

### Lopes and Brodlie Extension
An extension of Natarajan’s interior tests to all ambiguous cases of Marching Cubes, proposed by Lopes and Brodlie in 2003 [S1]. While the Asymptotic Decider handles face ambiguities, this extension addresses the remaining interior ambiguities (such as Case 7 having two critical points) that the original NH91 test and Natarajan’s initial 1994 test did not fully cover [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per face. The test involves a constant number of arithmetic operations (multiplications and additions).
*   **Memory Complexity:** $O(1)$ auxiliary memory. No additional data structures are required beyond the vertex values.
*   **Performance:** Extremely fast, suitable for real-time GPU implementation. It adds negligible overhead to the standard Marching Cubes lookup [S5, unofficial].
*   **Strengths:** Simple to implement; resolves the most common topological errors (face saddles) that cause non-manifold edges in standard Marching Cubes [S1, S2].
*   **Limitations:** Does not resolve interior ambiguities (e.g., Case 13, 14, or the double-saddle Case 7). Using it alone results in topological errors in complex scalar fields [S1, S2].
*   **When to use:** When implementing a standard Marching Cubes algorithm that requires manifold output but cannot afford the complexity of full Chernyaev 33 or Lewiner tables. It is a necessary component for any "topologically correct" Marching Cubes variant that does not use tetrahedral decomposition [S5, unofficial].

## Background
The Asymptotic Decider was proposed by Nielson and Hamann in 1991 in the paper "The Asymptotic Decider: Resolving the Ambiguity in Marching Cubes" [S1, S2][S5, unofficial]. It was introduced to address the face ambiguities identified by Dürst, which arose from the behavior of the trilinear interpolant on the faces of the cube [S1, S2]. Prior to this, arbitrary choices were made for ambiguous cases, leading to topological inconsistencies [S6, unofficial]. Natarajan later observed in 1994 that similar ambiguities existed in the interior of the cube, leading to further extensions like Marching Cubes 33 [S2][S5, unofficial]. The method is named "Asymptotic" because it analyzes the asymptotic behavior of the bilinear interpolant near the saddle point [S4][S6, unofficial].

## Sources
[S1] A Evolucao do Algoritmo Marching Cubes, https://doi.org/10.5540/03.2017.005.01.0107, fetched 2026-10-26
[S2] Marching Cubes algorithm, https://en.wikipedia.org/wiki/Marching_cubes, fetched 2026-10-26
[S3] Isosurface, https://en.wikipedia.org/wiki/Isosurface, fetched 2026-10-26
[S4] Topologically Accurate Dual Isosurfacing Using Ray Intersection, https://doi.org/10.20385/1860-2037/4.2007.4, fetched 2026-10-26
[S5] RobertBoettcherSF/Ada-Marching-Cubes, https://github.com/RobertBoettcherSF/Ada-Marching-Cubes, fetched 2026-10-27 (unofficial)
[S6] Resolving Ambiguities in Marching Squares - BorisTheBrave.Com, https://www.boristhebrave.com/2022/01/03/resolving-ambiguities-in-marching-squares/, fetched 2026-10-28 (unofficial)
