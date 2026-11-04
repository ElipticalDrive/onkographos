# Point Plane Test Traversal

Also called: Point-Plane Tests for In-Element Lookup, Tetrahedral Refinement Tree Traversal

## Overview
Point Plane Test Traversal is a method for locating the specific child tetrahedron within a TET-AMR (Tetrahedral Adaptive Mesh Refinement) refinement tree that contains a given query point [S1, unofficial]. It solves the in-element lookup problem by traversing from the coarse root to the leaf (or max depth) using geometric half-space tests against planes defined by edge midpoints, avoiding the redundancy of checking all eight children individually [S1, unofficial]. It is distinguished from [Leaf Collection](899-leaf-collection.md) by its on-the-fly vertex computation and from [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by its reliance on explicit geometric plane tests rather than incremental rasterization logic [S1, unofficial].

## How it works
**Inputs:**
*   `coarseTetID`: The element ID of the root tetrahedron.
*   `childCounts`: An array where `childCounts[i]` stores the number of children of element `i`.
*   `maxDepth`: The maximum traversal depth.
*   `point`: The 3D query point $(x, y, z)$.
*   `GetCoarseVertices(tetID)`: A function returning the four corner vertices of the coarse tetrahedron.

**Outputs:**
*   `tetID`: The global element ID of the located child tetrahedron.
*   `V`: The ordered set of four vertices $\{t, l, r, b\}$ of the located tetrahedron.

**Data Structures:**
*   Vertices are represented as 3D vectors. The order is fixed as Top ($t$), Left ($l$), Right ($r$), Back ($b$).
*   `childCounts` is an integer array indexed by element ID.

**Definitions:**
*   $halfSpace(P, A, B, C)$: Returns `true` if point $P$ is in the "front" half-space of the plane defined by points $A, B, C$. The normal direction is determined by the right-hand rule applied to vectors $(B-A)$ and $(C-A)$. Specifically, it checks if the dot product of the normal $\vec{n} = (B-A) \times (C-A)$ and the vector $(P-A)$ is positive.
*   $RB$: Boolean result of the first plane test.
*   $TB$: Boolean result of the second plane test.
*   $inner$: Boolean result indicating if the point is in the "inner" child (the central tetrahedron formed by midpoints).

**Algorithm Steps:**

1.  Initialize `rLevel` to 0.
2.  Set `tetID` to `coarseTetID`.
3.  Retrieve the initial vertex set $V$ using `GetCoarseVertices(tetID)`.
4.  Loop while `childCounts[tetID] > 0` AND `rLevel <= maxDepth`:
    a.  Extract vertices $t, l, r, b$ from $V$.
    b.  Compute $RB$: Test if `point` is in the front half-space of the plane passing through midpoints $M_{TR} = \frac{t+r}{2}$, $M_{LB} = \frac{l+b}{2}$, and $M_{TB} = \frac{t+b}{2}$.
        $$ RB = halfSpace(point, M_{TR}, M_{LB}, M_{TB}) $$
    c.  Compute $TB$: Test if `point` is in the front half-space of the plane passing through midpoints $M_{TR} = \frac{t+r}{2}$, $M_{LB} = \frac{l+b}{2}$, and $M_{TL} = \frac{t+l}{2}$.
        $$ TB = halfSpace(point, M_{TR}, M_{LB}, M_{TL}) $$
    d.  Compute $inner$: Perform four half-space tests and AND the results. The tests check if the point is on the same side of the planes forming the inner tetrahedron as the interior of the parent.
        $$ inner = halfSpace(point, M_{TL}, M_{TB}, M_{TR}) \ \& \ halfSpace(point, M_{LT}, M_{LR}, M_{LB}) \ \& \ halfSpace(point, M_{RT}, M_{RB}, M_{RL}) \ \& \ halfSpace(point, M_{BT}, M_{BL}, M_{BR}) $$
        *Note: The source lists specific midpoint combinations for these four tests. To ensure self-containment, the midpoints are defined as:*
        *   Test 1: Plane through $\frac{t+l}{2}, \frac{t+b}{2}, \frac{t+r}{2}$.
        *   Test 2: Plane through $\frac{l+t}{2}, \frac{l+r}{2}, \frac{l+b}{2}$.
        *   Test 3: Plane through $\frac{r+t}{2}, \frac{r+b}{2}, \frac{r+l}{2}$.
        *   Test 4: Plane through $\frac{b+t}{2}, \frac{b+l}{2}, \frac{b+r}{2}$.
    e.  Calculate `localChildNo` using the formula:
        $$ localChildNo = (RB \times 4) + ((RB == TB) \times 2) + inner $$
        *Where boolean values are treated as 1 for true and 0 for false.*
    f.  Update $V$ to the child vertices using `CalculateTetrahedronVertices(V, localChildNo)`.
    g.  Update `tetID` to the child ID using `CalculateChildID(tetID, localChildNo, childCounts)`.
    h.  Increment `rLevel`.
5.  Return `tetID` and $V$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def halfSpace(point, A, B, C):
    """
    Returns True if point is in the front half-space of plane ABC.
    Normal n = (B - A) x (C - A).
    Returns True if dot(n, point - A) > 0.
    """
    BA = B - A
    CA = C - A
    normal = cross(BA, CA)
    PA = point - A
    return dot(normal, PA) > 0

def calculate_tetrahedron_vertices(V, childNo):
    """
    Computes the vertex coordinates of a child tetrahedron given the parent vertices V
    and the local child number (0-7).
    V is a tuple/list of 4 vectors: [t, l, r, b]
    """
    t, l, r, b = V
    midpoints = {
        'tl': (t + l) / 2,
        'tr': (t + r) / 2,
        'tb': (t + b) / 2,
        'lr': (l + r) / 2,
        'lb': (l + b) / 2,
        'rb': (r + b) / 2
    }
    
    if childNo == 0:
        return [t, midpoints['tl'], midpoints['tr'], midpoints['tb']]
    elif childNo == 1:
        return [midpoints['lb'], midpoints['tb'], midpoints['tr'], midpoints['tl']]
    elif childNo == 2:
        return [l, midpoints['tl'], midpoints['lb'], midpoints['lr']]
    elif childNo == 3:
        return [midpoints['tr'], midpoints['lr'], midpoints['lb'], midpoints['tl']]
    elif childNo == 4:
        return [r, midpoints['rb'], midpoints['tr'], midpoints['lr']]
    elif childNo == 5:
        return [midpoints['lb'], midpoints['lr'], midpoints['tr'], midpoints['rb']]
    elif childNo == 6:
        return [b, midpoints['rb'], midpoints['lb'], midpoints['tb']]
    elif childNo == 7:
        return [midpoints['tr'], midpoints['tb'], midpoints['lb'], midpoints['rb']]
    else:
        raise ValueError("Invalid childNo")

def calculate_child_id(tetID, childNo, childCounts):
    """
    Calculates the global tetrahedron ID of a child element.
    """
    current_id = tetID + 1
    for i in range(childNo + 1):
        # Add the count of children of the current sibling plus 1 for the sibling itself
        # The loop runs childNo + 1 times to skip 'childNo' siblings before the target
        # Note: The source algorithm loops i from 0 to childNo.
        # Inside the loop: tetID += childCounts[tetID] + 1
        # This effectively skips over the previous children blocks.
        current_id = current_id + childCounts[current_id] + 1
    return current_id

def point_plane_traversal(coarseTetID, childCounts, maxDepth, point, get_coarse_vertices):
    rLevel = 0
    tetID = coarseTetID
    V = get_coarse_vertices(tetID)
    
    while childCounts[tetID] > 0 and rLevel <= maxDepth:
        t, l, r, b = V
        
        # Midpoints
        m_tr = (t + r) / 2
        m_lb = (l + b) / 2
        m_tb = (t + b) / 2
        m_tl = (t + l) / 2
        
        # RB half: Plane through TR, LB, TB
        RB = halfSpace(point, m_tr, m_lb, m_tb)
        
        # TB half: Plane through TR, LB, TL
        TB = halfSpace(point, m_tr, m_lb, m_tl)
        
        # Inner tests
        # Test 1: (t+l)/2, (t+b)/2, (t+r)/2
        inner1 = halfSpace(point, (t+l)/2, (t+b)/2, (t+r)/2)
        # Test 2: (l+t)/2, (l+r)/2, (l+b)/2
        inner2 = halfSpace(point, (l+t)/2, (l+r)/2, (l+b)/2)
        # Test 3: (r+t)/2, (r+b)/2, (r+l)/2
        inner3 = halfSpace(point, (r+t)/2, (r+b)/2, (r+l)/2)
        # Test 4: (b+t)/2, (b+l)/2, (b+r)/2
        inner4 = halfSpace(point, (b+t)/2, (b+l)/2, (b+r)/2)
        
        inner = inner1 and inner2 and inner3 and inner4
        
        # Calculate local child number
        # RB and TB are booleans. Convert to int (1 or 0)
        rb_int = 1 if RB else 0
        tb_int = 1 if TB else 0
        inner_int = 1 if inner else 0
        
        localChildNo = (rb_int * 4) + ((rb_int == tb_int) * 2) + inner_int
        
        V = calculate_tetrahedron_vertices(V, localChildNo)
        tetID = calculate_child_id(tetID, localChildNo, childCounts)
        rLevel += 1
        
    return tetID, V
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point errors in `halfSpace` tests can lead to incorrect quadrant identification if the point lies exactly on a plane. [G] A small epsilon tolerance should be considered for robustness, though the source does not specify one.
*   **Termination:** The loop terminates if a leaf is reached (`childCounts[tetID] == 0`), `maxDepth` is exceeded, or (in the broader rendering context) if the tetrahedron size on screen falls below a threshold. The core traversal algorithm returns the ID and vertices of the last traversed level in these cases [S1, unofficial].
*   **Vertex Order:** The correctness of the plane tests depends strictly on the consistent ordering of vertices $(t, l, r, b)$.

## Subtopics

### Calculate Tetrahedron Vertices
Computes the vertex coordinates of a child tetrahedron given the parent vertices $V$ and the local child number (0-7). It maps the local child index to a specific subset of the parent's vertices and the midpoints of its edges. The mapping is deterministic: child 0 is the top corner; child 7 is the central inner tetrahedron formed by all midpoints [S1, unofficial].

### Calculate Child ID
Calculates the global tetrahedron ID of a child element given the parent ID, the local child number, and the `childCounts` array. It iterates through the siblings of the target child, skipping over their respective subtrees (counted by `childCounts`) to find the correct global index. This allows the traversal to maintain a flat array representation of the tree [S1, unofficial].

### Exact Cell Sampling
A method that uses refinement tree traversal to find the vertices of neighbor elements for accurate cell-centered sampling. It relies only on coarse mesh neighborhood information. It finds which coarse tetrahedron the neighbor is in, then uses the traversal process to find the vertices of the element to sample from, ensuring exact geometric alignment unlike approximate methods [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(D)$ per query, where $D$ is the traversal depth. Each level requires a constant number of plane tests (6 total: 2 for quadrants, 4 for inner/outer, though the source notes a single test suffices once the quadrant is known, the algorithm provided uses 4 for conciseness) and arithmetic operations.
*   **Memory Complexity:** $O(1)$ auxiliary space per traversal step, as vertex coordinates are computed on-the-fly and not stored for lower levels. The `childCounts` array is required for ID calculation.
*   **Performance:** Faster than naive containment tests against all 8 children because it prunes the search space using geometric properties of red-refinement tetrahedra [S1, unofficial].
*   **Strengths:** No need to store coordinates for refined elements; efficient lookup for sparse data.
*   **Limitations:** Requires the `childCounts` array and a method to retrieve coarse vertices. Precision issues may arise near plane boundaries.

## Background
The method originates from work on Direct Volume Rendering of Tree-Based Tetrahedral Adaptive Mesh Refinement (TET-AMR) data [S1, unofficial]. It was developed to address the inefficiency of naive in-element lookups in adaptive tetrahedral grids, leveraging the regular geometry of red-refinement tetrahedra to optimize the traversal process [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
