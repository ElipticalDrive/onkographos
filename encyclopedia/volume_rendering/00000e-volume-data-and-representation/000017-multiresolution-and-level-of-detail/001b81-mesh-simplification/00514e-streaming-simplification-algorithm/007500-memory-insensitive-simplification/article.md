# Memory Insensitive Simplification

Also called: Cluster-based simplification, Lindstrom and Silva algorithm, OoCSx Simplification, Out-of-core simplification, memory insensitive mesh simplification

## Overview
Memory Insensitive Simplification is an out-of-core mesh simplification algorithm that reduces the resolution of a large triangle mesh by clustering vertices into octree nodes and computing representative positions via quadric error metrics, all while streaming data from disk to avoid loading the entire mesh into RAM [S1]. It differs from in-core methods by using external sorts to process triangles one by one, producing a simplified static mesh and vertex quadrics on disk, which serves as the input for subsequent octree construction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle soup $T_{in}$ (sequence of triplets of vertex coordinates) and a grid resolution parameter $n$.
*   **Output:** A triangle file $T_{out}$ containing simplified triangles and a vertex file $V_n$ containing quadric matrices, positions, and normals for the leaf level $n$ of the octree [S1].

**Data Structures:**
*   **Octcode:** A unique identifier for octree nodes. The root node has octcode $v=1$. The $k^{th}$ child of node $v$ is $8v + k$ with $k \in \{0, \dots, 7\}$ [S1]. Octcodes are ordered by level (top to bottom), and sibling nodes have consecutive octcodes [S1].
*   **Quadric Matrix ($Q_v$):** A symmetric $4 \times 4$ matrix encoding the weighted sum of squared distances to the triangle planes associated with a cluster [S1].
*   **Plane File ($P$):** A temporary on-disk file storing tuples of (octcode, weighted normal vector) [S1].
*   **Triangle File ($T_{out}$):** Stores tuples of (reversed octcode $v^{*t}$, vertex octcodes $v_1^t, v_2^t, v_3^t$) for triangles with distinct vertices [S1].
*   **Vertex File ($V_n$):** Stores tuples of (octcode $v$, quadric matrix $Q_v$, vertex position $p_v$, approximate normal $\hat{\mathbf{n}}_v$) for level $n$ [S1].

**Parameters:**
*   **Grid Resolution:** A uniform rectilinear grid with dimensions $2^n \times 2^n \times 2^n$ for some positive integer $n$, completely containing the input mesh [S1].
*   **Weighting:** Default relative weight $w_t = A_t$ (area of triangle $t$) [S1].

**Algorithm Steps:**

1.  **Initialization:** Define the grid resolution $n$. Initialize empty plane file $P$ and triangle file $T_{out}$ [S1].
2.  **Triangle Processing:** For each triangle $t = \langle p_1^t, p_2^t, p_3^t \rangle \in T_{in}$:
    a. Compute the implicit plane equation $\hat{\mathbf{n}}_t^T \mathbf{x} + d_t = 0$ and the normalized face normal $\hat{\mathbf{n}}_t$ [S1].
    b. Compute the weighted normal vector $\bar{\mathbf{n}}_t$:
       $$ \bar{\mathbf{n}}_t = \hat{w}_t \begin{pmatrix} \hat{\mathbf{n}}_t \\ d_t \end{pmatrix} $$
       where $\hat{w}_t$ is the normalized weight defined below [S1].
    c. For each vertex $p_i^t$ ($i=1,2,3$):
       i. Quantize the vertex coordinates to an integer grid cell location.
       ii. Convert this location to an octcode $v_i^t$ [S1].
       iii. Append the pair $(v_i^t, \bar{\mathbf{n}}_t)$ to the plane file $P$ [S1].
    d. If $v_1^t, v_2^t, v_3^t$ are distinct:
       i. Determine the octree node $v_t$ to assign the triangle to. $v_t$ is the lowest common ancestor of all pairs of vertices such that expanding $v_t$ would introduce the triangle $t$ [S1].
       ii. Compute the modified sort key (reversed octcode) $v^{*t}$ for $t$. This is done by inverting all bits lower than the most significant one bit of the octcode $v_t$ and negating the result. This orders nodes by increasing octcode within each level, but reverses inter-level order (leaves first) [S1].
       iii. Append $(v^{*t}, v_1^t, v_2^t, v_3^t)$ to the triangle file $T_{out}$ [S1].
3.  **External Sort:** Exhaust the input $T_{in}$. Externally sort the plane file $P$ on the octcode field $v$ [S1].
4.  **Quadric Accumulation:** For each unique octcode $v$ in the sorted plane file $P$:
    a. Initialize a $4 \times 4$ quadric matrix $Q_v$ to zero and an approximate normal $\hat{\mathbf{n}}_v$ to zero [S1].
    b. For each plane $\bar{\mathbf{n}}_t$ associated with octcode $v$:
       i. Add $\bar{\mathbf{n}}_t \bar{\mathbf{n}}_t^\top$ to $Q_v$ [S1].
       ii. Add $\hat{\mathbf{n}}_t$ to $\hat{\mathbf{n}}_v$ [S1].
    c. Compute the representative vertex position $p_v$ from $Q_v$ by minimizing the quadric error [S1].
    d. Append $(v, Q_v, p_v, \hat{\mathbf{n}}_v)$ to the vertex file $V_n$ [S1].
