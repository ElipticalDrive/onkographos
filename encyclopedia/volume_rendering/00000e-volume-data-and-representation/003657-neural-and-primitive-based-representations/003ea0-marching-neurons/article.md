# Marching Neurons

Also called: Edge Subdivision, ReLU-induced polyhedral complex construction

## Overview
Marching Neurons is an analytical isosurface extraction method that constructs the exact zero-level set of a ReLU-based neural implicit function by traversing its decision boundaries directly, rather than sampling a spatial grid [S2]. Unlike grid-based methods such as Marching Cubes or Dual Contouring, which introduce discretization errors and aliasing artifacts unless resolution is excessively high, Marching Neurons eliminates these artifacts by exploiting the piecewise-linear structure of the network to extract a high-fidelity mesh that faithfully captures the geometry encoded by the weights [S2].

## How it works
Marching Neurons incrementally constructs the 1-skeleton (vertices and edges) of the ReLU-induced polyhedral complex in the input space by sequentially processing neuron boundaries [S1]. The algorithm maintains a set of vertices $V$ and edges $E$ representing the current 1-skeleton [S1]. It iterates through neurons $(m, k)$ in a specific order, subdividing existing edges that cross the neuron's decision boundary $H^{(m)}_k$ and connecting the resulting intersection points [S1].

**Inputs:**
*   Current vertex set $V$ and edge set $E$ [S1].
*   Neuron $(m, k)$ defined by its pre-activation function $\nu^{(m)}_k(\cdot)$ [S1].
*   Numerical stability threshold $\epsilon_s$ [S1].

**Outputs:**
*   Updated vertex set $V$ and edge set $E$ [S1].
*   Intersection set $S$ containing points on the boundary with their sign vectors [S1].

**Data Structures:**
*   **Sign:** $\gamma^{(m)}_k(x) \in \{-1, 0, +1\}$, representing the sign of the pre-activation $\nu^{(m)}_k(x)$, computed with threshold $\epsilon_s$ for stability [S1].
*   **Sign Vector:** $s^{(m)}_k(x) = [\gamma^{(1)}_1(x), \dots, \gamma^{(m)}_k(x)]$, tracking the linear region state of point $x$ up to neuron $(m, k)$ [S1].
*   **Intersection Set:** $S$, a set of tuples $(\hat{x}, s^{(m)}_k(\hat{x}))$ where $\hat{x}$ is an intersection point on $H^{(m)}_k$ [S1].

**Algorithm Steps:**
1.  Initialize $S \leftarrow \emptyset$ and snapshot current edges $E_{old} \leftarrow E$ [S1].
2.  Iterate over all edges $(x_0, x_1) \in E_{old}$:
    a. Compute signs $\gamma_0 = \gamma^{(m)}_k(x_0)$ and $\gamma_1 = \gamma^{(m)}_k(x_1)$ [S1].
    b. If $\gamma_0 \gamma_1 < 0$ (the edge crosses the boundary):
        i. Compute pre-activations $d_0 = \nu^{(m)}_k(x_0)$ and $d_1 = \nu^{(m)}_k(x_1)$ [S1].
        ii. Calculate interpolation weight $w = |d_0| / |d_0 - d_1|$ [S1].
        iii. Compute intersection point $\hat{x} = (1 - w)x_0 + w x_1$ [S1].
        iv. Update vertices: $V \leftarrow V \cup \{\hat{x}\}$ [S1].
        v. Update edges: $E \leftarrow (E \setminus \{(x_0, x_1)\}) \cup \{(x_0, \hat{x}), (\hat{x}, x_1)\}$ [S1].
        vi. Add to intersection set: $S \leftarrow S \cup \{(\hat{x}, s^{(m)}_k(\hat{x}))\}$ [S1].
3.  Connect intersections: $E \leftarrow E \cup \text{ConnectIntersections}(S)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sign(val, eps):
    if val > eps:
        return 1
    elif val < -eps:
        return -1
    else:
        return 0

def compute_sign_vector(x, neuron_m_k, epsilon_s):
    # s^(m)_k(x) requires evaluating all previous neurons up to (m,k) at point x [S1]
    # This is computationally expensive and typically optimized in practice.
    return [sign(neuron_m_k.pre_activation(x), epsilon_s)]

