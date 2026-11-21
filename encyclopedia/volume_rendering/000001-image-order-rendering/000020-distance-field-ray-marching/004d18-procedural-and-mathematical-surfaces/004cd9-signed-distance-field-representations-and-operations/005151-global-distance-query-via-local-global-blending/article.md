# Global Distance Query via Local-Global Blending

Also called: Local-Global Blending SDF

## Overview
This method constructs a global signed distance field (SDF) $F_G$ that retains the sharp geometric features of a local patch-based representation $F_L$ while enabling valid distance queries across the entire bounding domain. It solves the problem where local representations are accurate near the surface but fail or become unstable far away, and standard learned global fields smooth out sharp corners. It differs from pure local methods by providing a global query capability and from pure global methods by preserving high-frequency local details through a blending region.

## How it works
**Inputs and Outputs**
*   **Input**: A local SDF $F_L(\mathbf{x})$ defined near the surface $S$, and a global SDF $F_O(\mathbf{x})$ defined over the entire domain.
*   **Output**: A blended global SDF $F_G(\mathbf{x})$.

**Data Structures**
The method operates on scalar fields. The global field $F_O$ is implemented as a vanilla 8-layered Multi-Layer Perceptron (MLP) with 512 neurons per layer, equipped with positional encoding [S1]. The local field $F_L$ is typically a patch-based representation [S1].

**Parameters**
*   $\delta_1 = 0.001$: Inner boundary of the blending region.
*   $\delta_2 = 0.03$: Outer boundary of the blending region.
*   Domain $D = [-1, 1]^3$: The bounding box of the shape [S1].

**Steps**
1.  Define three regions based on the absolute value of the local distance $|F_L(\mathbf{x})|$:
    *   Region $R_1 = \{ \mathbf{x} \mid |F_L(\mathbf{x})| < \delta_1 \}$
    *   Region $R_2 = \{ \mathbf{x} \mid \delta_1 \le |F_L(\mathbf{x})| \le \delta_2 \}$
    *   Region $R_3 = D \setminus (R_1 \cup R_2)$
2.  For a query point $\mathbf{x}$, determine which region it falls into.
3.  If $\mathbf{x} \in R_1$, return $F_L(\mathbf{x})$.
4.  If $\mathbf{x} \in R_3$, return $F_O(\mathbf{x})$.
5.  If $\mathbf{x} \in R_2$, compute the blended value $F_G(\mathbf{x})$ using weights derived from the normalized distance parameter $s(\mathbf{x})$.

**Formulas**
The normalized distance parameter $s(\mathbf{x})$ maps $|F_L(\mathbf{x})| \in [\delta_1, \delta_2]$ to $[0, 1]$:
$$ s(\mathbf{x}) = \frac{|F_L(\mathbf{x})| - \delta_1}{\delta_2 - \delta_1} $$

The weight functions use the smoothstep function $h(s) = 3s^2 - 2s^3$:
$$ w_0(s) = 1 - h(s) $$
$$ w_1(s) = h(s) $$

The blended field in $R_2$ is:
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
*   **Precision**: The transition in $R_2$ is $C^1$ continuous due to smoothstep, but the derivative of the absolute value function at $F_L(\mathbf{x})=0$ is discontinuous. However, since $R_1$ uses $F_L$ directly and $R_2$ starts at $\delta_1 > 0$, the discontinuity at zero is handled by the exclusive boundary of $R_1$.
*   **Boundary Definition**: The regions are defined by the magnitude of $F_L$. If $F_L$ is not a true distance field (violating Lipschitz condition), the blending may not preserve the exact geometry as intended [G].
*   **Domain**: The method assumes the domain $D$ is bounded, typically $[-1, 1]^3$ for CAD models [S1].

## Subtopics

### SDF Primitive Definitions
Defines the basic analytical shapes (spheres, boxes) used to construct $F_L$ or $F_O$. [SDF Primitive Definitions](1578-sdf-primitive-definitions.md)

### Distance Field Representations and Sampling
Discusses how $F_L$ and $F_O$ are stored and sampled (e.g., neural networks, grids). [Distance Field Representations and Sampling](8725-distance-field-representations-and-sampling.md)

### SDF Transformations
Describes how to apply transformations to the primitives before blending. [SDF Transformations](11027-sdf-transformations.md)

## Operating characteristics
*   **Time Complexity**: $O(1)$ per query, dominated by the evaluation of $F_L$ and $F_O$. If $F_O$ is an 8-layer MLP, this involves several matrix multiplications and activations.
*   **Memory Complexity**: Depends on the storage of $F_L$ (patch-based) and $F_O$ (MLP weights). The MLP has $8 \times 512 \times 512$ weights (plus biases), which is significant but manageable.
*   **Strengths**: Retains sharp features (corners, edges) that are lost in standard global neural SDFs, while providing a valid distance field for the entire domain, enabling operations like offsetting and boolean operations globally.
*   **Limitations**: Requires both a local and a global representation. The blending region $R_2$ introduces a transition zone where the geometry is an approximation.
*   **When to use**: When high-fidelity local details are required alongside global scene queries (e.g., collision detection, ray marching across the whole object).
*   **When not to use**: If only local rendering is needed, or if memory is extremely constrained and storing both fields is prohibitive.

## Background
This method was introduced in the context of CAD model representation, specifically to address the need for global signed distance functions in CAD applications [S1]. Standard patch-based representations are accurate near the surface but do not support global queries. Learned global fields (like those using MLPs) smooth out sharp features. The blending strategy allows the retention of sharp features from the local representation while leveraging the global coverage of the learned field. The global field $F_O$ is obtained following the strategy by Lin et al. [2024], using positional encoding as introduced by Mildenhall et al. [2020] and Tancik et al. [2020b].

## Sources
[S1] Patch-Grid: An Efficient and Feature-Preserving Neural Implicit Surface Representation, https://doi.org/10.1145/3727142, fetched 2026-10-28
