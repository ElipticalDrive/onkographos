# Asymptotic Decider

Also called: Nielson and Hamann decider, Nielson-Hamann Asymptotic Decider, Nielson-Hamann Face Disambiguation

## Overview
The Asymptotic Decider (AD) is a method for resolving topological ambiguities in Marching Cubes when a cube face has vertices with alternating signs, creating a saddle point on the trilinear interpolant [S1, S2, S8]. Unlike the original Marching Cubes algorithm, which arbitrarily selects one of two valid edge connections for ambiguous faces, the AD uses the scalar values at the face corners to determine the correct topology based on the behavior of the bilinear interpolant [S7, unofficial][S8]. It sets itself apart from sibling methods like [Topologically Correct Marching Cubes](1368-topologically-correct-marching-cubes.md) and [Marching Cubes with Hermite Data and Full Case Table](961-marching-cubes-with-hermite-data-and-full-case-table.md) by relying on a local geometric test on the face itself rather than global connectivity constraints or complex Hermite data, though it does not resolve interior cube ambiguities [S1, S2].

## How it works
The Asymptotic Decider resolves face-level ambiguities by comparing the isovalue to the saddle point value of the bilinear interpolant on the ambiguous face [S8].

### Inputs and Outputs
*   **Input:** A scalar field defined on a regular grid. For a specific cube face, the scalar values at its four corners ($V_{00}, V_{01}, V_{10}, V_{11}$) and the target isovalue $I$.
*   **Output:** A decision on which pair of edges to connect on the ambiguous face, effectively selecting one of two sub-cases in the triangulation table.

### Data Structures
The method operates on the scalar values of the four vertices of a single face. No additional data structures are required beyond the scalar field values.

### Steps
1.  **Identify Ambiguous Face:** A face is ambiguous if its four vertices have alternating signs (e.g., two diagonally opposite vertices are positive and the other two are negative) [S1][S7, unofficial].
2.  **Compute Discriminant Value ($V$):** Calculate the value $V$ using bilinear interpolation coefficients. The formula is:
    $$ V = T_{00}T_{11} - T_{01}T_{10} - I(T_{00} + T_{11} - T_{01} - T_{10}) $$
    where $T_{ij}$ are the scalar values at the corners of the face (indexed by local coordinates $i,j \in \{0,1\}$) and $I$ is the isovalue [S4].
    *   Note: In the 2D context (Marching Squares), this simplifies to $Q = \text{top\_left} \cdot \text{bottom\_right} - \text{bottom\_left} \cdot \text{top\_right}$, where the sign of $Q$ determines the connectivity [S7, unofficial]. The 3D formula above is the generalization for the face bilinear patch [S4].
3.  **Determine Topology:**
    *   If $V < 0$, the isocontour on the face is a rectangular hyperbola in the first and third quadrants (relative to the saddle point). The surface connects the edges associated with the positive diagonal [S4].
    *   If $V > 0$, the isocontour is a rectangular hyperbola in the second and fourth quadrants. The surface connects the edges associated with the negative diagonal [S4].
