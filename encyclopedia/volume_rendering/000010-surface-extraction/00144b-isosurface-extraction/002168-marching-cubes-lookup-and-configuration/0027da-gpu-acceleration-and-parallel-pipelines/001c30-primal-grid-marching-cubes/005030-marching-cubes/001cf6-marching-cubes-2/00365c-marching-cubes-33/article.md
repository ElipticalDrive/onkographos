# Marching Cubes 33

Also called: 33-case Marching Cubes, Advanced Marching Cubes, Chernyaev 1995, Construction of Topologically Correct Isosurfaces

## Overview
Marching Cubes 33 is an isosurface extraction algorithm that resolves the topological ambiguities inherent in the original 15-configuration Marching Cubes method by analyzing the trilinear interpolant within each voxel [S2, unofficial]. It increases the lookup table from 15 to 33 entries to ensure the generated triangle mesh topology coincides exactly with the topology of the underlying trilinear function, preventing artifacts such as holes at shared faces between adjacent cells [S2, unofficial]. Unlike [Marching Cubes 2](1393-marching-cubes-2.md), which may use alternative lookup strategies, MC33 explicitly derives sub-cases based on face and internal connectivity tests [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A volumetric scalar field defined on a cubic grid. For each voxel (cube), the scalar values at the 8 corner nodes ($F_{000}$ to $F_{111}$) and a target isovalue [S2, unofficial].
*   **Output:** A set of triangles representing the isosurface within the voxel [S2, unofficial].

**Data Structures**
*   **Lookup Table:** An array of 33 entries. Each entry corresponds to a specific topological configuration derived from the 16 standard binary cases, split into sub-cases (e.g., 4.1.1, 4.1.2, 7.3, 13.3) based on ambiguity resolution [S2, unofficial].
*   **Vertex Data:** Coordinates of vertices on the cube edges (via linear interpolation) and potentially one additional vertex inside the cube for complex cases [S2, unofficial].

**Definitions**
*   **Trilinear Interpolation:** The scalar function $F(q,s,t)$ over a cube with local coordinates $q,s,t \in [0,1]$ is:
    $$F(q,s,t) = F_{000}(1-q)(1-s)(1-t) + F_{100}q(1-s)(1-t) + F_{010}(1-q)s(1-t) + F_{110}qs(1-t) + F_{001}(1-q)(1-s)t + F_{101}q(1-s)t + F_{011}(1-q)st + F_{111}qst$$
    [S2, unofficial]
*   **Separated vs. Joined Nodes:** Two nodes of the same sign are **joined** (non-separated) if there exists a path between them along which $F$ does not change sign. They are **separated** if no such path exists [S2, unofficial].
*   **Ambiguous Face:** A face with two positive and two negative diagonally opposed nodes. The node signs alone are insufficient to determine connectivity [S2, unofficial].
*   **Internal Ambiguity:** A situation where nodes are separated on the faces but joined inside the cube (e.g., Configuration 4) [S2, unofficial].

**Algorithm Steps**

1.  **Determine Base Configuration:** Compute the binary index $I$ based on the signs of the 8 corner nodes relative to the isovalue. Map $I$ to one of the 16 base configurations. For symmetric cases (e.g., 10, 12, 13), reverse signs if necessary so that the number of faces with separated positive nodes $\ge$ number of faces with joined positive nodes [S2, unofficial].
2.  **Resolve Face Ambiguity:** For each face of the cube that is ambiguous (two positive, two negative diagonally opposed nodes): Let the corner values on the face be $A, B, C, D$ such that $A, C$ are positive and $B, D$ are negative. Compare the products $AC$ and $BD$: If $AC > BD$, positive nodes are joined and negative nodes are separated. If $AC < BD$, positive nodes are separated and negative nodes are joined [S2, unofficial].
3.  **Resolve Internal Ambiguity:** For configurations with internal ambiguity (e.g., 4, 6, 7, 10, 12, 13), determine if separated nodes on faces are joined inside the cube.
    *Method 1 (Hyperbola Intersection):* Compare hyperbolas on opposite faces. If the projections intersect, nodes are joined. This is determined by: $(A_1 - A_0)(C_1 - C_0) - (B_1 - B_0)(D_1 - D_0) > 0$ [S2, unofficial].
    *Method 2 (Quadratic Inequality - Preferred):* Check for a plane parallel to a face where nodes of the same sign become joined. Define a quadratic inequality $at^2 + bt + c > 0$ derived from bilinear interpolation on parallel faces. Steps: 1. Check if $a < 0$ (parabola opens downward). 2. Check if the maximum $t_{max} = -b/2a$ lies in $(0, 1)$. 3. Check if the corner nodes at $t=t_{max}$ have the required signs and the inequality holds. If all conditions are met, the areas are joined inside the cube [S2, unofficial].
4.  **Select Sub-Case:** Based on the results of face and internal ambiguity tests, select the specific sub-case from the 33-entry lookup table. Simple configurations (0, 1, 2, 5, 8, 9, 11, 14) have no ambiguous faces and no internal ambiguity. Configuration 4 resolves to 4.1.1 or 4.1.2. Configuration 6 resolves to 6.1.1, 6.1.2, or 6.2. Configuration 7 resolves to 7.1–7.4.2. Configuration 10 resolves to 10.1.1, 10.1.2, or 10.2. Configuration 12 resolves to 12.1.1, 12.1.2, 12.2, or 12.3. Configuration 13 resolves to 13.1–13.5.2. Cases requiring an additional vertex (e.g., 7.3, 10.2, 12.2, 12.3, 13.3, 13.4) are identified here [S2, unofficial].
5.  **Generate Vertices:** For each edge intersected by the isosurface, compute the vertex position using linear interpolation: $V = V_1 + \frac{I - F_1}{F_2 - F_1} (V_2 - V_1)$, where $V_1, V_2$ are cube corners and $F_1, F_2$ are their scalar values [S2, unofficial]. For cases requiring an internal vertex (e.g., 7.3, 10.2, 12.2, 12.3, 13.3, 13.4), place a vertex inside the cube. A simple choice is the average of all other generated vertices in the cell [S2, unofficial].
6.  **Construct Triangles:** Use the selected sub-case pattern to connect the generated vertices into triangles. If an additional vertex exists, it serves as a common node for all triangles in the model [S2, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_33(cube_values, isovalue=0.0):
    # Corner ordering: 000, 100, 110, 010, 001, 101, 111, 011
    # Edge definitions (12 edges, each defined by two corner indices)
    EDGES = [
        (0, 1), (2, 1), (2, 3), (0, 3),  # Bottom face (t=0)
        (4, 5), (6, 5), (6, 7), (4, 7),  # Top face (t=1)
        (0, 4), (1, 5), (2, 6), (3, 7)   # Vertical edges
    ]
    # Face definitions (6 faces, each defined by 4 corner indices in order)
    FACES = [
        (0, 1, 2, 3), (4, 5, 6, 7),  # Bottom, Top
        (0, 1, 5, 4), (3, 2, 6, 7),  # Front, Back
        (0, 3, 7, 4), (1, 2, 6, 5)   # Left, Right
    ]
    # Diagonal pairs for each face (A,C are positive, B,D are negative)
    FACE_DIAGS = [
        ((0, 2), (1, 3)), ((4, 6), (5, 7)),
        ((0, 5), (1, 4)), ((3, 6), (2, 7)),
        ((0, 7), (3, 4)), ((1, 6), (2, 5))
    ]
    # Unit cube corner positions
    CUBE_POS = [
        (0,0,0), (1,0,0), (1,1,0), (0,1,0),
        (0,0,1), (1,0,1), (1,1,1), (0,1,1)
    ]

    # 1. Determine base configuration index
    bits = [1 if v > isovalue else 0 for v in cube_values]
    config_index = sum(b << i for i, b in enumerate(bits))

    # 2. Resolve Face Ambiguities
    face_joined = [False] * 6
    for f_idx, (diag_pos, diag_neg) in enumerate(FACE_DIAGS):
        A, C = cube_values[diag_pos[0]], cube_values[diag_pos[1]]
        B, D = cube_values[diag_neg[0]], cube_values[diag_neg[1]]
        if A * C > B * D:
            face_joined[f_idx] = True

    # 3. Resolve Internal Ambiguities
    internal_joined = False
    # Method 2 check for Configuration 4 (diagonal nodes joined inside)
    if config_index == 4:
        A0, C0, B0, D0 = cube_values[0], cube_values[2], cube_values[1], cube_values[3]
        A1, C1, B1, D1 = cube_values[4], cube_values[6], cube_values[5], cube_values[7]
        val = (A1 - A0) * (C1 - C0) - (B1 - B0) * (D1 - D0)
        internal_joined = val > 0

    # 4. Select Sub-Case
    sub_case = compute_sub_case(config_index, face_joined, internal_joined)

    # 5. Generate Vertices
    edge_vertices = []
    for e_idx in range(12):
        if is_edge_intersected(e_idx, bits):
            v1, v2 = cube_values[EDGES[e_idx][0]], cube_values[EDGES[e_idx][1]]
            p1, p2 = CUBE_POS[EDGES[e_idx][0]], CUBE_POS[EDGES[e_idx][1]]
            t = (isovalue - v1) / (v2 - v1)
            edge_vertices.append(lerp(p1, p2, t))
        else:
            edge_vertices.append(None)

    additional_vertex = None
    if sub_case.requires_additional_vertex:
        valid_verts = [v for v in edge_vertices if v is not None]
        if valid_verts:
            additional_vertex = sum(valid_verts) / len(valid_verts)

    # 6. Construct Triangles
    triangles = []
    pattern = LOOKUP_TABLE[sub_case.key]
    for tri in pattern:
        triangle = []
        for idx in tri:
            if idx == -1:
                triangle.append(additional_vertex)
            else:
                triangle.append(edge_vertices[idx])
        triangles.append(triangle)
    return triangles

def is_edge_intersected(edge_idx, bits):
    c1, c2 = EDGES[edge_idx]
    return bits[c1] != bits[c2]

def lerp(p1, p2, t):
    return (p1[0] + t*(p2[0]-p1[0]), p1[1] + t*(p2[1]-p1[1]), p1[2] + t*(p2[2]-p1[2]))

def compute_sub_case(config, face_joined, internal_joined):
    # The sources do not provide the full 33-case branching logic; 
    # in practice this is a direct lookup or a cascade of if/elif statements 
    # mapping (config, face_joined, internal_joined) to the sub-case key.
    if config == 4:
        return SubCase(4.1.1 if not internal_joined else 4.1.2, False, [[0]])
    # ... (Full 33-case logic would be enumerated here)
    return SubCase(0, False, [])

class SubCase:
    def __init__(self, key, requires_additional_vertex, pattern):
        self.key = key
        self.requires_additional_vertex = requires_additional_vertex
        self.pattern = pattern

# Lookup table mapping sub-case keys to triangle index lists
# -1 denotes the additional internal vertex
LOOKUP_TABLE = {
    0: [],
    1: [[0]],
    2: [[1]],
    3.1: [[2]],
    3.2: [[3]],
    4.1.1: [[4]],
    4.1.2: [[5]],
    # ... (All 33 cases enumerated per source [S2, unofficial])
}
```

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels. Each voxel is processed independently. The constant factor is higher than standard Marching Cubes due to the additional arithmetic for ambiguity resolution (multiplications, comparisons, and potentially quadratic root finding) [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for the output mesh. The lookup table is small (33 entries) [S2, unofficial].
*   **Performance:** The algorithm is computationally more expensive than the original 15-case MC due to the branching and arithmetic required for ambiguity resolution. However, it avoids post-processing steps to fix topological errors [S2, unofficial].
*   **Strengths:** Guarantees topological correctness of the isosurface relative to the trilinear interpolant. Eliminates holes and self-intersections caused by ambiguous face/internal connectivity mismatches between adjacent cells [S2, unofficial].
*   **Limitations:** More complex implementation. The "additional vertex" in cases like 7.3 and 13.3 can introduce non-manifold geometry or visual artifacts if not handled carefully in rendering. The choice of the additional vertex (e.g., average) is heuristic and may not perfectly represent the true isosurface geometry [S2, unofficial].
*   **When to use:** When topological correctness is critical (e.g., medical imaging, scientific visualization) and the underlying data is well-sampled enough that the trilinear interpolant is a valid approximation [S2, unofficial].
*   **When not to use:** For real-time applications where performance is paramount and minor topological artifacts are acceptable. For data with high-frequency noise where the trilinear interpolant is not a good model [S2, unofficial].

## Background
The Marching Cubes 33 algorithm was proposed by Boris E. Chernyaev in 1995 in the paper "Construction of Topologically Correct Isosurfaces" [S2, unofficial]. It was implemented in the framework of the PAW/HIGZ project at CERN (European Organization for Nuclear Research) [S1]. The work addressed the limitations of the original Marching Cubes algorithm by Lorensen and Cline (1987), which used a 15-configuration lookup table and suffered from topological ambiguities [S1]. Chernyaev demonstrated that the trilinear function permits 33 topologically distinct configurations within a cube, necessitating an enlarged lookup table [S2, unofficial]. Nielson proved in 2003 that Chernyaev's lookup table is complete and can represent all possible behaviors of the trilinear interpolant [S3]. Lewiner et al. proposed an implementation in 2003, and Custodio et al. noted and corrected algorithmic inaccuracies in 2013 [S3].

## Sources
[S1] Computing and Displaying Isosurfaces in R, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] Marching cubes 33 : construction of topologically correct isosurfaces, https://repository.cern/records/7zfxg-q0t96/files/cn-95-017.pdf, fetched 2026-10-26 (unofficial)
[S3] A Evolucao do Algoritmo Marching Cubes, https://doi.org/10.5540/03.2017.005.01.0107, fetched 2026-10-26
[S4] Marching Cubes algorithm, https://en.wikipedia.org/wiki/Marching_cubes, fetched 2026-10-26