5.  **Cleanup:** Remove the temporary plane file $P$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def memory_insensitive_simplification(T_in, n):
    # T_in: list of triangles, each triangle is a list of 3 vertices (each vertex is a 3-tuple)
    # n: integer, grid resolution is 2^n x 2^n x 2^n
    
    P = [] # Plane file: list of (octcode, weighted_normal_4_vector)
    T_out = [] # Triangle file: list of (reversed_octcode, v1, v2, v3)
    
    # Helper: Compute octcode from grid coordinates
    # Assumes grid origin at (0,0,0) and cell size 1 for simplicity of mapping
    # In practice, quantization maps world coordinates to integer grid indices
    def world_to_octcode(point):
        # Quantize point to grid cell
        # This step depends on the specific grid bounding box and resolution n
        # For the purpose of this algorithm, we assume a function that returns
        # the octcode v for a given 3D point in the 2^n x 2^n x 2^n grid
        pass 

    # Helper: Compute reversed octcode
    def reversed_octcode(v):
        # Invert all bits lower than the most significant one bit and negate
        # Find most significant bit position
        msb = v.bit_length() - 1
        # Mask for bits lower than msb
        mask = (1 << msb) - 1
        # Invert bits lower than msb
        inverted = v ^ mask
        # Negate result (two's complement)
        return -inverted

    # Helper: Compute weighted normal
    def compute_weighted_normal(triangle):
        p1, p2, p3 = triangle
        # Compute face normal
        v1 = p2 - p1
        v2 = p3 - p1
        normal = cross(v1, v2)
        area = length(normal)
        if area == 0:
            return None, 0.0, 0.0
        
        # Normalize normal
        n_hat = normal / area
        
        # Plane equation: n_hat^T x + d = 0
        d = -dot(n_hat, p1)
        
        # Weight w_t = area (default)
        w_t = area
        
        # Normalized weight w_hat_t
        # This requires global sum of weights, which is not available in single pass
        # However, the formula for Q_v uses w_hat_t^2. 
        # The source states: w_hat_i^2 = (sum_i A_i)^2 * (w_i^2 / sum_i w_i^2)
        # This implies a two-pass approach or a scaling factor.
        # For the purpose of the algorithm description, we compute the unnormalized
        # plane vector and scale it later or assume a global scaling factor is applied.
        # The source pseudocode computes n_bar_t directly.
        # Let's assume we compute the raw plane vector and scale it by area.
        # The final Q_v formula uses n_bar_t * n_bar_t^T.
        
        # To strictly follow the source's "normalized weights" claim, 
        # we might need a first pass to compute sum(A_i). 
        # However, the source pseudocode does not show this. 
        # We will compute the weighted normal as area * (n_hat, d).
        # The quadric accumulation will then be scaled by area^2.
        
        weighted_normal_4 = w_t * np.array([n_hat[0], n_hat[1], n_hat[2], d])
        return weighted_normal_4, n_hat, area

    # Pass 1: Process triangles
    for t in T_in:
        weighted_n, n_hat, area = compute_weighted_normal(t)
        if weighted_n is None:
            continue
            
        p1, p2, p3 = t
        
        # Map vertices to octcodes
        v1 = world_to_octcode(p1)
        v2 = world_to_octcode(p2)
        v3 = world_to_octcode(p3)
        
        # Append to plane file
        P.append((v1, weighted_n))
        P.append((v2, weighted_n))
        P.append((v3, weighted_n))
        
        # Check for distinct vertices
        if v1 != v2 and v2 != v3 and v1 != v3:
            # Determine lowest common ancestor
            # This is complex to implement without an octree structure
            # The source says: "choose v_t as the lowest common ancestor of all pairs 
            # of vertices from t, such that expanding the node v_t would introduce the triangle t"
            # This requires checking if the triangle fits in the node.
            # For pseudocode completeness, we assume a function lca(v1, v2, v3) exists
            # that returns the appropriate octcode.
            v_t = lca(v1, v2, v3)
            
            v_star = reversed_octcode(v_t)
            T_out.append((v_star, v1, v2, v3))
            
    # External Sort P on octcode
    P.sort(key=lambda x: x[0])
    
    # Pass 2: Accumulate Quadrics
    V_n = []
    
    # Group planes by octcode
    current_v = None
    Q_v = np.zeros((4, 4))
    n_hat_v = np.zeros(3)
    
    for v, n_bar in P:
        if v != current_v:
            # Process previous octcode
            if current_v is not None:
                # Compute vertex position from Q_v
                # Minimize x^T Q_v x
                # This involves solving a linear system or eigenvalue decomposition
                # The source says "compute vertex position p_v from Q_v"
                p_v = compute_vertex_from_quadric(Q_v)
                
                # Normalize approximate normal
                if np.linalg.norm(n_hat_v) > 0:
                    n_hat_v = n_hat_v / np.linalg.norm(n_hat_v)
                else:
                    n_hat_v = np.array([0, 0, 1]) # Default
                    
                V_n.append((current_v, Q_v, p_v, n_hat_v))
            
            # Reset for new octcode
            current_v = v
            Q_v = np.zeros((4, 4))
            n_hat_v = np.zeros(3)
            
        # Add to quadric
        Q_v += np.outer(n_bar, n_bar)
        
        # Extract the 3D normal part from n_bar for approximate normal
        # n_bar = w_t * (n_hat, d)
        # We need the original n_hat. 
        # The source says "add n_hat_t to approximate normal n_hat_v"
        # But we only stored n_bar. 
        # We need to recover n_hat_t.
        # n_bar[0:3] = w_t * n_hat_t
        # So n_hat_t = n_bar[0:3] / w_t
        # But we don't have w_t stored separately.
        # However, n_hat_t is the direction of n_bar[0:3].
        # So we can just add the direction.
        n_bar_3d = n_bar[0:3]
        if np.linalg.norm(n_bar_3d) > 0:
            n_hat_t = n_bar_3d / np.linalg.norm(n_bar_3d)
            n_hat_v += n_hat_t
            
    # Process last octcode
    if current_v is not None:
        p_v = compute_vertex_from_quadric(Q_v)
        if np.linalg.norm(n_hat_v) > 0:
            n_hat_v = n_hat_v / np.linalg.norm(n_hat_v)
        else:
            n_hat_v = np.array([0, 0, 1])
        V_n.append((current_v, Q_v, p_v, n_hat_v))
        
    return T_out, V_n

