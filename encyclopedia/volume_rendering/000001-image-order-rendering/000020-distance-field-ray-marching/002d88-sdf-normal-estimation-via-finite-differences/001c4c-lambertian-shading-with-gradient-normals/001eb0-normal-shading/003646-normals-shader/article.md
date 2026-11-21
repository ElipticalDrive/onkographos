# Normals Shader

Also called: NormalsShader, Object Normals Shader

## Overview
The Normals Shader visualizes surface normals by converting vector components from the range $[-1, 1]$ to the displayable range $[0, 1]$. It serves as a debugging and visualization tool within the Normal Shading family, distinct from the Spin Shader's rotational encoding and the Normal Integrator's gradient accumulation. It outputs the normal direction as an emission color, allowing developers to verify normal orientation and smoothness.

## How it works
The method takes computed surface normals and applies a linear transformation to map them to the $[0, 1]$ range, optionally scaled and offset by user parameters.

**Inputs and Outputs:**
- Input: Surface normal vector $\mathbf{n} = (n_x, n_y, n_z)$, where each component is in $[-1, 1]$.
- Output: Emission color $\mathbf{c} = (c_x, c_y, c_z)$ in $[0, 1]$.

**Data Structures:**
- $\mathbf{n}$: `Vector3` or equivalent 3-component float array.
- Parameters: `_NormalScale` (float), `_NormalOffset` (float).

**Parameters:**
- `_NormalScale`: Scaling factor for the normal output. Default: $0$ [S1, unofficial].
- `_NormalOffset`: Offset added to the normal output. Default: $0$ [S1, unofficial].

**Formula:**
The transformation for each component $n_i$ is:
$$ c_i = \left( \frac{n_i - (-1)}{1 - (-1)} \right) \times \text{_NormalScale} + \text{_NormalOffset} $$
Simplifying the fraction:
$$ c_i = (n_i + 1) \times 0.5 \times \text{_NormalScale} + \text{_NormalOffset} $$

**Steps:**
1. Retrieve the surface normal vector $\mathbf{n}$.
2. For each component $i \in \{x, y, z\}$:
   a. Compute the normalized value: $v_i = (n_i + 1) \times 0.5$.
   b. Apply scale and offset: $c_i = v_i \times \text{_NormalScale} + \text{_NormalOffset}$.
3. Clamp the resulting color components to $[0, 1]$ if necessary, though the linear mapping from $[-1, 1]$ with default scale/offset naturally produces $[0, 1]$.
4. Output $\mathbf{c}$ as the emission color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normals_shader(vertex_normal: Vector3, normal_scale: float = 0.0, normal_offset: float = 0.0) -> Vector3:
    """
    Converts surface normals from [-1, 1] to [0, 1] range for visualization.
    
    Args:
        vertex_normal: The surface normal vector with components in [-1, 1].
        normal_scale: Scaling factor for the output. Default 0.
        normal_offset: Offset added to the output. Default 0.
        
    Returns:
        Vector3: The normalized normal vector as an emission color in [0, 1].
    """
    # Map [-1, 1] to [0, 1]
    # Formula: (n + 1) * 0.5
    normalized_x = (vertex_normal.x + 1.0) * 0.5
    normalized_y = (vertex_normal.y + 1.0) * 0.5
    normalized_z = (vertex_normal.z + 1.0) * 0.5
    
    # Apply scale and offset
    out_x = normalized_x * normal_scale + normal_offset
    out_y = normalized_y * normal_scale + normal_offset
    out_z = normalized_z * normal_scale + normal_offset
    
    return Vector3(out_x, out_y, out_z)
```

**Edge Cases and Pitfalls:**
- If `_NormalScale` is $0$, the output is constant `_NormalOffset` for all components, resulting in a uniform color (e.g., gray if offset is $0.5$) [S1, unofficial].
- If normals are not normalized, the visualization still works but may misrepresent the true direction if interpreted as a unit vector field.
- Components outside $[-1, 1]$ will result in values outside $[0, 1]$ after the initial mapping, potentially causing clipping or incorrect visualization if not clamped.

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel/vertex. The operation involves a constant number of arithmetic operations.
- **Memory Complexity:** $O(1)$. No additional data structures are required beyond the input normal and output color.
- **Strengths:** Simple, fast, and effective for debugging normal orientation.
- **Limitations:** Only visualizes direction, not curvature or other geometric properties. The default scale of $0$ [S1, unofficial] renders it useless for visualization unless explicitly set to $1.0$.

## Background
The Normals Shader is implemented in Unity-based projects, specifically noted in the context of dual contouring and surface reconstruction [S1, unofficial]. It is used to visualize vertex normals as emission colors, aiding in the verification of mesh quality and normal continuity.

## Sources
[S1] DMeville/Unity-Fast-Dual-Contouring, https://github.com/DMeville/Unity-Fast-Dual-Contouring, fetched 2026-10-26 (unofficial)
[S2] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
