# Global Distance Query via Local-Global Blending

Also called: Local-Global Blending SDF

## Overview
This method constructs a feature-preserving global signed distance field $F_G$ by blending a local patch-based representation $F_L$ with an ordinary global signed distance field $F_O$ [S1]. It solves the problem where local representations are accurate near the surface but fail to provide valid distance queries far away, while global neural fields often smooth out sharp geometric features [S1]. It is distinct from pure local methods (which lack global validity) and pure global neural fields (which lose sharp corners) by retaining local precision in a narrow band while falling back to the global field elsewhere [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Local signed distance field $F_L(\mathbf{x})$, Global signed distance field $F_O(\mathbf{x})$, bounding domain $D = [-1, 1]^3$, blending thresholds $\delta_1, \delta_2$.
*   **Output:** Blended global signed distance field $F_G(\mathbf{x})$.

**Data Structures:**
The method relies on two scalar fields. $F_L$ is typically a patch-based representation (e.g., grid-learned or mesh-based) valid near the surface. $F_O$ is a global field, implemented in the source as a vanilla 8-layered Multi-Layer Perceptron (MLP) with 512 neurons per layer, equipped with positional encoding [S1].

**Parameters:**
*   $\delta_1 = 0.001$: Inner boundary of the blending region [S1].
*   $\delta_2 = 0.03$: Outer boundary of the blending region [S1].
*   $D = [-1, 1]^3$: The bounding domain of the shape [S1].

**Steps:**
1.  Evaluate the local distance value $d_L = |F_L(\mathbf{x})|$ at the query point $\mathbf{x}$.
2.  Determine the region $R$ containing $\mathbf{x}$ based on $d_L$:
    *   Region $R_1$: $d_L < \delta_1$.
    *   Region $R_2$: $\delta_1 \le d_L \le \delta_2$.
    *   Region $R_3$: $d_L > \delta_2$ (i.e., $\mathbf{x} \in D \setminus (R_1 \cup R_2)$).
3.  Compute the blended field $F_G(\mathbf{x})$:
    *   If $\mathbf{x} \in R_1$, $F_G(\mathbf{x}) = F_L(\mathbf{x})$.
    *   If $\mathbf{x} \in R_3$, $F_G(\mathbf{x}) = F_O(\mathbf{x})$.
    *   If $\mathbf{x} \in R_2$, compute the normalized distance parameter $s(\mathbf{x})$ and weights $w_0, w_1$, then interpolate:
        $$ s(\mathbf{x}) = \frac{|F_L(\mathbf{x})| - \delta_1}{\delta_2 - \delta_1} $$
        $$ w_0(\mathbf{x}) = 1 - h(s(\mathbf{x})) $$
        $$ w_1(\mathbf{x}) = h(s(\mathbf{x})) $$
        $$ F_G(\mathbf{x}) = w_0(\mathbf{x}) F_L(\mathbf{x}) + w_1(\mathbf{x}) F_O(\mathbf{x}) $$
        where $h(s) = 3s^2 - 2s^3$ is the smoothstep function.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smoothstep(s):
    """Standard smoothstep function: 3s^2 - 2s^3."""
    return 3 * s * s - 2 * s * s * s

def blend_global_local_sdf(x, F_L, F_O, delta1=0.001, delta2=0.03):
    """
    Computes the blended global signed distance field F_G.
    
    Args:
        x: Query point (vector).
        F_L: Function evaluating the local signed distance field.
        F_O: Function evaluating the global signed distance field.
        delta1: Inner blending threshold (default 0.001).
        delta2: Outer blending threshold (default 0.03).
        
    Returns:
        F_G: The blended signed distance value.
    """
    # Evaluate local field
    f_l = F_L(x)
    abs_f_l = abs(f_l)
    
    # Region 1: Very close to surface, use local field directly
    if abs_f_l < delta1:
        return f_l
    
    # Region 3: Far from surface, use global field directly
    if abs_f_l > delta2:
        return F_O(x)
    
    # Region 2: Blending region
    # Normalize distance to [0, 1]
    s = (abs_f_l - delta1) / (delta2 - delta1)
    
    # Compute weights using smoothstep
    h_s = smoothstep(s)
    w_0 = 1.0 - h_s
    w_1 = h_s
    
    # Evaluate global field
    f_o = F_O(x)
    
    # Linear blend
    f_g = w_0 * f_l + w_1 * f_o
    
    return f_g
```

**Edge Cases and Pitfalls:**
*   **Precision:** The transition is smooth ($C^1$ continuous) due to smoothstep, but the derivative discontinuity at $\delta_1$ and $\delta_2$ boundaries is not explicitly addressed in the source. Implementers should ensure $F_L$ and $F_O$ are evaluated consistently.
*   **Domain:** The method assumes the bounding domain $D$ is known and that $F_O$ is valid throughout $D$. If $F_O$ is unstable outside a certain radius, $R_3$ logic may fail.
*   **Sharp Features:** The method preserves sharp features in $R_1$ because $F_L$ is used directly. $F_O$ alone would smooth these features [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query, assuming $F_L$ and $F_O$ are $O(1)$. The blending logic adds constant overhead.
*   **Memory Complexity:** Depends on the storage of $F_L$ and $F_O$. For the neural $F_O$ (8-layer MLP, 512 neurons), memory is proportional to the number of weights ($8 \times 512 \times 512$ floats).
*   **Strengths:** Retains sharp geometric features (corners, edges) that global neural fields typically smooth out, while providing a valid distance field for global queries (inside/outside tests) [S1].
*   **Limitations:** Requires both a local representation and a global representation. The local representation $F_L$ must be accurate within $\delta_2$. The choice of $\delta_1$ and $\delta_2$ affects the trade-off between feature preservation and global validity.
*   **When to use:** When CAD-like models with sharp features need to be queried globally (e.g., for collision detection or marching cubes) but local neural fields alone are insufficient for global queries.
*   **When not to use:** If only local surface rendering is needed, or if the global field $F_O$ is not available or accurate.

## Background
The method was introduced in the context of neural implicit surface representations for CAD models, specifically to address the limitation of local patch-based representations that cannot support global distance queries [S1]. It builds on the strategy by Lin et al. [2024] for generating the global field $F_O$ using an MLP with positional encoding [S1]. The approach is designed to combine the feature-preserving nature of local representations with the global validity of learned fields.

## Sources
[S1] Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation, https://doi.org/10.1145/3727142, fetched 2026-10-28
