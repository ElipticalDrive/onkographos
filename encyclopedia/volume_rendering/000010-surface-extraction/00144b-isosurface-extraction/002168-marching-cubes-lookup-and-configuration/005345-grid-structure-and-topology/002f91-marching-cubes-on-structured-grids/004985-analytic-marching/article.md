# Analytic Marching

Also called: Analytic Meshing

## Overview
Analytic Marching (AM) is a meshing algorithm that extracts the exact zero-level isosurface of an implicit function represented by a ReLU-based Multi-Layer Perceptron (MLP) as a polygon mesh, without relying on discrete 3D grid sampling [S1]. Unlike [Marching Cubes on Structured Grids](6051-marching-cubes-on-structured-grids.md), whose accuracy depends on discretization resolution, AM exploits the piecewise linear nature of ReLU networks to identify linear regions intersected by the surface and solve for vertices analytically, making its precision independent of grid resolution [S1].

## How it works
**Inputs:** An implicit surface field function $F: \mathbb{R}^3 \to \mathbb{R}$ implemented by a ReLU-based MLP $T$ [S1].
**Outputs:** A polygon mesh $Z = \{V, \{P\}\}$ representing the exact zero-level isosurface, where $V$ is the set of vertices and $\{P\}$ is the set of polygon faces [S1].

**Data Structures:**
*   **MLP Parameters:** Weights $W_l \in \mathbb{R}^{n_l \times n_{l-1}}$ for layers $l=1, \dots, L$. Biases are omitted for notational simplicity [S1].
*   **State Vector:** $s \in \{0, 1\}^N$, where $N = \sum_{l=1}^L n_l$ is the total number of hidden neurons. Each element corresponds to a neuron's activation state in a linear region [S1].
*   **Active Set $S$:** A set of state vectors $s(\tilde{r})$ representing analytic cells currently being processed [S1].
*   **Inactive Set $\bar{S}$:** A set of state vectors representing processed cells [S1].
*   **Boundary Planes:** For a region $r$, boundary planes $H_{lk}^r$ are defined by the condition that the $k$-th neuron in layer $l$ has zero pre-activation [S1].

**Formulas:**
The implicit surface is defined as the zero-level set:
$$ \{ \mathbf{x} \in \mathbb{R}^3 \mid F(\mathbf{x}) = 0, |\nabla_{\mathbf{x}} F(\mathbf{x})| \neq 0 \} \quad (1) $$
The constraint $|\nabla_{\mathbf{x}} F(\mathbf{x})| \neq 0$ ensures a non-degenerate isosurface [S1].

The MLP computes $T\mathbf{x} = g(W_L \dots g(W_1 \mathbf{x}))$, where $g$ is the ReLU activation [S1]. The pre-activation of the $k$-th neuron in layer $l$ is:
$$ a_{lk}(\mathbf{x}) = \pi_k \mathbf{W}_l \mathbf{g}(\mathbf{W}_{l-1} \dots \mathbf{g}(\mathbf{W}_1 \mathbf{x})) $$
where $\pi_k$ selects the $k$-th row of the matrix [S1].

Within a linear region $r$, the neuron-wise linear mapping $a_{lk}^r$ is:
$$ a_{lk}^r = \pi_k \mathbf{W}_l \prod_{i=1}^{l-1} \mathbf{W}_i^r \quad \text{for } l > 1, \quad a_{lk}^r = \pi_k \mathbf{W}_l \quad \text{for } l = 1 $$
where $\mathbf{W}_i^r$ represents the effective weight matrix in region $r$ [S1].

The state $s(\mathbf{x})$ is computed based on the signs of the pre-activations [S1].

**Parameters:**
*   $n_0 = 3$: Input dimensionality [S1].
*   $\tau = 5 \times 10^{-3}$: Default F-score distance threshold for evaluation [S1].

