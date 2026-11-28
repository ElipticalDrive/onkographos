# SDF Deformations

Also called: Non-uniform spatial distortion, Space Deformation, Twist deformation, Warping Operator

## Overview
SDF Deformations apply a non-uniform spatial distortion to a Signed Distance Field by warping the coordinate space, allowing for the creation of twisted, bent, or otherwise distorted geometric shapes. It solves the problem of generating complex, organic, or mechanical deformations (such as twisted cables) from simple primitive SDFs without requiring explicit mesh topology. It is distinguished from [Warped-Space SDF Ray Marching](4726-warped-space-sdf-ray-marching.md) by its explicit handling of multiphase SDFs and the mathematical constraint of the Lipschitz constant to preserve distance accuracy.

## How it works
**Inputs and Outputs:**
*   **Input:** A warping function $\omega: \mathbb{R}^3 \rightarrow \mathbb{R}^3$, a base SDF $\Phi_{\mathcal{A}}$ (potentially multiphase), and a scaling factor $\lambda$.
*   **Output:** A deformed SDF $\Phi(\mathbf{p})$ that preserves the signed distance property.

**Data Structures:**
*   **Warping Function $\omega$:** A mapping from world coordinates to deformed coordinates. The implementation requires the inverse function $\omega^{-1}$ to evaluate the distance at the deformed point.
*   **Lipschitz Constant $\lambda$:** A scalar value $\lambda \geq \sup \|\mathbf{J}_{\omega^{-1}}\|$, where $\mathbf{J}_{\omega^{-1}}$ is the Jacobian matrix of the inverse warping function. This ensures the deformed field remains a valid distance field.