def compute_vertex_from_quadric(Q):
    # Minimize x^T Q x
    # This is a standard problem. 
    # If Q is 4x4, it represents a quadric in homogeneous coordinates.
    # The minimum is found by solving the linear system derived from the gradient.
    # Let Q = [[A, b], [b^T, c]]
    # The minimum is at x = -A^{-1} b
    # If A is singular, use pseudo-inverse or regularization.
    A = Q[0:3, 0:3]
    b = Q[0:3, 3]
    try:
        p = np.linalg.solve(A, -b)
    except np.linalg.LinAlgError:
        # Fallback if A is singular
        p = np.zeros(3)
    return p

def lca(v1, v2, v3):
    # Placeholder for LCA logic
    # The source describes the logic but does not provide a simple formula.
    # It requires checking the octree structure.
    pass
```

**Edge Cases and Pitfalls:**
*   **Non-manifold Geometry:** Vertex clustering may introduce non-manifold vertices and edges. This is acceptable for interactive visualization where modest topology simplification eliminates topological noise [S1].
*   **Detail Loss:** If the cluster grid is not fine enough, the mesh is irreversibly decimated in the first phase, potentially losing fine details of the original mesh [S1].
*   **Sign Ambiguity:** The normal extracted from the quadric matrix has a sign ambiguity. This is resolved by using an approximate normal $\hat{\mathbf{n}}_v$ (sum of face normals) to determine the correct sign [S1]. If space is at a premium, $\hat{\mathbf{n}}_v$ can be avoided by performing additional computation and encoding the sign directly in the quadric matrix [S1].
*   **Weighting Units:** The normalized weights ensure that the quadric error has units of volume squared, regardless of the weighting scheme used [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the external sort of the plane file $P$, which is $O(N \log N)$ where $N$ is the number of triangles, assuming efficient external sorting [S1].
*   **Memory Complexity:** $O(1)$ in-core memory usage relative to the mesh size, as data is streamed from disk. The in-core memory is limited to the current triangle being processed and the buffers for the external sort [S1].
*   **Performance:** The algorithm avoids costly random disk accesses by using a sequence of external sorts followed by fast sequential accesses [S1].
*   **Strengths:** Suitable for very large meshes that do not fit in main memory. Produces a coherent streaming mesh suitable for subsequent octree construction [S1].
*   **Limitations:** The initial clustering is irreversible and may lose fine details if the grid resolution is too low [S1]. The algorithm produces a single-resolution static mesh; the octree construction is a separate phase [S1].

## Background
The Memory Insensitive Simplification algorithm, also known as OoCSx (Out-of-Core Simplification), was introduced by Lindstrom and Silva in 2001 [S1]. It is based on the vertex clustering method and uses external sorts to process triangle soups one by one. The algorithm was designed for the out-of-core construction and visualization of multiresolution surfaces, particularly for large datasets generated by scanning processes [S1]. It builds on the quadric error metric introduced by Garland and Heckbert [S1] and the octree representation for multiresolution meshes [S1].

## Sources
[S1] Out-of-Core Construction and Visualization of Multiresolution Surfaces, https://digital.library.unt.edu/ark:/67531/metadc1406579/?q=multiresolution volume rendering, fetched 2026-10-28