**Algorithm Steps:**
1.  Initialize active set $S = \emptyset$ and inactive set $\bar{S} = \emptyset$ [S1].
2.  Identify a starting point $\mathbf{x} \in \mathbb{R}^3$ such that $F(\mathbf{x}) \approx 0$ and $\nabla_{\mathbf{x}} F(\mathbf{x}) \neq 0$. This is typically done using a triggering scheme (e.g., Dichotomy, Sphere Tracing, or SGD) [S1]. Note: $\mathbf{x}$ does not need to satisfy $F(\mathbf{x})=0$ exactly, only sufficiently small to ensure it falls within an analytic cell [S1].
3.  Compute the state $s(\mathbf{x})$ corresponding to the analytic cell containing $\mathbf{x}$ and push $s(\mathbf{x})$ into $S$ [S1].
4.  While $S \neq \emptyset$:
    a.  Pop an active state $s(\tilde{r})$ from $S$ [S1].
    b.  Let $P_{\tilde{r}}^F$ be the analytic face in cell $\tilde{r}$. Enumerate all pairs of boundary planes $(H_{lk}^{\tilde{r}}, H_{l'k'}^{\tilde{r}})$ from the set of boundary planes $\{H_{lk}^{\tilde{r}}\}$ defined by the neuron zero-crossings, where $l, l' \in \{1, \dots, L\}$ and $k, k' \in \{1, \dots, n_l\}$ [S1].
    c.  For each pair, solve the $3 \times 3$ linear system to find the intersection vertex $\mathbf{v}$:
        $$ [a_{lk}^{\tilde{r}}; a_{l'k'}^{\tilde{r}}; a_F^{\tilde{r}}] \mathbf{v} = 0 \quad (16) $$
        Here, $a_F^{\tilde{r}}$ is the linear coefficient vector for the function $F$ in region $\tilde{r}$ [S1].
    d.  Verify the validity of $\mathbf{v}$ by checking if it satisfies the boundary conditions of the cell $C_{\tilde{r}}^F$ (i.e., it lies within the valid region defined by the state $s(\tilde{r})$) [S1].
    e.  Collect all valid vertices $\mathbf{v}$ to form the vertex set $V_{\tilde{r}}^P$ for face $P_{\tilde{r}}^F$. Add these vertices to the global vertex set $V$ and add the face $P_{\tilde{r}}^F$ to the output mesh $Z$ [S1].
    f.  Identify all boundary planes $H_{c_{\tilde{r}}lk}$ of the current cell that yielded valid vertices. For each such plane, infer the state $s(\tilde{r}_{\text{Neighbor}})$ of the neighboring analytic cell by flipping the bit $s(\tilde{r})_{lk}$ corresponding to that neuron [S1].
    g.  Move $s(\tilde{r})$ from $S$ to $\bar{S}$. Push any newly inferred neighbor states $s(\tilde{r}_{\text{Neighbor}})$ that are not already in $S \cup \bar{S}$ into $S$ [S1].
