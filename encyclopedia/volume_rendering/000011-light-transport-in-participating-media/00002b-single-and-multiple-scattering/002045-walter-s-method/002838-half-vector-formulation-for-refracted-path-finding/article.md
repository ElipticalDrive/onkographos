# Half-vector formulation for refracted path finding

Also called: Half-vector based formulation

## Overview
This method finds points $P$ on a boundary surface that refract light from a source $\mathbf{L}$ to a scatter point $\mathbf{V}$, specifically handling cases where the shading normal $\hat{\mathbf{N}}_s$ differs from the geometric normal $\hat{\mathbf{N}}_g$ [S1]. It sets itself apart from standard Fermat’s principle approaches by using a refractive half-vector $\hat{\mathbf{H}}$ to enforce Snell's law relative to the shading normal, allowing for accurate path finding on surfaces with interpolated normals or normal maps [S1].

## How it works
**Inputs:**
*   Source position $\mathbf{L}$ (3D vector) [S1].
*   Scatter/view position $\mathbf{V}$ (3D vector) [S1].
*   Candidate point $P$ on the boundary (3D vector) [S1].
*   Shading normal at $P$, $\hat{\mathbf{N}}_s$ (unit 3D vector) [S1].
*   Relative index of refraction $\eta = \eta_{in}/\eta_{out}$ (scalar) [S1].

**Outputs:**
*   Boolean validity of $P$ as a refraction point [S1].
*   The half-vector $\hat{\mathbf{H}}$ used for validation [S1].

**Data Structures:**
*   Vectors are stored as 3-component floating-point arrays or standard vector types [S1].
*   Normals are unit vectors [S1].

**Steps:**
1.  Compute the direction vector from $P$ to $V$:
    $$ \hat{\omega}_V = \frac{\mathbf{V} - \mathbf{P}}{\|\mathbf{V} - \mathbf{P}\|} $$
    The source text contains a typo in Equation (1) writing $\mathbf{V} \cdot \mathbf{P}$, but context and Equation (2) confirm subtraction is required for direction vectors [S1].

2.  Compute the direction vector from $P$ to $L$:
    $$ \hat{\omega}_L = \frac{\mathbf{L} - \mathbf{P}}{\|\mathbf{L} - \mathbf{P}\|} $$

3.  Compute the unnormalized half-vector sum: $\mathbf{H}_{\text{raw}} = \eta \hat{\omega}_V + \hat{\omega}_L$ [S1]. The source text uses $\hat{\omega}_{\text{A}}$ in Equation (3), which contextually refers to $\hat{\omega}_{\text{V}}$ (the internal direction) [S1].

4.  Normalize to get the half-vector:
    $$ \hat{\mathbf{H}} = \frac{\mathbf{H}_{\text{raw}}}{\|\mathbf{H}_{\text{raw}}\|} $$

