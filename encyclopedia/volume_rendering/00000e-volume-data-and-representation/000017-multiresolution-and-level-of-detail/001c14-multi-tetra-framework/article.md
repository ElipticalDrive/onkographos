# Multi-Tetra Framework

Also called: BPS, DPS, Initial Subdivision, Multi-Tetra Framework Subdivision, Multiresolution Tetrahedral Framework, Recursive Tetrahedral Subdivision

## Overview
The Multi-Tetra Framework is a multiresolution representation that approximates regular volume data using tetrahedra of varying levels of detail, generated through recursive subdivision and stored in a binary tree structure [S1]. It solves the crack problem between different levels of detail by ensuring that the subdivision process only uses existing vertex indices without adding new sample points or requiring interpolation [S1]. Unlike sibling methods that rely on fixed grid resolutions or texture mipmaps, this framework allows for smooth changes in resolution across areas of different levels within the same framework by enabling error-based fusion of tetrahedra from multiple levels [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A regular volume data set with dimensions $L \times L \times L$.
*   **Output:** A multiresolution tetrahedral framework represented by a full binary tree, where each node corresponds to a tetrahedron. The framework supports the generation of an Error-Based Model (EBM) by selectively fusing tetrahedra [S1].

**Data Structures:**
The framework is represented by a full binary tree and stored in a linear array to allow for instant indexing [S1].
*   **Vertex Indexing:** Vertices are identified by their grid indices $(i, j, k)$. The linear index $V$ for a vertex is calculated as:
    $$ V = i + j(2N + 1) + k(2N + 1)^2 $$
    where $N$ is a parameter defining the volume extension (see Parameters).
*   **Tetrahedron Structure:** Each node in the binary tree represents a tetrahedron. Since the subdivision is topological (based on vertex indices), the physical coordinates are not stored in the tree structure itself but are derived from the vertex indices and the original volume geometry [S1].

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
    The subdivision is a vertex-adding process where each step picks a vertex from the original volume and inserts it into the framework, dividing a tetrahedron into two. Each vertex is added only once [S1]. The process involves three specific steps (Step 1, Step 2, Step 3) that cycle based on the configuration of the tetrahedra [S1].

    *   **Step 1 Subdivision:**
        *   Applies to tetrahedra in **Case 1**.
        *   Adds the midpoint of an edge of the tetrahedron. The point added has an index equal to the average of the index values of the two endpoints [S1].
        *   The tetrahedron is bisected into two by the plane passing through the added point and its opposite edge [S1].
        *   The resulting new tetrahedra are in **Case 2** [S1].

    *   **Step 2 Subdivision:**
        *   Applies to tetrahedra in **Case 2**.
        *   Performs a similar bisection operation [S1].
        *   The resulting new tetrahedra are in **Case 3** [S1].

    *   **Step 3 Subdivision:**
        *   Applies to tetrahedra in **Case 3**.
        *   Performs a similar bisection operation [S1].
        *   The resulting new tetrahedra recursively return to **Case 1** [S1].

    *   **Execution Order:**
        *   After Initial Subdivision, perform Step 1.
        *   After Step 1, perform Step 2.
        *   After Step 2, perform Step 3.
        *   If more vertices remain, return to Step 1 [S1].
        *   For a volume with parameter $N=n$, the total required steps are: $n$ Steps 1, $n$ Steps 2, and $n-1$ Steps 3 [S1].

4.  **Subdivision Strategies:**
    The recursive subdivision can be implemented in two ways [S1]:
    *   **Breadth Priority Subdivision (BPS):** All tetrahedra at the same level undergo the same subdivision step before moving to the next level. After each BPS step, all tetrahedra undergo the same subdivision (a tetrahedron breaks into two). After one level is finished, the process proceeds to the next step [S1].
    *   **Depth Priority Subdivision (DPS):** A single tetrahedron descends through all steps (1, 2, 3) before other tetrahedra are subdivided [S1].

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
    # 1. Volume Extension
    # Extend volume to (2N+1)^3 by padding with zeros
    extended_dim = 2 * N + 1
    extended_volume = pad_volume(volume_data, extended_dim)
    
    # 2. Initial Subdivision
    # Create the root bounding box (Case 1)
    # The root represents the entire volume box
    root_tetra = create_bounding_box_tetra(extended_dim)
    
    # Add center point and split into 6 pyramids, then 12 tetrahedra
    # This is a topological operation based on vertex indices
    initial_tetrahedra = initial_subdivision(root_tetra, extended_dim)
    
    # Store in a linear array (binary tree representation)
    # Each node is a tetrahedron. Children are the two sub-tetrahedra.
    tree = initial_tetrahedra
    
    # 3. Recursive Subdivision
    # Determine the sequence of steps
    # For N=n: n Steps 1, n Steps 2, n-1 Steps 3
    step_sequence = []
    for _ in range(N):
        step_sequence.append(1)
    for _ in range(N):
        step_sequence.append(2)
    for _ in range(N - 1):
        step_sequence.append(3)
        
    # Process steps based on chosen strategy (BPS or DPS)
    # Here we show a generic iterative approach that processes all active tetrahedra
    # for a given step type, similar to BPS level-by-level processing
    
    current_level_tetrahedra = initial_tetrahedra
    
    for step_type in step_sequence:
        next_level_tetrahedra = []
        for tet in current_level_tetrahedra:
            if tet.is_case(step_type):
                # Perform subdivision for the specific case
                # Step 1: Case 1 -> Case 2
                # Step 2: Case 2 -> Case 3
                # Step 3: Case 3 -> Case 1
                child1, child2 = subdivide_tetra(tet, step_type, extended_dim)
                
                # Add children to the tree
                # In a linear array representation, indices are managed automatically
                # or explicitly calculated based on parent index
                tree.append(child1)
                tree.append(child2)
                
                next_level_tetrahedra.append(child1)
                next_level_tetrahedra.append(child2)
            else:
                # If the tetrahedron is not in the required case for this step,
                # it remains as is (or is processed in a different branch in DPS)
                # For BPS, we only process those matching the current step's case
                # Note: The source implies a strict cycle. In practice, 
                # the "active" set for a step consists of tetrahedra in the 
                # corresponding case.
                pass
        
        current_level_tetrahedra = next_level_tetrahedra
        
    return tree

def pad_volume(volume, target_dim):
    """
    Pads the volume to the target dimension with zeros.
    """
    # Implementation details for padding are not specified in the source
    # [G] Assuming standard zero-padding to reach (2N+1)^3
    pass

def create_bounding_box_tetra(dim):
    """
    Creates the initial bounding box tetrahedron (Case 1).
    """
    # Returns a tetrahedron structure representing the whole volume box
    pass

def initial_subdivision(root_tetra, dim):
    """
    Performs the initial subdivision:
    1. Add center point.
    2. Connect to box vertices -> 6 pyramids.
    3. Split pyramids -> 12 tetrahedra (Case 1).
    """
    # Returns list of 12 tetrahedra
    pass

def subdivide_tetra(tet, step_type, dim):
    """
    Subdivides a tetrahedron based on its case and the step type.
    """
    # Step 1: Case 1 -> Case 2
    # Step 2: Case 2 -> Case 3
    # Step 3: Case 3 -> Case 1
    # The subdivision adds the midpoint of an edge.
    # The point added has an index equal to the average of the index values 
    # of the two endpoints.
    # The tetrahedron is bisected into two by the plane passing through 
    # the added point and its opposite edge.
    
    # [G] The specific edge to bisect and the resulting vertex indices 
    # depend on the geometric configuration of Case 1, 2, or 3.
    # The source states: "The partitioning operation involves only vertex indices 
    # and not the physical coordinates."
    
    # Calculate the index of the midpoint
    # Let the endpoints of the edge be V1 and V2
    # Midpoint Index = (Index(V1) + Index(V2)) / 2
    
    # Create two new tetrahedra using the midpoint and the opposite edge
    # Return the two new tetrahedra structures
    pass
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
