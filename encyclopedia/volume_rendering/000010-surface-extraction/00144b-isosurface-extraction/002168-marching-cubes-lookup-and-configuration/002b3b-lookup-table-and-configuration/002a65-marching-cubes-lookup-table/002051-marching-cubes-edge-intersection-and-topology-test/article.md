# Marching Cubes Edge Intersection and Topology Test

Also called: Marching Cubes Case 1-11 Logic, testInterior

## Overview
This method resolves the topological ambiguity of the six ambiguous cases (6, 10, 14, 15, 18, 19) in the Marching Cubes algorithm by determining the correct triangulation of intersection points on a voxel edge based on the signs of the scalar field values at the four vertices of an adjacent face. It serves as the decision logic for cases 1 through 11 in specific implementations that use auxiliary arrays or directional masks rather than the full 256-case lookup table. It differs from standard Marching Cubes by relying on a directional mask and sign-change detection rather than a full 256-case lookup table, and is typically used in dual contouring or octree-based surface extraction where edge topology is managed via node indices [S1, unofficial].

## How it works
**Inputs and Outputs:**
- `cube`: An array of 8 scalar field values corresponding to the 8 vertices of a voxel. The vertex indexing follows the standard Marching Cubes convention:
  - Bottom face ($z=0$): `cube[0]` (0,0,0), `cube[1]` (1,0,0), `cube[2]` (1,1,0), `cube[3]` (0,1,0).
  - Top face ($z=1$): `cube[4]` (0,0,1), `cube[5]` (1,0,1), `cube[6]` (1,1,1), `cube[7]` (0,1,1).
- `edgeIdx`: The index of the edge being processed (0–11).
- `alongEdgeIdx`: An integer identifier for the edge direction or topology context.
- **Output**: An integer representing the triangulation choice or edge orientation. A negative value indicates one orientation/triangulation, a positive value indicates the opposite.

**Data Structures:**
- `cube`: `float[8]` or `double[8]`.
- `test`: `int`, bitmask accumulator.

**Numbered Steps:**

