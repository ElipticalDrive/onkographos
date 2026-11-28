# Multi-Tetra Framework

Also called: BPS, DPS, Initial Subdivision, Multi-Tetra Framework Subdivision, Multiresolution Tetrahedral Framework, Recursive Tetrahedral Subdivision

## Overview
The Multi-Tetra Framework is a multiresolution representation that approximates regular volume data using tetrahedra of varying levels of detail, generated through recursive subdivision and stored in a binary tree structure [S1]. It solves the crack problem between different levels of detail by ensuring that the subdivision process only uses existing vertex indices without adding new sample points or requiring interpolation [S1]. Unlike sibling methods that rely on fixed grid resolutions or texture mipmaps, this framework allows for smooth changes in resolution across areas of different levels within the same framework by enabling error-based fusion of tetrahedra from multiple levels [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A regular volume data set with dimensions $L \times L \times L$ [S1].
*   **Output:** A multiresolution tetrahedral framework represented by a full binary tree, where each node corresponds to a tetrahedron [S1]. The framework supports the generation of an Error-Based Model (EBM) by selectively fusing tetrahedra [S1].

**Data Structures:**
The framework is represented by a full binary tree and stored in a linear array to allow for instant indexing [S1].
*   **Vertex Indexing:** Vertices are identified by their grid indices $(i, j, k)$ [S1]. The linear index $V$ for a vertex is calculated as:
    $$ V = i + j(2N + 1) + k(2N + 1)^2 $$
    where $N$ is a parameter defining the volume extension [S1].
*   **Tetrahedron Structure:** Each node in the binary tree represents a tetrahedron [S1]. Since the subdivision is topological (based on vertex indices), the physical coordinates are not stored in the tree structure itself but are derived from the vertex indices and the original volume geometry [S1].

**Parameters:**
*   **$N$:** An integer parameter that determines the resolution of the extended volume. The input volume is extended to dimensions $(2N + 1) \times (2N + 1) \times (2N + 1)$ by padding with zero slices [S1].

**Algorithm Steps:**

1.  **Volume Extension:**
    Extend the input regular volume data set to dimensions $(2N + 1) \times (2N + 1) \times (2N + 1)$ by padding with slices of zero [S1].

2.  **Initial Subdivision:**
    *   Start with the bounding box of the extended volume, imagined as a cube with 8 vertices coinciding with the corners of the volume [S1].
    *   Add the center point of the volume to the framework. Connect this center point to all 8 vertices of the box, forming 6 pyramids [S1].
    *   Divide each pyramid into two tetrahedra by connecting the diagonal of the base face. This generates **12 tetrahedra** [S1].
    *   All 12 initial tetrahedra are classified as **Case 1** [S1].

3.  **Recursive Subdivision:**
    The subdivision is a vertex-adding process where each step picks a vertex from the original volume and inserts it into the framework, dividing a tetrahedron into two [S1]. Each vertex is added only once [S1]. The process involves three specific steps (Step 1, Step 2, Step 3) that cycle based on the configuration of the tetrahedra [S1].

    *   **Step 1 Subdivision:**
        *   Applies to tetrahedra in **Case 1** [S1].
        *   Adds the midpoint of an edge of the tetrahedron. The point added has an index equal to the average of the index values of the two endpoints [S1].
        *   The tetrahedron is bisected into two by the plane passing through the added point and its opposite edge [S1].
        *   The resulting new tetrahedra are in **Case 2** [S1].

    *   **Step 2 Subdivision:**
        *   Applies to tetrahedra in **Case 2** [S1].
        *   Performs a similar bisection operation [S1].
        *   The resulting new tetrahedra are in **Case 3** [S1].

    *   **Step 3 Subdivision:**
        *   Applies to tetrahedra in **Case 3** [S1].
        *   Performs a similar bisection operation [S1].
        *   The resulting new tetrahedra recursively return to **Case 1** [S1].

    *   **Execution Order:**
        *   After Initial Subdivision, perform Step 1 [S1].
        *   After Step 1, perform Step 2 [S1].
        *   After Step 2, perform Step 3 [S1].
        *   If more vertices remain, return to Step 1 [S1].
        *   For a volume with parameter $N=n$, the total required steps are: $n$ Steps 1, $n$ Steps 2, and $n-1$ Steps 3 [S1].

4.  **Subdivision Strategies:**
    The recursive subdivision can be implemented in two ways [S1]:
    *   **Breadth Priority Subdivision (BPS):** All tetrahedra at the same level undergo the same subdivision step before moving to the next level [S1]. After each BPS step, all tetrahedra undergo the same subdivision (a tetrahedron breaks into two) [S1]. After one level is finished, the process proceeds to the next step [S1].
    *   **Depth Priority Subdivision (DPS):** A single tetrahedron descends through all steps before other tetrahedra are subdivided [S1].

5.  **Termination:**
    The subdivision continues until all vertices from the original volume grid are added to the framework [S1]. Regardless of the order (BPS or DPS), when subdivision stops, each original volume cell is divided into exactly **six tetrahedra** [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_multi_tetra_framework(volume_data, N):
    """
    Builds the Multi-Tetra Framework for a regular volume.
    
    Args:
        volume_data: 3D array of original volume data.
        N: Integer parameter for volume extension.
        
    Returns:
        tree: A list representing the full binary tree of tetrahedra.
    """
    dim = 2 * N + 1
    # [G] Pad volume_data to (dim, dim, dim) with zeros.
    
    # Linear array to store tetrahedra. Each element: [case, v0, v1, v2, v3, children_indices]
    # Initial subdivision generates 12 tetrahedra, all Case 1.
    # We initialize them with placeholder vertex indices that will be mapped later.
    tetrahedra = [{'case': 1, 'verts': [0]*4, 'children': [-1, -1]} for _ in range(12)]
    
    # Step sequence for N=n: n Steps 1, n Steps 2, n-1 Steps 3
    step_sequence = [1] * N + [2] * N + [3] * (N - 1)
    
    for step in step_sequence:
        # Collect tetrahedra of the current step's case
        current_tets = [t for t in tetrahedra if t['case'] == step]
        for tet in current_tets:
            # [Comment] Source does not specify which edge midpoint is chosen for each case.
            # Implementation must select the correct edge based on the tetrahedron's geometry/case.
            # The added vertex index is the average of the two endpoints' indices.
            v_a, v_b = tet['verts'][0], tet['verts'][1] # Placeholder edge
            mid_v = (v_a + v_b) // 2
            
            new_case = (step % 3) + 1
            # Create two child tetrahedra
            child1 = {'case': new_case, 'verts': [0]*4, 'children': [-1, -1]}
            child2 = {'case': new_case, 'verts': [0]*4, 'children': [-1, -1]}
            
            # Link children to parent
            idx1 = len(tetrahedra)
            idx2 = len(tetrahedra) + 1
            tetrahedra.append(child1)
            tetrahedra.append(child2)
            tet['children'] = [idx1, idx2]
            
    return tetrahedra
```

**Edge Cases and Pitfalls:**
*   **Vertex Uniqueness:** Each vertex from the original volume is added only once. The subdivision must track which vertices have been added to avoid duplication [S1].
*   **Case Classification:** The subdivision steps are strictly tied to the case of the tetrahedron (Case 1, 2, or 3). Applying the wrong step to a tetrahedron will result in an incorrect topology [S1].
*   **No Interpolation:** Since the partitioning process does not add new sample points, no interpolation is needed for sampling point values. This preserves the original data values at the vertices [S1].

## Subtopics

### [Error-Based Model Generation (Bottom-Up Fusion)](1306-error-based-model-generation-bottom-up-fusion.md)
This method performs a bottom-up simplification of the Multi-Tetra Framework to approximate high-resolution volume data with fewer tetrahedra, thereby reducing rendering time. It merges child tetrahedra into their parent node if the geometric approximation error remains within a specified threshold, solving the trade-off between detail and performance. Unlike top-down approaches, it starts from the leaf nodes and ascends, ensuring that fusion decisions are based on accumulated local errors.

## Operating characteristics
*   **Time Complexity:** The subdivision process involves a fixed number of steps based on $N$. For $N=n$, there are $3n-1$ steps. Each step processes a subset of tetrahedra. The total number of tetrahedra generated is proportional to the number of vertices in the extended grid, which is $(2N+1)^3$. The binary tree structure allows for $O(1)$ indexing [S1].
*   **Memory Complexity:** The framework is stored in a linear array representing a full binary tree. The size of the array is proportional to the number of tetrahedra, which is $6 \times L^3$ where $L$ is the original volume dimension (since each original cell becomes 6 tetrahedra) [S1].
*   **Performance:** The framework significantly reduces the number of voxels required to model an object while preserving the original topology [S1]. The error-based model generation allows for smooth changes in resolution across areas of different levels [S1].
*   **Strengths:**
    *   Solves the crack problem between different levels of detail [S1].
    *   No interpolation required for sampling point values [S1].
    *   Instant indexing via linear array storage [S1].
    *   Supports both BPS and DPS strategies [S1].
*   **Limitations:**
    *   Requires the volume to be extended to $(2N+1)^3$, which may increase memory usage for non-power-of-two dimensions [S1].
    *   The complexity of managing the three cases (1, 2, 3) and the specific subdivision steps may be complex to implement [S1].

## Background
The Multi-Tetra Framework was presented by Yong Zhou, Baoquan Chen, and Arie Kaufman from the Center for Visual Computing and Department of Computer Science at the State University of New York at Stony Brook [S1]. It extends Lindstrom's subdivision mechanism to the 3D case, using tetrahedra as the basic operation element to recursively build a uniform framework [S1]. The framework is designed for visualizing regular volume data and allows for the generation of an error-based model (EBM) that approximates the volume within given precisions [S1].

## Sources
[S1] Multiresolution tetrahedral framework for visualizing regular volume data, https://doi.org/10.5555/266989.267040, fetched 2026-10-30