5.  End while loop [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def analytic_marching(MLP_weights, layer_widths):
    """
    Extracts the zero-level isosurface of a ReLU-MLP.
    
    Args:
        MLP_weights: List of weight matrices W_l.
        layer_widths: List of layer widths n_l.
        
    Returns:
        V: List of vertices (3D coordinates).
        Z: List of triangular faces (list of vertex indices).
    """
    L = len(MLP_weights)
    N = sum(layer_widths)
    V = []
    Z = []
    S = set()
    S_bar = set()

    # Step 2: Triggering (Dichotomy recommended [S1])
    x = find_surface_point_dichotomy(MLP_weights, layer_widths)

    # Step 3: Compute initial state
    s_x = compute_state(x, MLP_weights, layer_widths)
    S.add(s_x)

    while S:
        s_r = S.pop()

        # Step 4b: Enumerate boundary plane pairs
        # Boundary planes correspond to neurons with zero pre-activation.
        # In the linear region defined by s_r, these are the neurons where the state bit is 0.
        zero_neurons = []
        idx = 0
        for l in range(L):
            for k in range(layer_widths[l]):
                if s_r[idx] == 0:
                    zero_neurons.append((l, k))
                idx += 1

        face_vertices = []

        # Step 4c: Solve for vertices
        for i in range(len(zero_neurons)):
            for j in range(i + 1, len(zero_neurons)):
                l1, k1 = zero_neurons[i]
                l2, k2 = zero_neurons[j]

                a_lk_r = compute_linear_coefficient(MLP_weights, l1, k1, s_r)
                a_l2k2_r = compute_linear_coefficient(MLP_weights, l2, k2, s_r)
                a_F_r = compute_function_linear_coefficient(MLP_weights, layer_widths, s_r)

                # Solve [a_lk; a_l'k'; a_F] * v = 0
                v = solve_homogeneous_system([a_lk_r, a_l2k2_r, a_F_r])

                # Step 4d: Validate vertex
                if v is not None and is_valid_vertex(v, s_r, MLP_weights, layer_widths):
                    face_vertices.append(v)

        # Step 4e: Add face
        if face_vertices:
            face_indices = []
            for v in face_vertices:
                if v not in V:
                    V.append(v)
                face_indices.append(V.index(v))
            # Triangulate polygon
            for i in range(1, len(face_indices) - 1):
                Z.append([face_indices[0], face_indices[i], face_indices[i+1]])

        # Step 4f & 4g: March to neighbors
        for (l, k) in zero_neurons:
            new_s = list(s_r)
            idx = 0
            for ll in range(L):
                for kk in range(layer_widths[ll]):
                    if ll == l and kk == k:
                        new_s[idx] = 1 - new_s[idx]
                    idx += 1
            new_s_tuple = tuple(new_s)
            if new_s_tuple not in S and new_s_tuple not in S_bar:
                S.add(new_s_tuple)

        S_bar.add(s_r)

    return V, Z

def compute_state(x, MLP_weights, layer_widths):
    """Compute the binary state vector s(x) for point x."""
    s = []
    current_x = x
    idx = 0
    for l, W in enumerate(MLP_weights):
        pre_act = [sum(W[i][j] * current_x[j] for j in range(len(current_x))) for i in range(len(W))]
        current_x = [max(0, p) for p in pre_act]
        for p in pre_act:
            s.append(1 if p > 0 else 0)
            idx += 1
    return tuple(s)

def compute_linear_coefficient(MLP_weights, l, k, s_r):
    """Compute a_lk^r for neuron k in layer l in region r."""
    # a_lk^r = pi_k W_l prod_{i=1}^{l-1} W_i^r
    # W_i^r is W_i masked by the state of layer i
    current_vec = [1.0, 0.0, 0.0] # Input dimension n0=3
    for i in range(l):
        W = MLP_weights[i]
        # Apply state mask: if neuron j is inactive (0), zero out its output
        masked_vec = []
        vec_idx = 0
        for row in W:
            # pre-activation for this neuron
            pre = sum(row[j] * current_vec[j] for j in range(len(current_vec)))
            if s_r[vec_idx] == 0:
                masked_vec.append(0.0)
            else:
                masked_vec.append(pre)
            vec_idx += 1
        current_vec = masked_vec
    # Extract k-th neuron's linear coefficient vector
    W_l = MLP_weights[l]
    return [W_l[k][j] * current_vec[j] for j in range(len(current_vec))]

def compute_function_linear_coefficient(MLP_weights, layer_widths, s_r):
    """Compute the linear coefficient vector a_F^r for the output function F."""
    # Propagate through all layers to get the final linear map to scalar output
    current_vec = [1.0, 0.0, 0.0]
    idx = 0
    for l, W in enumerate(MLP_weights):
        masked_vec = []
        for row in W:
            pre = sum(row[j] * current_vec[j] for j in range(len(current_vec)))
            if s_r[idx] == 0:
                masked_vec.append(0.0)
            else:
                masked_vec.append(pre)
            idx += 1
        current_vec = masked_vec
    # Final output layer is typically a single neuron; return its weights
    return current_vec

def solve_homogeneous_system(norms):
    """Solve [n1; n2; n3] * v = 0 using cross products."""
    def cross(a, b):
        return [a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0]]
    def dot(a, b):
        return sum(a[i]*b[i] for i in range(3))
    def norm_sq(a):
        return sum(a[i]*a[i] for i in range(3))
    
    d = cross(norms[0], norms[1])
    if norm_sq(d) < 1e-9:
        return None
    if abs(dot(norms[2], d)) > 1e-9:
        return None
    # Normalize
    length = d[0]**0.5 + d[1]**0.5 + d[2]**0.5
    if length < 1e-9:
        return None
    return [d[i] / length for i in range(3)]

def is_valid_vertex(v, s_r, MLP_weights, layer_widths):
    """Check if v satisfies the boundary conditions of cell C_r^F."""
    current_x = v
    idx = 0
    for l, W in enumerate(MLP_weights):
        pre_act = [sum(W[i][j] * current_x[j] for j in range(len(current_x))) for i in range(len(W))]
        current_x = [max(0, p) for p in pre_act]
        for p in pre_act:
            if s_r[idx] == 1 and p < -1e-6:
                return False
            if s_r[idx] == 0 and p > 1e-6:
                return False
            idx += 1
    return True