1. **Calculate Intersection Parameters and Points:**
   For the specific `edgeIdx` being processed, calculate the linear interpolation parameter $t$ and the four intersection points $A_t, B_t, C_t, D_t$. These points represent the scalar values at the intersection of the isosurface (value 0) with the edges of the adjacent face. The formulas depend on the edge index. Note that the source provides specific formulas for cases 1–11, which map to specific edge indices. The general form for linear interpolation between two vertices with values $V_0$ and $V_1$ is $t = V_0 / (V_0 - V_1)$, and the interpolated value at the intersection is $0$ by definition, but the source code calculates `Bt`, `Ct`, `Dt` as the *values* at the intersection points relative to the face geometry, effectively checking the sign of the field at the intersection. However, looking closely at the source, `At` is always 0. `Bt`, `Ct`, `Dt` are calculated as linear combinations. Let's trace the logic:
   
   The source code calculates `t` and then uses it to compute `Bt`, `Ct`, `Dt`. These variables `At`, `Bt`, `Ct`, `Dt` appear to be the *scalar field values* at the intersection points on the four edges of the face adjacent to the edge being tested. If the isosurface passes through an edge, the value at the intersection is 0. The source logic seems to be checking the sign of the field at the *other* vertices of the face to determine the topology. Let's re-read the source carefully.
   
   Source [S1, unofficial] says:
   `t = cube[1] / (cube[1] - cube[0]);`
   `At = 0;`
   `Bt = cube[0] + (cube[1] - cube[0]) * t;` -> This simplifies to `cube[0] + cube[1] - cube[0] = cube[1]`. This seems redundant.
   
   Actually, looking at the standard Marching Cubes ambiguity resolution, we often look at the sign of the field at the center of the face or the vertices of the adjacent face. The source code's variable names `At, Bt, Ct, Dt` are used in the test bitmask.
   
   Let's look at the bitmask logic:
   `if (At >= 0) test++;`
   `if (Bt >= 0) test += 2;`
   `if (Ct >= 0) test += 4;`
   `if (Dt >= 0) test += 8;`
   
   And the return values depend on `test`.
   
   The source provides specific formulas for `edgeIdx` 0 through 10. These correspond to the 11 cases mentioned in the alias.
   
   **Edge Index 0 (Case 1):**
   $t = \frac{cube[1]}{cube[1] - cube[0]}$
   $A_t = 0$
   $B_t = cube[0] + (cube[1] - cube[0]) \cdot t = cube[1]$
   $C_t = cube[4] + (cube[5] - cube[4]) \cdot t$
   $D_t = cube[5] + (cube[6] - cube[5]) \cdot t$
   
   Wait, the source says `Bt = cube[0] + (cube[1] - cube[0]) * t`. If $t = cube[1] / (cube[1] - cube[0])$, then $B_t = cube[0] + cube[1] - cube[0] = cube[1]$.
   $C_t = cube[4] + (cube[5] - cube[4]) \cdot \frac{cube[1]}{cube[1] - cube[0]}$
   $D_t = cube[5] + (cube[6] - cube[5]) \cdot \frac{cube[1]}{cube[1] - cube[0]}$
   
   This interpretation suggests that `Bt`, `Ct`, `Dt` are not the intersection points themselves (which would be 0), but rather the scalar values at the *other* vertices of the face, or interpolated values. However, the bitmask checks `>= 0`.
   
   Let's look at the standard ambiguity resolution for Case 6. The ambiguity is resolved by checking the sign of the field at the center of the cube or the vertices of the ambiguous face.
   
   The source code is from an unofficial repository. The logic seems to be:
   For a given edge, we define a face. The intersection on the edge is at $t$. We then evaluate the field at the other three corners of that face (or interpolated points) to determine the topology.
   
   Let's stick strictly to the formulas provided in [S1, unofficial].

   **Formulas for Edge Indices 0–10:**

   **Edge 0:**
   $$t = \frac{cube[1]}{cube[1] - cube[0]}$$
   $$A_t = 0$$
   $$B_t = cube[1]$$
   $$C_t = cube[4] + (cube[5] - cube[4]) \cdot t$$
   $$D_t = cube[5] + (cube[6] - cube[5]) \cdot t$$

   **Edge 1:**
   $$t = \frac{cube[2]}{cube[2] - cube[3]}$$
   $$A_t = 0$$
   $$B_t = cube[1]$$
   $$C_t = cube[5] + (cube[4] - cube[5]) \cdot t$$
   $$D_t = cube[6] + (cube[7] - cube[6]) \cdot t$$

   **Edge 2:**
   $$t = \frac{cube[3]}{cube[3] - cube[0]}$$
   $$A_t = 0$$
   $$B_t = cube[2]$$
   $$C_t = cube[6] + (cube[5] - cube[6]) \cdot t$$
   $$D_t = cube[7] + (cube[4] - cube[7]) \cdot t$$

   **Edge 3:**
   $$t = \frac{cube[4]}{cube[4] - cube[5]}$$
   $$A_t = 0$$
   $$B_t = cube[7]$$
   $$C_t = cube[3] + (cube[2] - cube[3]) \cdot t$$
   $$D_t = cube[0] + (cube[1] - cube[0]) \cdot t$$

   **Edge 4:**
   $$t = \frac{cube[5]}{cube[5] - cube[6]}$$
   $$A_t = 0$$
   $$B_t = cube[4]$$
   $$C_t = cube[0] + (cube[3] - cube[0]) \cdot t$$
   $$D_t = cube[1] + (cube[2] - cube[1]) \cdot t$$

   **Edge 5:**
   $$t = \frac{cube[6]}{cube[6] - cube[7]}$$
   $$A_t = 0$$
   $$B_t = cube[5]$$
   $$C_t = cube[1] + (cube[0] - cube[1]) \cdot t$$
   $$D_t = cube[2] + (cube[3] - cube[2]) \cdot t$$

   **Edge 6:**
   $$t = \frac{cube[7]}{cube[7] - cube[4]}$$
   $$A_t = 0$$
   $$B_t = cube[6]$$
   $$C_t = cube[2] + (cube[1] - cube[2]) \cdot t$$
   $$D_t = cube[3] + (cube[0] - cube[3]) \cdot t$$

   **Edge 7:**
   $$t = \frac{cube[0]}{cube[0] - cube[4]}$$
   $$A_t = 0$$
   $$B_t = cube[3]$$
   $$C_t = cube[2] + (cube[6] - cube[2]) \cdot t$$
   $$D_t = cube[1] + (cube[5] - cube[1]) \cdot t$$

   **Edge 8:**
   $$t = \frac{cube[1]}{cube[1] - cube[5]}$$
   $$A_t = 0$$
   $$B_t = cube[0]$$
   $$C_t = cube[3] + (cube[7] - cube[3]) \cdot t$$
   $$D_t = cube[2] + (cube[6] - cube[2]) \cdot t$$

   **Edge 9:**
   $$t = \frac{cube[2]}{cube[2] - cube[6]}$$
   $$A_t = 0$$
   $$B_t = cube[1]$$
   $$C_t = cube[0] + (cube[4] - cube[0]) \cdot t$$
   $$D_t = cube[3] + (cube[7] - cube[3]) \cdot t$$

   **Edge 10:**
   $$t = \frac{cube[3]}{cube[3] - cube[7]}$$
   $$A_t = 0$$
   $$B_t = cube[2]$$
   $$C_t = cube[1] + (cube[5] - cube[1]) \cdot t$$
   $$D_t = cube[0] + (cube[4] - cube[0]) \cdot t$$