5.  Validate the point $P$. $P$ is a valid solution if and only if both conditions are met [S1]:
    *   **Normal Alignment:** $\hat{\mathbf{H}} = -\hat{\mathbf{N}}_s$ [S1].
    *   **Side Constraints:** $(\mathbf{V} \cdot \hat{\mathbf{N}}_s) \leq (\mathbf{P} \cdot \hat{\mathbf{N}}_s) \leq (\mathbf{L} \cdot \hat{\mathbf{N}}_s)$ [S1].
    The normal alignment ensures Snell's law is obeyed relative to the shading normal [S1]. The side constraints ensure that $V$ and $L$ lie on opposite sides of the local tangent plane defined by $\hat{\mathbf{N}}_s$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def half_vector_formulation(L, V, P, Ns, eta):
    """
    L: Source position (vec3)
    V: Scatter/View position (vec3)
    P: Candidate point on boundary (vec3)
    Ns: Shading normal at P (vec3, unit length)
    eta: Relative index of refraction (float)
    Returns: (is_valid: bool, H: vec3)
    """
    # Step 1: Direction from P to V
    dir_V = V - P
    norm_V = length(dir_V)
    if norm_V == 0:
        return False, vec3(0, 0, 0)
    omega_V = dir_V / norm_V

    # Step 2: Direction from P to L
    dir_L = L - P
    norm_L = length(dir_L)
    if norm_L == 0:
        return False, vec3(0, 0, 0)
    omega_L = dir_L / norm_L

    # Step 3 & 4: Compute normalized half-vector H
    H_raw = omega_V * eta + omega_L
    norm_H = length(H_raw)
    if norm_H == 0:
        return False, vec3(0, 0, 0)
    H = H_raw / norm_H

    # Step 5: Validation
    # Condition 1: H must be anti-parallel to shading normal
    is_aligned = abs(dot(H, -Ns) - 1.0) < 1e-6

    # Condition 2: V and L must satisfy the side constraint inequality
    proj_V = dot(V, Ns)
    proj_P = dot(P, Ns)
    proj_L = dot(L, Ns)
    is_between = (proj_V <= proj_P) and (proj_P <= proj_L)
    
    is_valid = is_aligned and is_between
    return is_valid, H
```

**Parameters:**
*   $\eta$: Relative index of refraction. Not given in the sources. [S1]
*   Epsilon for floating point comparison: $10^{-6}$ is typical for normal alignment checks [G].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $P = V$ or $P = L$, the direction vectors are undefined and the method fails [G].
*   **Typo Correction:** The source text contains typos in Equations (1) and (3) ($\mathbf{V} \cdot \mathbf{P}$ and $\hat{\omega}_{\text{A}}$). Implementers must use subtraction for direction vectors and substitute $\hat{\omega}_{\text{V}}$ for $\hat{\omega}_{\text{A}}$ [S1].
*   **Normal Orientation:** The side constraint $(\mathbf{V} \cdot \hat{\mathbf{N}}_s) \leq (\mathbf{P} \cdot \hat{\mathbf{N}}_s) \leq (\mathbf{L} \cdot \hat{\mathbf{N}}_s)$ assumes a specific orientation of $\hat{\mathbf{N}}_s$ [S1].

## Subtopics

### [Sidedness Test](4237-sidedness-test.md)
A geometric pruning test to determine if a position bounding box and normal bounding cone can satisfy refraction conditions. It uses 3D boxes for positions instead of 2D triangles.

### [Newton-Raphson for Refraction Point Search](4659-newton-raphson-for-refraction-point-search.md)
An iterative method to find the point $P$ on a triangle boundary using the half-vector formulation. It uses a specific heuristic for the initial guess and clamping, applicable when shading normals are constant (Case 1).

### [Accurate single scattering under point light (Walter et al.)](7000-accurate-single-scattering-under-point-light-walter-et-al.md)
Computes single scattering by treating it as a deterministic search problem using interval Newton optimization to find all valid refraction points.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per candidate point $P$ [G].
*   **Memory Complexity:** $O(1)$ [G].
*   **Strengths:** Handles varying shading normals ($\hat{\mathbf{N}}_g \neq \hat{\mathbf{N}}_s$) which Fermat's principle does not support directly [S1].
*   **Limitations:** Requires a candidate point $P$ to validate and functions as a validation step rather than a search algorithm [S1].
*   **When to use:** When rendering refractive caustics in participating media with complex surface normals (normal maps, interpolated normals) [S1].

## Background
The half-vector formulation was introduced by Walter et al. in 2007 [S1]. It was developed to address the limitations of Fermat's principle in handling shading normals that differ from geometric normals, such as those found in triangles with interpolated normals or normal maps [S1]. This method is a core component of Walter's Method for accurate single scattering in refractive media [S1].

## Sources
[S1] Single Scattering in Refractive Media with Triangle Mesh Boundaries, https://doi.org/10.1145/1531326.1531398, fetched 2026-10-26