def find_surface_point_dichotomy(MLP_weights, layer_widths):
    """Find a point x such that F(x) ~ 0 using Dichotomy [S1]."""
    import random
    def eval_F(x):
        current = x
        for W in MLP_weights:
            pre = [sum(W[i][j] * current[j] for j in range(3)) for i in range(len(W))]
            current = [max(0, p) for p in pre]
        return current[0]
    
    for _ in range(100):
        origin = [random.uniform(-10, 10) for _ in range(3)]
        direction = [random.uniform(-1, 1) for _ in range(3)]
        length = (direction[0]**2 + direction[1]**2 + direction[2]**2)**0.5
        direction = [d/length for d in direction]
        
        t_min, t_max = 0.0, 20.0
        f_min, f_max = eval_F([origin[i] + t_min*direction[i] for i in range(3)]), \
                       eval_F([origin[i] + t_max*direction[i] for i in range(3)])
        
        if f_min * f_max > 0:
            continue
            
        for _ in range(30):
            t_mid = (t_min + t_max) / 2.0
            x_mid = [origin[i] + t_mid*direction[i] for i in range(3)]
            f_mid = eval_F(x_mid)
            if abs(f_mid) < 1e-4:
                return x_mid
            if f_min * f_mid < 0:
                t_max, f_max = t_mid, f_mid
            else:
                t_min, f_min = t_mid, f_mid
        return [origin[i] + t_mid*direction[i] for i in range(3)]
    return [0.0, 0.0, 0.0]
```

**Edge Cases and Pitfalls:**
*   **Degenerate Surfaces:** The method requires $|\nabla_{\mathbf{x}} F(\mathbf{x})| \neq 0$. If the gradient vanishes, the zero-level set may not be a proper isosurface [S1].
*   **Triggering Failure:** For occupancy fields, SGD and Sphere Tracing fail to efficiently trigger AM; only Dichotomy converges successfully [S1].
*   **High Face Count:** AM produces meshes with extremely high face counts, requiring post-processing simplification [S1].
*   **Precision:** The identified point $\mathbf{x}$ does not need to satisfy $F(\mathbf{x})=0$ exactly, but must be sufficiently small to ensure it falls in an analytic cell [S1].

## Operating characteristics
**Complexity:**
*   Per analytic cell: $O(n^3 L^3)$, where $n$ is the number of neurons per layer and $L$ is the depth [S1].
*   Overall: $O((n/2)^{2(L-1)} n^5 L^3)$, exponential with respect to depth $L$ and polynomial with respect to width $n$ [S1].
*   This contrasts with Marching Cubes, where complexity depends on grid resolution, not function complexity [S1].

**Performance:**
*   On Nvidia GPUs, parallel marching with Sphere Tracing triggering for 1,024 points is nearly an order-of-magnitude faster than CPU-based SGD triggering [S1].
*   Mesh simplification preserving 10% of original faces results in negligible precision loss; 1% preservation maintains accuracy [S1].

**Strengths:**
*   Exact recovery of the surface captured by the MLP, independent of grid resolution [S1].
*   Higher accuracy than discrete methods (MC, MT, DC) for the same implicit function [S1].

**Limitations:**
*   High face counts complicate downstream processing [S1].
*   Complexity grows exponentially with network depth [S1].
*   Requires ReLU-based MLPs [S1].

## Background
Analytic Marching was introduced in the paper "Learning and Meshing From Deep Implicit Surface Networks Using an Efficient Implementation of Analytic Marching" [S1]. It addresses the limitation of discrete meshing algorithms like Marching Cubes, which suffer from a trade-off between sampling efficiency and recovery precision due to discretization [S1]. The method leverages the property that ReLU MLPs partition input space into linear regions, allowing the zero-level isosurface to be identified analytically [S1]. It is implemented in the publicly released AnalyticMesh package and tested on Nvidia GPUs [S1].

## Sources
[S1] Learning and Meshing From Deep Implicit Surface Networks Using an Efficient Implementation of Analytic Marching, https://doi.org/10.1109/tpami.2021.3135007, fetched 2026-10-26
