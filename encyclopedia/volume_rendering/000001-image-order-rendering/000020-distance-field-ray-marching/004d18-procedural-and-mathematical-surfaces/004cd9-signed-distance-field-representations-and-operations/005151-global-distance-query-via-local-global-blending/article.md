# Global Distance Query via Local-Global Blending

Also called: Local-Global Blending SDF

## Overview
This method constructs a global signed distance field (SDF) $F_G$ that retains the sharp geometric features of a local patch-based representation $F_L$ while enabling valid distance queries across the entire bounding domain [S1]. It solves the problem where local representations are accurate near the surface but become unstable far away, and standard learned global fields smooth out sharp corners [S1]. It differs from pure local methods by providing a global query capability and from pure global methods by preserving high-frequency local details through a blending region [S1].

## How it works
**Inputs and Outputs**
*   **Input**: A local SDF $F_L(\mathbf{x})$ defined near the surface $S$, and a global SDF $F_O(\mathbf{x})$ defined over the entire domain [S1].
*   **Output**: A blended global SDF $F_G(\mathbf{x})$ [S1].

**Data Structures**
The method operates on scalar fields. The global field $F_O$ is implemented as a vanilla 8-layered Multi-Layer Perceptron (MLP) with 512 neurons per layer, equipped with positional encoding [S1]. The local field $F_L$ is a patch-based representation [S1].

**Parameters**
*   $\delta_1 = 0.001$: Inner boundary of the blending region $R_2$ [S1].
*   $\delta_2 = 0.03$: Outer boundary of the blending region $R_2$ [S1].
*   Domain $D = [-1, 1]^3$: The bounding box of the shape [S1].

**Steps**
1.  Define three regions based on the absolute value of the local distance $|F_L(\mathbf{x})|$: $R_1 = \{ \mathbf{x} \mid |F_L(\mathbf{x})| < \delta_1 \}$, $R_2 = \{ \mathbf{x} \mid \delta_1 \le |F_L(\mathbf{x})| \le \delta_2 \}$, and $R_3 = D \setminus (R_1 \cup R_2)$ [S1].
2.  For a query point $\mathbf{x}$, determine which region it falls into [S1].
3.  If $\mathbf{x} \in R_1$, return $F_L(\mathbf{x})$ [S1].
4.  If $\mathbf{x} \in R_3$, return $F_O(\mathbf{x})$ [S1].
5.  If $\mathbf{x} \in R_2$, compute the blended value $F_G(\mathbf{x})$ using weights derived from the normalized distance parameter $s(\mathbf{x})$ [S1].

**Formulas**
The normalized distance parameter $s(\mathbf{x})$ maps $|F_L(\mathbf{x})| \in [\delta_1, \delta_2]$ to $[0, 1]$ [S1]:
$$ s(\mathbf{x}) = \frac{|F_L(\mathbf{x})| - \delta_1}{\delta_2 - \delta_1} $$

The weight functions use the smoothstep function $h(s) = 3s^2 - 2s^3$ [S1]:
$$ w_0(s) = 1 - h(s) $$
$$ w_1(s) = h(s) $$

The blended field in $R_2$ is [S1]:
$$ F_G(\mathbf{x}) = w_0(s(\mathbf{x})) F_L(\mathbf{x}) + w_1(s(\mathbf{x})) F_O(\mathbf{x}) $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def smoothstep(s):
    """Standard smoothstep function."""
    return 3 * s * s - 2 * s * s * s

def global_distance_query(x, F_L, F_O, delta1=0.001, delta2=0.03):
    """
    Compute the blended global SDF value at point x.
    
    Args:
        x: Query point (vector).
        F_L: Function returning local SDF value.
        F_O: Function returning global SDF value.
        delta1: Inner blending boundary.
        delta2: Outer blending boundary.
        
    Returns:
        Blended SDF value.
    """
    # Evaluate local distance
    val_L = F_L(x)
    abs_val_L = abs(val_L)
    
    # Region R1: Use local field directly
    if abs_val_L < delta1:
        return val_L
    
    # Region R3: Use global field directly
    if abs_val_L > delta2:
        return F_O(x)
    
    # Region R2: Blend
    # Normalize distance to [0, 1]
    s = (abs_val_L - delta1) / (delta2 - delta1)
    
    # Compute weights using smoothstep
    h = smoothstep(s)
    w0 = 1.0 - h
    w1 = h
    
    # Evaluate global field
    val_O = F_O(x)
    
    # Linear blend
    return w0 * val_L + w1 * val_O
```

**Edge Cases and Pitfalls**
*   **Precision**: The transition in $R_2$ is $C^1$ continuous due to smoothstep, but the derivative of the absolute value function at $F_L(\mathbf{x})=0$ is discontinuous [G]. Since $R_1$ uses $F_L$ directly and $R_2$ starts at $\delta_1 > 0$, the discontinuity at zero is handled by the exclusive boundary of $R_1$ [G].
*   **Boundary Definition**: The regions are defined by the magnitude of $F_L$ [S1].
*   **Domain**: The method assumes the domain $D$ is bounded, typically $[-1, 1]^3$ for CAD models [S1].

## Subtopics

### [Local Gaussian Process Signed Distance Field (L-GPDF)](19271-local-gaussian-process-signed-distance-field-l-gpdf.md)
The Local Gaussian Process Signed Distance Field (L-GPDF) is a temporary, patch-based representation used within the VDB-GPDF framework to enable fast, probabilistic queries of Euclidean distance and surface properties for arbitrary points in large-scale environments. It sets itself apart from its siblings by providing the specific local inference engine that feeds into the global blending process, focusing on incremental, frame-by-frame updates of local surface geometry without the post-processing required by TSDF or occupancy fusion frameworks. It should be chosen when high-frequency local geometry updates are needed alongside global distance queries. [Local Gaussian Process Signed Distance Field (L-GPDF)](19271-local-gaussian-process-signed-distance-field-l-gpdf.md)

## Operating characteristics
*   **Time Complexity**: $O(1)$ per query, dominated by the evaluation of $F_L$ and $F_O$ [G]. If $F_O$ is an 8-layer MLP, this involves several matrix multiplications and activations [G].
*   **Memory Complexity**: Depends on the storage of $F_L$ (patch-based) and $F_O$ (MLP weights) [G]. The MLP has $8 \times 512 \times 512$ weights (plus biases), which is significant but manageable [S1].
*   **Strengths**: Retains sharp features (corners, edges) that are lost in standard global neural SDFs, while providing a valid distance field for the entire domain, enabling operations like offsetting and boolean operations globally [S1].
*   **Limitations**: Requires both a local and a global representation [S1]. The blending region $R_2$ introduces a transition zone where the geometry is an approximation [S1].
*   **When to use**: When high-fidelity local details are required alongside global scene queries (e.g., collision detection, ray marching across the whole object) [G].
*   **When not to use**: If only local rendering is needed, or if memory is extremely constrained and storing both fields is prohibitive [G].

## Background
This method was introduced in the context of CAD model representation, specifically to address the need for global signed distance functions in CAD applications [S1]. Standard patch-based representations are accurate near the surface but do not support global queries, while learned global fields smooth out sharp features [S1]. The blending strategy allows the retention of sharp features from the local representation while leveraging the global coverage of the learned field [S1]. The global field $F_O$ is obtained following the strategy by Lin et al. [2024], using positional encoding as introduced by Mildenhall et al. [2020] and Tancik et al. [2020b] [S1].

## Sources
[S1] Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation, https://doi.org/10.1145/3727142, fetched 2026-10-28