**Numbered Steps:**
1.  **Compute Inverse Warp:** For a query point $\mathbf{p}$, determine the pre-warp coordinate $\mathbf{p}' = \omega^{-1}(\mathbf{p})$.
2.  **Evaluate Base SDF:** Compute the base distance $\phi_{\mathcal{A}}(\mathbf{p}')$ at the pre-warp coordinate.
3.  **Scale Distance:** Divide the base distance by $\lambda$ to obtain the deformed distance: $\phi_{\text{def}} = \phi_{\mathcal{A}}(\mathbf{p}') / \lambda$.
4.  **Multiphase Handling (Optional):** If deforming only a specific phase $k$ within a multiphase structure $\mathcal{A}$:
    *   For the target phase $k$: $\phi_k = \phi_{\mathcal{A}_k}(\mathbf{p}') / \lambda$.
    *   For all other phases $i \neq k$: $\phi_i = \max(\phi_{\mathcal{A}_i}(\mathbf{p}), -\phi_{\mathcal{A}_i}(\mathbf{p}') / \lambda)$. This combines the undeformed other phases with the deformed phase $k$ using the "over" operator.

**Formulas:**
For global warping of all phases:
$$ \Phi(\mathbf{p}) = \frac{1}{\lambda} \Phi_{\mathcal{A}} \circ \omega^{-1}(\mathbf{p}) $$
where $\lambda \geq \sup \|\mathbf{J}_{\omega^{-1}}\|$ [S1].

For partial phase deformation $D(\mathcal{A}, k)$:
$$ D(\mathcal{A}, k) = D(\mathcal{A}_k) \cup_+ (\mathcal{A} - \mathcal{A}_k) $$
Function evaluation:
$$ \phi_i = \max(\phi_{\mathcal{A}_i}, -\phi_{\mathcal{A}_i} \circ \omega^{-1}/\lambda) \quad \forall i \neq k $$
$$ \phi_k = \phi_{\mathcal{A}_k} \circ \omega^{-1}/\lambda $$
[S1].

**Parameters:**
*   $\lambda$: Scaling factor. Must be greater than or equal to the supremum of the Jacobian norm of the inverse warping function. Not given in the sources for specific numerical values, but critical for correctness [S1].

**Edge Cases and Pitfalls:**
*   **Lipschitz Violation:** If $\lambda$ is too small, the resulting field is no longer a valid signed distance field, leading to incorrect ray marching steps and visual artifacts.
*   **Inverse Solving:** Computing $\omega^{-1}(\mathbf{p})$ may require iterative numerical methods if $\omega$ is complex, which can impact performance.
*   **Phase Boundaries:** When deforming only a subset of phases, the interaction between deformed and undeformed regions must be handled carefully to avoid gaps or overlaps, typically using the `max` operator as defined [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_deformation(p, base_sdf, omega_inv, lambda_val, phase_k=None, num_phases=None):
    """
    Computes the deformed SDF value at point p.
    
    Args:
        p: vec3, query point in world space.
        base_sdf: function, returns SDF value(s) at a point.
        omega_inv: function, returns omega^-1(p).
        lambda_val: float, Lipschitz constant >= sup||J_omega_inv||.
        phase_k: int or None, if set, deforms only this phase.
        num_phases: int or None, required if phase_k is set.
        
    Returns:
        float or vecN, deformed SDF value(s).
    """
    # Step 1: Compute inverse warp
    p_prime = omega_inv(p)
    
    # Step 2 & 3: Evaluate base SDF and scale
    if phase_k is None:
        # Global warping
        phi_base = base_sdf(p_prime)
        return phi_base / lambda_val
    else:
        # Partial phase warping
        # Evaluate base SDF at both original and deformed points
        phi_base_original = base_sdf(p)
        phi_base_deformed = base_sdf(p_prime)
        
        result_phases = []
        for i in range(num_phases):
            if i == phase_k:
                # Deform only phase k
                phi_k = phi_base_deformed[i] / lambda_val
                result_phases.append(phi_k)
            else:
                # Combine undeformed phase i with deformed phase k
                # phi_i = max(phi_A_i, -phi_A_i_composed / lambda)
                # Note: phi_base_deformed[i] corresponds to phi_A_i composed with omega^-1
                phi_i = max(phi_base_original[i], -phi_base_deformed[i] / lambda_val)
                result_phases.append(phi_i)
        
        return result_phases
```

## Subtopics

### Twist Deformation
A specific warping function where rotation angle varies linearly with position (e.g., along the Y-axis). It is chosen for creating helical or twisted structures like cables or springs. See [Twist deformation](4726-warped-space-sdf-ray-marching.md#twist-deformation) for implementation details.

### Multiphase Warping
Applying deformation to only a subset of phases in a multiphase SDF. It is chosen when only specific components of a complex assembly need to be distorted while others remain static. See [Multiphase SDFs](1557-lipschitz-transformation-bounds.md#multiphase-sdfs) for context.

## Operating characteristics
*   **Time Complexity:** Dominated by the evaluation of $\omega^{-1}(\mathbf{p})$ and the base SDF. If $\omega^{-1}$ requires iteration, complexity increases.
*   **Memory Complexity:** O(1) additional memory per query, assuming $\omega^{-1}$ is computed on the fly.
*   **Strengths:** Enables complex, organic shapes from simple primitives; supports multiphase structures for detailed internal geometry.
*   **Limitations:** Requires careful calculation of $\lambda$ to preserve distance accuracy; inverse warping can be computationally expensive or numerically unstable.
*   **When to Use:** When creating twisted, bent, or distorted geometries that are difficult to model with standard primitives.
*   **When Not to Use:** When high performance is critical and $\omega^{-1}$ is expensive to compute, or when the deformation is uniform (use simple transformation instead).

## Background
The concept of space deformation for SDFs was formalized in the context of multiphase SDFs by the PhaseTree authors. The warping operator allows for the deformation of space following classical formulations, enabling the creation of complex scenes like twisted electrical cables [S1]. The technique is widely used in ray marching for procedural geometry, as demonstrated by examples such as twisting deformations and displacement maps [S2, unofficial].

## Sources
[S1] The PhaseTree: Multiphase Signed Distance Fields, https://doi.org/10.1145/3811379, fetched 2026-10-26
[S2] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