4.  **Select Triangulation:** Use the result to select the appropriate sub-case from the extended Marching Cubes table. Nielson and Hamann added 11 new cases to the original 14-case table, creating subcases for original cases 3, 6, 7, 10, 12, and 13, which exhibit face ambiguity [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def asymptotic_decider(face_values, isovalue):
    """
    Resolves face ambiguity for Marching Cubes.
    
    Args:
        face_values: List of 4 scalar values [v00, v01, v10, v11]
                     corresponding to corners (0,0), (0,1), (1,0), (1,1).
        isovalue: The scalar threshold I.
        
    Returns:
        int: 0 if V < 0 (connect positive diagonal), 1 if V > 0 (connect negative diagonal).
             Returns None if the face is not ambiguous.
    """
    v00, v01, v10, v11 = face_values
    
    # Check for ambiguity: alternating signs
    # Ambiguous if (v00 > 0 and v11 > 0 and v01 < 0 and v10 < 0)
    # OR (v00 < 0 and v11 < 0 and v01 > 0 and v10 > 0)
    is_ambiguous = (
        (v00 > isovalue and v11 > isovalue and v01 < isovalue and v10 < isovalue) or
        (v00 < isovalue and v11 < isovalue and v01 > isovalue and v10 > isovalue)
    )
    
    if not is_ambiguous:
        return None

    # Compute V
    # V = T00*T11 - T01*T10 - I*(T00 + T11 - T01 - T10)
    V = (v00 * v11) - (v01 * v10) - isovalue * (v00 + v11 - v01 - v10)
    
    if V < 0:
        return 0  # Connect positive diagonal
    else:
        return 1  # Connect negative diagonal
```

### Parameters
*   **Isovalue ($I$):** The threshold value for the scalar field. Typical values depend on the data (e.g., 0 for signed distance fields, mean intensity for medical imaging).
*   **Scalar Values ($T_{ij}$):** The values at the grid vertices.

### Edge Cases and Pitfalls
*   **Precision:** If $V$ is very close to 0, the saddle point is near the isovalue, leading to numerical instability in determining the topology [S4].
*   **Interior Ambiguities:** The Asymptotic Decider only resolves face ambiguities. It does not handle interior ambiguities (saddle points inside the cube), which require additional tests like Natarajan's test or the full Marching Cubes 33 table [S1, S2].
*   **Complexity:** Implementing the AD adds considerable complexity to the Marching Cubes implementation due to the need for extended case tables and additional calculations [S6, unofficial].

## Subtopics

### [Ambiguous Face Triangulation for Regular Grid](10105-ambiguous-face-triangulation-for-regular-grid.md)
Ambiguous Face Triangulation is a technique to resolve topological ambiguity in cell triangulation when a single face has sign changes along all four edges, resulting in four vertices on that face. It sets itself apart from the parent [Asymptotic Decider](790-asymptotic-decider.md) by using a geometric connectivity rule based on face orientation to determine the triangulation, ensuring that adjacent cells fit together topologically.

### Lopes and Brodlie Extension
An extension of Natarajan's tests to all ambiguous cases of Marching Cubes. Proposed by Lopes and Brodlie in 2003 [S1].

### Asymptotic Decider Interpolant
An alternative interpolant for ambiguous faces using the Asymptotic Decider [14] to resolve ambiguity. It produces four triangular sections and a flat diamond in the middle [S9, book].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per face. The calculation involves a constant number of arithmetic operations.
*   **Memory Complexity:** $O(1)$ additional memory per face.
*   **Strengths:** Simple to implement; resolves the most common source of topological errors in Marching Cubes (face ambiguities); improves surface smoothness by adapting to the scalar field values [S7, unofficial].
*   **Limitations:** Does not resolve interior ambiguities; requires the original scalar values at vertices (not just signs); adds complexity to the case table [S6, unofficial].
*   **When to use:** When face-level topological correctness is required and the scalar field values are available.
*   **When not to use:** When interior ambiguities are present and a full topologically correct algorithm (like MC33 or Lewiner) is needed; when only binary (sign-only) data is available.

## Background
The Asymptotic Decider was proposed by Nielson and Hamann in 1991 in the paper "The Asymptotic Decider: Resolving the Ambiguity in Marching Cubes" [S1, S2][S5, S7, unofficial]. It was developed to address the ambiguity problem identified in the original Marching Cubes algorithm, where linear interpolation on ambiguous faces could not determine the correct surface topology [S2]. Natarajan later observed in 1994 that the ambiguity problem also occurs inside the cube, leading to further extensions like Marching Cubes 33 [S2]. The method is widely used in medical imaging and computer graphics for isosurface extraction [S5, S6, unofficial].

## Sources
[S1] A Evolucao do Algoritmo Marching Cubes, https://doi.org/10.5540/03.2017.005.01.0107, fetched 2026-10-26
[S2] Marching Cubes algorithm, https://en.wikipedia.org/wiki/Marching_cubes, fetched 2026-10-26
[S3] Isosurface, https://en.wikipedia.org/wiki/Isosurface, fetched 2026-10-26
[S4] Topologically Accurate Dual Isosurfacing Using Ray Intersection, https://doi.org/10.20385/1860-2037/4.2007.4, fetched 2026-10-26
[S5] RobertBoettcherSF/Ada-Marching-Cubes, https://github.com/RobertBoettcherSF/Ada-Marching-Cubes, fetched 2026-10-27 (unofficial)
[S6] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
[S7] Resolving Ambiguities in Marching Squares - BorisTheBrave.Com, https://www.boristhebrave.com/2022/01/03/resolving-ambiguities-in-marching-squares/, fetched 2026-10-28 (unofficial)
[S8] New metric for evaluating the accuracy of marching isosurfacing algorithms, https://doi.org/10.1145/2638404.2638484, fetched 2026-11-05
[S9] Interpolants Induced by Marching Cases, https://doi.org/10.4230/dfu.vol2.sciviz.2011.48, fetched 2026-10-26 (book)
