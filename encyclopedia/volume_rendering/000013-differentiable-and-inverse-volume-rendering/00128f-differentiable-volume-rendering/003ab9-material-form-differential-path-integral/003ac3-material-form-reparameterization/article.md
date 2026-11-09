# Material-form reparameterization

Also called: Material-form formulation, Material-form path integral

## Overview
Material-form reparameterization is a technique for differentiating the path integral with respect to scene parameters by mapping evolving geometry to a fixed reference domain, thereby avoiding the direct differentiation of path space boundaries [S1]. It sets itself apart from standard path integral differentiation by converting light paths into material paths via a differentiable mapping, allowing the derivative to be computed as an integral over a static material path space plus a boundary term [S1]. This method builds on [Material-Form Differential Path Integral](8831-material-form-differential-path-integral.md).

## How it works
**Inputs:**
*   Scene parameter $\pi \in \mathbb{R}$.
*   Evolving surface $M(\pi)$.
*   Fixed reference surface $B$.
*   Differentiable one-to-one mapping $\mathbf{X}(\cdot, \pi): B \to M(\pi)$ [S1].

**Outputs:**
*   Material-form path integral representation of the radiometric measurement $I$ [S1].

**Data Structures:**
*   **Material Path Space $\hat{\Omega}$**: Defined as $\hat{\Omega} := \cup_{N \ge 1} \mathbf{B}^{N+1}$, where $\mathbf{B}$ is the reference surface [S1].
*   **Material Path $\bar{\mathbf{p}}$**: A sequence $(\mathbf{p}_0, \dots, \mathbf{p}_N)$ where $\mathbf{p}_n \in B$ [S1].
*   **Light Path $\bar{\mathbf{x}}$**: A sequence $(\mathbf{x}_0, \dots, \mathbf{x}_N)$ where $\mathbf{x}_n \in M(\pi)$ [S1].

**Steps:**
1.  Define the fixed reference surface $B$ and the differentiable mapping $\mathbf{X}(\mathbf{p}, \pi)$ such that $\mathbf{x} = \mathbf{X}(\mathbf{p}, \pi)$ maps any point on $B$ to the corresponding point on $M(\pi)$ [S1].
2.  Transform every light path $\bar{\mathbf{x}} = (\mathbf{x}_0, \dots, \mathbf{x}_N)$ into a material path $\bar{\mathbf{p}} = (\mathbf{p}_0, \dots, \mathbf{p}_N)$ using the relation $\mathbf{x}_n = \mathbf{X}(\mathbf{p}_n, \pi)$ for all $0 \le n \le N$ [S1].
3.  Apply the change of variable to the original path integral. The measurement $I$ is expressed as:
    $$ I = \int_{\hat{\Omega}} \hat{f}(\bar{\mathbf{p}}) d\mu(\bar{\mathbf{p}}) $$
    where $\hat{\Omega}$ is the material path space and $d\mu$ is the associated measure [S1].
4.  Compute the material measurement contribution $\hat{f}(\bar{\mathbf{p}})$ by scaling the original light path contribution $f(\bar{\mathbf{x}})$ with the Jacobian of the transformation:
    $$ \hat{f}(\bar{\mathbf{p}}) = f(\bar{\mathbf{x}}) \prod_{n=0}^{N} J(\mathbf{p}_n, \pi) $$
    where $J(\mathbf{p}, \pi)$ is the Jacobian determinant resulting from the reparameterization [S1].
