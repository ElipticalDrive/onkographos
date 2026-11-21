# Curvature-Based Anisotropy Estimation
Also called: Kindlmann et al

## Overview
This method estimates anisotropy for surface shading by computing the principal curvatures of an isosurface [S1]. It serves as a baseline comparison against covariance-based methods, relying on differential geometry properties where principal curvature directions form an orthonormal basis in the tangent plane [S1]. Unlike covariance-based approaches that evaluate feature extent in both spatial and data domains, this method focuses strictly on the local geometry of the isosurface [S1].

## How it works
**Inputs:** Scalar field volume, isovalue for surface extraction, precomputed gradient and Hessian fields.
**Outputs:** Anisotropy parameters (principal curvature magnitudes and directions) for use in anisotropic BRDFs (e.g., Ward’s BRDF).

**Data Structures:**
*   **Gradient Field:** 3-component vector per voxel ($\nabla f$).
*   **Hessian Field:** Symmetric $3 \times 3$ matrix per voxel ($H$). Stored as 6 values due to symmetry ($H_{xx}, H_{xy}, H_{xz}, H_{yy}, H_{yz}, H_{zz}$).

**Steps:**
1.  **Precomputation:** Compute the Hessian matrix of the scalar field on a 3D grid. Filter the Hessian matrix to reduce noise [S1].
2.  **Gradient Computation:** Compute the gradient vector $\nabla f$ at each voxel [S1].
3.  **Geometry Tensor Construction:** At rendering time, for each shading point on the isosurface, construct the geometry tensor using the filtered gradient and Hessian [S1]. The geometry tensor relates to the shape operator or second fundamental form in differential geometry.
4.  **Eigen Decomposition:** Compute the eigenvalues and eigenvectors of the geometry tensor on-the-fly [S1].
    *   The eigenvalues correspond to the principal curvature magnitudes ($\kappa_1, \kappa_2$).
    *   The eigenvectors correspond to the principal curvature directions [S1].
5.  **Shading:** Use the principal curvatures and their directions as input for an anisotropic BRDF, such as Ward’s BRDF [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_curvature_anisotropy(shading_point, gradient_field, hessian_field):
    """
    Computes anisotropy parameters from principal curvatures.
    
    Args:
        shading_point: 3D coordinate on the isosurface.
        gradient_field: Precomputed and filtered gradient field.
        hessian_field: Precomputed and filtered Hessian field.
        
    Returns:
        kappa1, kappa2: Principal curvature magnitudes.
        v1, v2: Principal curvature directions (eigenvectors).
    """
    # 1. Retrieve precomputed data at the shading point
    grad = gradient_field.sample(shading_point)
    hess = hessian_field.sample(shading_point)
    
    # 2. Construct the geometry tensor (Shape Operator)
    # The shape operator S is typically defined as - (Jacobian of normal) projected to tangent plane.
    # In discrete grid terms, this involves the Hessian and Gradient.
    # Normal n = normalize(grad)
    # The geometry tensor G is derived from H and grad.
    # Note: Exact formulation of "geometry tensor" from H and grad is implied by differential geometry context.
    # Commonly, the Weingarten map or shape operator is computed.
    # Let's assume a standard projection-based construction for the shape operator S:
    # S = - (I - n*n^T) * H * (I - n*n^T) / |grad|
    # Or similar variant depending on specific implementation details not fully explicit in source.
    # However, source states: "geometry tensor is computed from the filtered gradient vector and Hessian matrix."
    
    # For implementation completeness, we define the standard differential geometry approach:
    n = normalize(grad)
    # Project Hessian onto tangent plane to get shape operator components
    # This step requires specific linear algebra operations not fully detailed in source text,
    # but is standard for extracting curvatures from Hessian.
    # We assume the source implies the standard extraction:
    # Eigenvalues of the shape operator are the principal curvatures.
    
    # Simplified representation of the tensor construction based on source description:
    # The source does not give the explicit formula for the "geometry tensor" from H and grad,
    # only that it is computed from them. We mark this as a gap.
    geometry_tensor = construct_geometry_tensor(grad, hess) # [G] Placeholder for specific linear algebra construction
    
    # 3. Compute eigenvalues and eigenvectors
    eigenvalues, eigenvectors = compute_eigen_decomposition(geometry_tensor)
    
    # 4. Extract principal curvatures and directions
    # Eigenvalues correspond to principal curvature magnitudes
    kappa1 = eigenvalues[0]
    kappa2 = eigenvalues[1]
    
    # Eigenvectors correspond to principal curvature directions
    v1 = eigenvectors[0]
    v2 = eigenvectors[1]
    
    return kappa1, kappa2, v1, v2
```

**Parameters:**
*   **Filtering:** The Hessian matrix is filtered during precomputation [S1]. Specific filter type and kernel size are not given in the sources.
*   **BRDF:** Ward’s BRDF is used as the shading model [S1].

**Edge Cases and Pitfalls:**
*   **Artifacts at Transitions:** The method is susceptible to visual artifacts close to transitions of structures (e.g., boundaries of ellipsoids) [S1].
*   **Aliasing:** Local estimation can lead to disturbing visual patterns (aliasing) when the dataset is noisy or features are small compared to image resolution [S1].
*   **Precision:** Eigen decomposition is performed on-the-fly, which may introduce numerical instability if the geometry tensor is ill-conditioned [G].

## Subtopics

## Operating characteristics
*   **Precomputation Time:** Faster than covariance-based methods. Measured at 465 ms for the tested dataset [S1].
*   **Memory Footprint:** Higher than covariance-based methods. Requires storing 3 values for the gradient and 6 values for the symmetric $3 \times 3$ Hessian per voxel. Total memory consumption was 67.4 MB for the tested dataset [S1].
*   **Rendering Performance:** Slower than covariance-based methods. Measured at 69 fps compared to 81 fps for the covariance method [S1]. This is attributed to the need to store and access a symmetric $3 \times 3$ matrix and perform on-the-fly eigen decomposition [S1].
*   **Hardware:** Results obtained on Intel Core i7 3.4 GHz CPU, 32 GB RAM, and NVIDIA Titan GPU [S1].
*   **Strengths:** Faster precomputation [S1].
*   **Limitations:** Higher memory usage, slower rendering, and susceptibility to artifacts at structural transitions [S1].

## Background
This method is based on the work of Kindlmann et al. [24], as cited in the source [S1]. It utilizes principles from differential geometry, specifically the property that principal curvature directions form an orthonormal basis in the tangent plane [S1]. It is used as a comparative baseline for evaluating new anisotropy estimation techniques in volume rendering, particularly in the context of anisotropic ambient volume shading [S1].

## Sources
[S1] Anisotropic Ambient Volume Shading, https://doi.org/10.1109/tvcg.2015.2467963, fetched 2026-10-28