2. **Compute Test Bitmask:**
   Initialize `test = 0`.
   If $A_t \ge 0$, `test += 1`.
   If $B_t \ge 0$, `test += 2`.
   If $C_t \ge 0$, `test += 4`.
   If $D_t \ge 0$, `test += 8`.

3. **Return Result:**
   Based on the value of `test`:
   - If `test` is 0, 1, 2, 3, 4, 6, 9, 12: Return `-alongEdgeIdx`.
   - If `test` is 5: Return $(A_t \cdot C_t - B_t \cdot D_t) \cdot alongEdgeIdx$.
   - If `test` is 7, 11, 13, 14, 15: Return `alongEdgeIdx`.
   - If `test` is 8, 10: Return $-(A_t \cdot C_t - B_t \cdot D_t) \cdot alongEdgeIdx$.
   
   *Note: The source code snippet in [S1, unofficial] has a slightly different structure for the return values than the "step" description. The "code" section is more precise.*
   
   **Precise Return Logic from Code Section:**
   - If `test <= 4`: Return `-alongEdgeIdx`.
   - If `test == 5`: Return $(A_t \cdot C_t - B_t \cdot D_t) \cdot alongEdgeIdx$.
   - If `test == 6`: Return `-alongEdgeIdx`.
   - If `test == 7`: Return `alongEdgeIdx`.
   - If `test <= 9` (i.e., 8, 9): Return `-alongEdgeIdx`.
   - If `test == 10`: Return $-(A_t \cdot C_t - B_t \cdot D_t) \cdot alongEdgeIdx$.
   - If `test == 11`: Return `alongEdgeIdx`.
   - If `test == 12`: Return `-alongEdgeIdx`.
   - If `test <= 15` (i.e., 13, 14, 15): Return `alongEdgeIdx`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def test_interior(cube, edge_idx, along_edge_idx):
    # Validate edge_idx
    if edge_idx < 0 or edge_idx > 10:
        raise ValueError(f"testInterior got wrong edgeIdx: {edge_idx}")

    # Calculate t and intersection points based on edge_idx
    if edge_idx == 0:
        t = cube[1] / (cube[1] - cube[0])
        At = 0
        Bt = cube[1]
        Ct = cube[4] + (cube[5] - cube[4]) * t
        Dt = cube[5] + (cube[6] - cube[5]) * t
    elif edge_idx == 1:
        t = cube[2] / (cube[2] - cube[3])
        At = 0
        Bt = cube[1]
        Ct = cube[5] + (cube[4] - cube[5]) * t
        Dt = cube[6] + (cube[7] - cube[6]) * t
    elif edge_idx == 2:
        t = cube[3] / (cube[3] - cube[0])
        At = 0
        Bt = cube[2]
        Ct = cube[6] + (cube[5] - cube[6]) * t
        Dt = cube[7] + (cube[4] - cube[7]) * t
    elif edge_idx == 3:
        t = cube[4] / (cube[4] - cube[5])
        At = 0
        Bt = cube[7]
        Ct = cube[3] + (cube[2] - cube[3]) * t
        Dt = cube[0] + (cube[1] - cube[0]) * t
    elif edge_idx == 4:
        t = cube[5] / (cube[5] - cube[6])
        At = 0
        Bt = cube[4]
        Ct = cube[0] + (cube[3] - cube[0]) * t
        Dt = cube[1] + (cube[2] - cube[1]) * t
    elif edge_idx == 5:
        t = cube[6] / (cube[6] - cube[7])
        At = 0
        Bt = cube[5]
        Ct = cube[1] + (cube[0] - cube[1]) * t
        Dt = cube[2] + (cube[3] - cube[2]) * t
    elif edge_idx == 6:
        t = cube[7] / (cube[7] - cube[4])
        At = 0
        Bt = cube[6]
        Ct = cube[2] + (cube[1] - cube[2]) * t
        Dt = cube[3] + (cube[0] - cube[3]) * t
    elif edge_idx == 7:
        t = cube[0] / (cube[0] - cube[4])
        At = 0
        Bt = cube[3]
        Ct = cube[2] + (cube[6] - cube[2]) * t
        Dt = cube[1] + (cube[5] - cube[1]) * t
    elif edge_idx == 8:
        t = cube[1] / (cube[1] - cube[5])
        At = 0
        Bt = cube[0]
        Ct = cube[3] + (cube[7] - cube[3]) * t
        Dt = cube[2] + (cube[6] - cube[2]) * t
    elif edge_idx == 9:
        t = cube[2] / (cube[2] - cube[6])
        At = 0
        Bt = cube[1]
        Ct = cube[0] + (cube[4] - cube[0]) * t
        Dt = cube[3] + (cube[7] - cube[3]) * t
    elif edge_idx == 10:
        t = cube[3] / (cube[3] - cube[7])
        At = 0
        Bt = cube[2]
        Ct = cube[1] + (cube[5] - cube[1]) * t
        Dt = cube[0] + (cube[4] - cube[0]) * t
    else:
        # Edge 11 is not covered in the source's case 1-11 logic
        # This might be handled by a different method or assertion
        raise ValueError("Edge 11 not supported in this specific logic variant")

    # Compute test bitmask
    test = 0
    if At >= 0:
        test += 1
    if Bt >= 0:
        test += 2
    if Ct >= 0:
        test += 4
    if Dt >= 0:
        test += 8

    # Return result based on test value
    if test <= 4:
        return -along_edge_idx
    elif test == 5:
        return (At * Ct - Bt * Dt) * along_edge_idx
    elif test == 6:
        return -along_edge_idx
    elif test == 7:
        return along_edge_idx
    elif test <= 9:
        return -along_edge_idx
    elif test == 10:
        return -(At * Ct - Bt * Dt) * along_edge_idx
    elif test == 11:
        return along_edge_idx
    elif test == 12:
        return -along_edge_idx
    else: # test <= 15
        return along_edge_idx
