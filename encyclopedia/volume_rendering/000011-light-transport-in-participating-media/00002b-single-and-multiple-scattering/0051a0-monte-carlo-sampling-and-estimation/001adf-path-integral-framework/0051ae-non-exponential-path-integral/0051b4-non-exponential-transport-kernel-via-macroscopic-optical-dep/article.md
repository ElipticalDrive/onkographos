# Non-exponential transport kernel via macroscopic optical depth

Also called: Base transport kernel decomposition, Heterogeneity model

## Overview
This method models light transport in non-exponential, heterogeneous participating media by decomposing the medium description into an explicit 3D macroscopic density field $\sigma(\mathbf{x})$ and a 1D base transport kernel $T_{\text{base}}(t)$ that captures statistical correlations [S1]. It solves the difficulty of numerically evaluating the ensemble-averaged path integral for heterogeneous media by reducing the transport kernel to a function of macroscopic optical depth, allowing practical rendering while preserving non-exponential behavior [S1]. It differs from fully homogenized models (like [Larsen and Vasques](link)) by allowing arbitrary spatial distribution of correlations, and from general heterogeneous models by assuming statistical homogeneity of the underlying correlations [S1].

## How it works
The method computes the ensemble-averaged transport kernel $\tilde{T}(\mathbf{x}_i, \mathbf{x}_{i+1})$ for a segment between positions $\mathbf{x}_i$ and $\mathbf{x}_{i+1}$ using the macroscopic optical depth and the relative location of the endpoints with respect to the volume boundary $\partial\mathcal{V}$ and interior $\mathcal{V}$ [S1].

**Inputs and Outputs**
*   **Inputs**:
    *   Positions $\mathbf{x}_i, \mathbf{x}_{i+1} \in \mathbb{R}^3$.
    *   Macroscopic extinction coefficient field $\sigma(\mathbf{x})$.
    *   Base kernel functions $\text{ff}(t), \text{f}(t), \text{pf}(t), \text{pp}(t)$ (pre-computed or tabulated).
*   **Outputs**:
    *   Scalar transport kernel value $\tilde{T}(\mathbf{x}_i, \mathbf{x}_{i+1})$.

**Data Structures**
The base kernel functions are typically stored as 1D lookup tables or parametric functions mapping optical depth $t \in [0, \infty)$ to kernel values. The macroscopic density $\sigma(\mathbf{x})$ is a 3D scalar field.

**Steps**
1.  **Calculate Macroscopic Optical Depth**: Compute the macroscopic optical depth $\tau_i$ between $\mathbf{x}_i$ and $\mathbf{x}_{i+1}$ along the straight line segment connecting them:
    $$ \tau_i = \tau(\mathbf{x}_i, \mathbf{x}_{i+1}) = \int_0^{|\mathbf{x}_{i+1} - \mathbf{x}_i|} \sigma(\mathbf{x}_i + s \cdot \frac{\mathbf{x}_{i+1} - \mathbf{x}_i}{|\mathbf{x}_{i+1} - \mathbf{x}_i|}) \, ds $$
    [S1]

2.  **Determine Endpoint Locations**: Check the location of $\mathbf{x}_i$ and $\mathbf{x}_{i+1}$ relative to the volume boundary $\partial\mathcal{V}$ and interior $\mathcal{V}$:
    *   If a point is on the boundary, it is in $\partial\mathcal{V}$.
    *   If a point is inside the medium, it is in $\mathcal{V}$.
    [S1]

3.  **Select Base Function**: Choose the appropriate base function based on the location logic:
    *   $\text{ff}(\tau_i)$ if $\mathbf{x}_i \in \partial\mathcal{V}$ AND $\mathbf{x}_{i+1} \in \partial\mathcal{V}$.
    *   $\text{pp}(\tau_i)$ if $\mathbf{x}_i \in \mathcal{V}$ AND $\mathbf{x}_{i+1} \in \mathcal{V}$.
    *   $\text{pf}(\tau_i)$ otherwise (mixed boundary/interior).
    [S1]

4.  **Evaluate Kernel**: Return the value of the selected base function at $\tau_i$.
    $$ \tilde{T}(\mathbf{x}_i, \mathbf{x}_{i+1}) = \tilde{T}_{\text{base}}(\tau_i) $$
    [S1]

**Formulas**
The base kernel functions are defined as ensemble averages over realizations $\mu$ [S1]:
$$ \text{ff}(t) = \langle \text{Tr}(t) \rangle $$
$$ \text{f}(t) = \langle \text{Tr}(\sigma_\mu(\mathbf{x}_t)) \rangle $$
$$ \text{pf}(t) = \langle \text{Tr}(t)\rho_\mu(\mathbf{x}_t) \rangle $$
$$ \text{pp}(t) = \langle \text{Tr}(t) \sigma_{\mu}(\mathbf{x}_t) \rho_{\mu}(\mathbf{x}_t) \rangle $$
where $\text{Tr}(t)$ is the transmittance for a realization, and $\rho_\mu$ is related to the free-flight PDF [S1].