5.  Calculate the Jacobian $J(\mathbf{p}, \pi)$ as the norm of the derivative of the surface-area measure:
    $$ J(\mathbf{p}, \pi) := \left\| \frac{dA(\mathbf{x})}{dA(\mathbf{p})} \right\| $$
    where $A$ indicates the surface-area measure [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def material_form_reparameterization(light_path, pi, reference_surface, mapping_X):
    """
    Computes the material-form path integral contribution for a given light path.
    
    Args:
        light_path: List of points [x_0, ..., x_N] on the evolving surface M(pi).
        pi: Scene parameter.
        reference_surface: The fixed reference surface B.
        mapping_X: Function X(p, pi) mapping points from B to M(pi).
        
    Returns:
        material_contribution: The value of f_hat(p_bar).
        material_path: List of points [p_0, ..., p_N] on B.
    """
    N = len(light_path) - 1
    material_path = []
    jacobian_product = 1.0
    
    # Step 2: Transform light path to material path
    # Inverse mapping is required: p = X^{-1}(x, pi)
    # Note: The source defines x = X(p, pi). Implementation requires inverting X.
    for x_n in light_path:
        # [G] Invert the mapping X to find p such that X(p, pi) = x_n.
        # This assumes X is bijective and differentiable.
        p_n = invert_mapping(mapping_X, x_n, pi)
        material_path.append(p_n)
        
    # Step 4 & 5: Compute Jacobian product and material contribution
    f_light = evaluate_light_path_contribution(light_path) # Defined in parent context
    
    for p_n in material_path:
        # Calculate Jacobian J(p, pi) = || dA(x) / dA(p) ||
        # [G] Compute the derivative of the mapping X with respect to p,
        # then compute the ratio of area measures.
        # For a surface, this involves the determinant of the Jacobian matrix
        # of the mapping restricted to the tangent space.
        dx_dp = compute_jacobian_matrix(mapping_X, p_n, pi)
        # [G] Compute the norm of the area measure derivative.
        # This is typically the square root of the determinant of (J^T J)
        # for the tangent space projection.
        jacobian_n = compute_area_jacobian_norm(dx_dp)
        jacobian_product *= jacobian_n
        
    material_contribution = f_light * jacobian_product
    
    return material_contribution, material_path
```

**Edge Cases and Pitfalls:**
*   **Mapping Differentiability**: The mapping $\mathbf{X}$ must be differentiable with respect to both $\mathbf{p}$ and $\pi$ [S1]. Non-differentiable mappings break the change of variable.
*   **Bijectivity**: The mapping must be one-to-one [S1]. Self-intersections or folding of the surface $M(\pi)$ relative to $B$ violate this condition.
*   **Jacobian Zero/Infinity**: If the mapping collapses area ($J=0$) or stretches it infinitely ($J \to \infty$), the integral becomes ill-defined [S1].

## Subtopics

### Material-form reparameterization for participating media
This variant extends the surface-only formulation to volumes. The reference surface union $\mathcal{B}$ becomes a union of reference surfaces $\mathcal{B}_M$ and interior volumes $\mathcal{B}_V$ [S2, unofficial]. Consequently, the definitions of the material path space $\hat{\Omega}$, the measure $d\mu$, and the Jacobian $J(\mathbf{p}_n)$ change to account for volume elements and interior boundaries [S2, unofficial]. The resulting differential path integral retains the same form (interior + boundary terms) but the boundary vertices $\mathbf{p}_K$ can reside on surfaces or in volumes, altering the definition of $\partial \hat{\Omega}$, the visibility term $V_K$, and the boundary measure $d\mu'$ [S2, unofficial].

## Operating characteristics
*   **Complexity**: Not given in the sources.
*   **Strengths**: Eliminates the need to differentiate the path space domain $\Omega$ directly, which evolves with $\pi$ [S1]. It converts the problem into an integral over a fixed domain $\hat{\Omega}$ [S1].
*   **Limitations**: Requires the existence of a differentiable, one-to-one mapping $\mathbf{X}$ between the reference and evolving geometry [S1]. This may be difficult to construct for complex, topologically changing scenes.
*   **When to use**: When differentiating radiometric measurements with respect to parameters that deform geometry, and a suitable reference mapping can be defined [S1].

## Background
The material-form reparameterization was proposed by Zhang et al. [2020] to address the complexity of differentiating the path integral when scene geometry evolves with the parameter $\pi$ [S1]. It is a core component of the Material-Form Differential Path Integral formulation [S1].

## Sources
[S1] Path-Space Differentiable Rendering of Implicit Surfaces, https://doi.org/10.1145/3641519.3657473, fetched 2026-10-30
[S2] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