```

**Parameters:**
- `cube`: Array of 8 floats.
- `edge_idx`: Integer 0–10.
- `along_edge_idx`: Integer, context-dependent.

**Edge Cases and Pitfalls:**
- **Division by Zero:** If `cube[i] - cube[j] == 0`, the calculation of `t` will result in infinity or NaN. This occurs when the scalar field values at the two vertices of the edge are equal. In this case, the isosurface is parallel to the edge, and the intersection is undefined or at infinity. The source does not specify a guard for this, but it is a critical pitfall [G].
- **Invalid Edge Index:** The source asserts failure if `edgeIdx` is not 0–10. Edge 11 is not handled in this specific logic [S1, unofficial].
- **Zero Values:** If any `cube` value is exactly 0, the intersection is at the vertex. The linear interpolation still works, but precision issues may arise.

## Subtopics

### [Marching Cubes Edge Processing Logic](3702-marching-cubes-edge-processing-logic.md)
This method generates two triangles for a voxel edge based on sign changes between the corners of four adjacent nodes, using `processEdgeMask` to identify the specific edge. It differs from standard Marching Cubes by relying on a directional mask and sign-change detection rather than a full 256-case lookup table, and is typically used in dual contouring or octree-based surface extraction where edge topology is managed via node indices.

## Operating characteristics
- **Time Complexity:** $O(1)$ per edge, as it involves a fixed number of arithmetic operations and comparisons.
- **Memory Complexity:** $O(1)$, requiring only a few temporary variables.
- **Strengths:** Avoids the memory overhead of a 256-entry lookup table. Provides a deterministic resolution for ambiguous cases.
- **Limitations:** The logic is specific to the implementation in [S1, unofficial] and may not be directly compatible with other Marching Cubes variants without adaptation. The handling of Edge 11 is missing.
- **When to use:** When memory is constrained and the specific ambiguity resolution logic of [S1, unofficial] is desired.
- **When not to use:** When using a standard 256-case lookup table or when Edge 11 needs to be processed.

## Background
This method originates from an unofficial implementation of Marching Cubes found in the `upupming/marching-cubes` repository [S1, unofficial]. It represents a specific approach to resolving the topological ambiguities of the Marching Cubes algorithm by using a bitwise test on interpolated values rather than a precomputed lookup table.

## Sources
[S1] upupming/marching-cubes, https://github.com/upupming/marching-cubes, fetched 2026-10-27 (unofficial)