The final transport kernel is:
$$ \tilde{T}(\mathbf{x}_i, \mathbf{x}_{i+1}) = \begin{cases} \text{ff}(\tau_i) & \text{if } \mathbf{x}_i \in \partial\mathcal{V} \wedge \mathbf{x}_{i+1} \in \partial\mathcal{V} \\ \text{pp}(\tau_i) & \text{if } \mathbf{x}_i \in \mathcal{V} \wedge \mathbf{x}_{i+1} \in \mathcal{V} \\ \text{pf}(\tau_i) & \text{else} \end{cases} $$
[S1]

**Parameters**
*   $\sigma(\mathbf{x})$: Macroscopic extinction coefficient. Represents large-scale heterogeneous variation. Determines local stretching of the base medium [S1].

**Pitfalls**
*   The function $\text{pp}$ is a PDF, so it incurs a Jacobian factor of $\sigma(\mathbf{x}_{i+1})$ from the transformation. This factor cancels the division by $\sigma$ from the equivalent of Eq 22, resulting in a simple transport kernel [S1]. Implementers must ensure this cancellation is handled correctly if deriving $\text{pp}$ from first principles rather than using the pre-defined base kernel [S1].
*   The method assumes statistical homogeneity of correlations throughout the medium. If correlations vary spatially, the 1D base kernel assumption may not hold [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_transport_kernel(x_i, x_next, sigma_field, boundary_test, base_kernel_ff, base_kernel_pf, base_kernel_pp):
    """
    Computes the non-exponential transport kernel for a segment.
    
    Args:
        x_i: Position vector (3D)
        x_next: Next position vector (3D)
        sigma_field: Function sigma(x) returning macroscopic extinction
        boundary_test: Function is_boundary(x) returning True if x is on boundary, False if in interior
        base_kernel_ff: Function ff(tau) for boundary-to-boundary
        base_kernel_pf: Function pf(tau) for mixed
        base_kernel_pp: Function pp(tau) for interior-to-interior
        
    Returns:
        Float: Transport kernel value
    """
    # Step 1: Calculate macroscopic optical depth
    # Approximate integral using numerical quadrature or analytical solution if sigma is simple
    # For implementation, assume a helper function exists or use simple midpoint rule
    direction = x_next - x_i
    length = norm(direction)
    if length == 0:
        return 1.0 # Zero distance, full transmission
    
    # Simple numerical integration for optical depth
    # In practice, this might be more sophisticated
    num_samples = 10
    tau = 0.0
    for k in range(num_samples):
        t = (k + 0.5) / num_samples
        point = x_i + t * direction
        sigma_val = sigma_field(point)
        tau += sigma_val * (length / num_samples)
        
    # Step 2: Determine endpoint locations
    is_xi_boundary = boundary_test(x_i)
    is_xnext_boundary = boundary_test(x_next)
    
    # Step 3 & 4: Select base function and evaluate
    if is_xi_boundary and is_xnext_boundary:
        return base_kernel_ff(tau)
    elif not is_xi_boundary and not is_xnext_boundary:
        return base_kernel_pp(tau)
    else:
        return base_kernel_pf(tau)
```

## Subtopics

### Heterogeneous Decomposition
A specific application of the base transport kernel decomposition where the medium is described by a heterogeneous macroscopic density $\sigma(\mathbf{x})$ and a 1D transport kernel $T_{\text{base}}(t)$ that captures only correlations, assuming statistical homogeneity of correlations throughout the medium [S1]. This model lies between the fully homogenized model of Larsen and Vasques (identical statistics everywhere) and the general heterogeneous model (no assumptions about correlation distribution) [S1]. It is chosen when the medium has large-scale density variations but consistent small-scale correlation statistics [S1].

## Operating characteristics
*   **Time Complexity**: The evaluation of the transport kernel is $O(1)$ if the base kernel functions are pre-computed lookup tables or simple analytical functions, plus the cost of computing the macroscopic optical depth $\tau_i$. Computing $\tau_i$ requires integrating $\sigma(\mathbf{x})$ along the segment, which is $O(N)$ where $N$ is the number of samples in the integration [S1].
*   **Memory Complexity**: $O(V)$ for storing the 3D macroscopic density field $\sigma(\mathbf{x})$, plus $O(K)$ for storing the 1D base kernel tables, where $K$ is the number of entries in the tables [S1].
*   **Strengths**: Reduces the complex 3D heterogeneous non-exponential problem to a 1D kernel lookup, making it practical to estimate numerically or derive analytically [S1]. Allows easy superposition of density variations and correlation models [S1].
*   **Limitations**: Relies on the assumption of statistical homogeneity of correlations. If correlations are not homogeneous, the 1D base kernel may not accurately represent the medium [S1].
*   **When to use**: When rendering heterogeneous media with non-exponential attenuation where the small-scale correlation statistics are uniform, but the large-scale density varies [S1].
*   **When not to use**: When the medium has spatially varying correlation statistics that cannot be captured by a single 1D base kernel [S1].

## Background
The method was introduced in "A radiative transfer framework for non-exponential media" [S1]. It addresses the difficulty of numerically evaluating the path integral for non-exponential media by decomposing the medium into a 1D transport kernel and a 3D density field [S1]. The approach is motivated by the desire to combine the benefits of fully homogenized models (like Larsen and Vasques) and general heterogeneous models, providing a middle ground that supports heterogeneous media while maintaining practical rendering methods [S1].

## Sources
[S1] A radiative transfer framework for non-exponential media, https://doi.org/10.1145/3272127.3275103, fetched 2026-11-08