def connect_intersections(E, S):
    # The sources do not give the full implementation of ConnectIntersections.
    # It connects intersection points on H^(m)_k into edges by tracking adjacent 
    # regions via sign-vector perturbation (Secs. 3.2 and 3.3 of [S1]).
    # A standard implementation would iterate over S, compare sign vectors 
    # to identify shared linear region boundaries, and append connecting edges to E.
    return E

def marching_neurons_step(V, E, neuron_m_k, epsilon_s):
    """
    Performs one step of edge subdivision for neuron (m, k).
    
    Args:
        V: List of vertices (3D coordinates).
        E: List of edges (pairs of vertex coordinates).
        neuron_m_k: Object with method pre_activation(x) returning nu^(m)_k(x).
        epsilon_s: Threshold for sign stability.
        
    Returns:
        V: Updated vertex list.
        E: Updated edge list.
        S: List of tuples (intersection_point, sign_vector).
    """
    S = []
    E_old = list(E)
    new_edges = []
    
    for (x0, x1) in E_old:
        gamma0 = sign(neuron_m_k.pre_activation(x0), epsilon_s)
        gamma1 = sign(neuron_m_k.pre_activation(x1), epsilon_s)
        
        if gamma0 * gamma1 < 0:
            d0 = neuron_m_k.pre_activation(x0)
            d1 = neuron_m_k.pre_activation(x1)
            
            # Avoid division by zero; gamma check implies d0 != d1
            if abs(d0 - d1) < 1e-9:
                continue
                
            w = abs(d0) / abs(d0 - d1)
            x_hat = (1 - w) * x0 + w * x1
            
            V.append(x_hat)
            new_edges.append((x0, x_hat))
            new_edges.append((x_hat, x1))
            
            s_hat = compute_sign_vector(x_hat, neuron_m_k, epsilon_s)
            S.append((x_hat, s_hat))
            
    # Remove crossed edges and add subdivided edges
    E = [e for e in E if e not in E_old] + new_edges
    
    # Connect intersections on the same face/region
    E = connect_intersections(E, S)
    
    return V, E, S
```

**Parameters:**
*   $\epsilon_s$: Numerical stability threshold for sign determination. Value not given in the sources.
*   Neuron Order: The sequence in which neurons $(m, k)$ are processed. Value not given in the sources.

**Edge Cases and Pitfalls:**
*   **TetraSDF Integration:** In TetraSDF, region tracking must account for both ReLU MLP linear regions and encoder-induced polyhedral cells [S1]. The pure sign-vector test is replaced with an augmented indicator $r(x) \oplus s^{(m)}_k(x)$ to specify the region, and barycentric masks $m(x)$ are used to check if a point lies on a cell boundary [S1].
*   **Sign Vector Computation:** Computing the full sign vector $s^{(m)}_k(x)$ for every intersection point requires evaluating all previous neurons, which can be costly [S1]. The sources note that `ConnectIntersections` relies on this vector for adjacency determination [S1].

## Operating characteristics
*   **Accuracy:** Eliminates grid aliasing and discretization artifacts, preserving sharp features without requiring infinite resolution sampling [S2].
*   **Complexity:** Not given in the sources.
*   **Memory:** Memory usage is constant with respect to spatial resolution, as it does not rely on a dense voxel grid [S1].
*   **Strengths:** Extracts exact zero-level sets; differentiable integration into learning pipelines is possible (though not inherent to the extraction itself) [S2].
*   **Limitations:** Requires the network to be ReLU-based to exploit piecewise-linearity [S2].

## Background
Marching Neurons was introduced by Stippel et al. in 2025 [S2]. It addresses the limitations of post-processing mesh extraction methods like Marching Cubes [LC87], Dual Contouring [JLSW02], and Marching Tetrahedra [DK91], which introduce discretization errors and cannot preserve sharp features without excessive grid resolution [S2]. While some works integrated differentiable contouring layers into the learning pipeline [LDG18, GCX*20, CZ21, GSF22], they often incurred high memory footprints due to underlying grid resolutions [S2]. Marching Neurons overcomes these precision limits by traversing decision boundaries directly [S2]. The method is closely related to TetraSDF, which adapts the edge subdivision scheme for multi-resolution tetrahedral grids [S1].

## Sources
[S1] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
