# 1D Slice Reconstruction with Piecewise Constant Model

Also called: 1D Piecewise Constant Slice Reconstruction, Constrained Minimization for 1D Slice Reconstruction

## Overview
This method reconstructs a 2D slice from parallel projections by modeling the slice as a 1D piecewise constant function defined by $n$ parameters [S1, unofficial]. It solves for these parameters by minimizing a constrained quadratic criterion that compares model projections against measured data [S1, unofficial]. Unlike voxel-based methods that solve for individual grid cells, this approach models the object structure directly via zones, reducing the parameter space but introducing non-differentiability in the optimization criterion [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Measured parallel projection data $p$ for a specific 2D slice.
*   **Output:** A vector of $n$ parameters defining the 1D piecewise constant model of the slice.

**Data Structures:**
The model represents the 2D slice as a function constant along one axis (e.g., $y$) and piecewise constant along the other (e.g., $x$).
*   **Model Parameters:** A vector $\theta \in \mathbb{R}^n$. Each parameter corresponds to the value (e.g., density or attenuation coefficient) of a specific zone in the 1D profile.
*   **Zones:** The domain is partitioned into $n$ contiguous zones. The boundaries between zones are determined by the parameterization.

**Steps:**
1.  **Model Definition:** Define the 1D piecewise constant model $f(x; \theta)$ for the slice, where $\theta$ contains $n$ parameters representing the values of $n$ zones [S1, unofficial].
2.  **Forward Projection:** Compute the theoretical projection $\hat{p}(\theta)$ of the model $f(x; \theta)$ using parallel ray geometry. This involves integrating the model values along the projection rays [S1, unofficial].
3.  **Criterion Formulation:** Construct a quadratic criterion $J(\theta)$ measuring the difference between the measured data $p$ and the model projection $\hat{p}(\theta)$ [S1, unofficial]:
    $$
    J(\theta) = \| p - \hat{p}(\theta) \|^2
    $$
4.  **Regularization:** Since the quadratic criterion $J(\theta)$ is non-differentiable due to the piecewise constant nature of the model (discontinuities at zone boundaries), apply one of three proposed regularization techniques to handle this non-differentiability [S1, unofficial].
5.  **Constrained Minimization:** Solve for the optimal parameters $\theta^*$ by minimizing the regularized criterion subject to constraints:
    $$
    \min_\theta J_{reg}(\theta) \quad \text{subject to constraints}
    $$
    [S1, unofficial].
6.  **Noise Analysis (Optional):** Compute the covariance matrix of the estimated parameter vector $\theta$ to analyze sensitivity to noise in the input data [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstruct_1d_slice(projections_p, n_zones, regularization_method):
    """
    Reconstruct a 2D slice modeled as a 1D piecewise constant function.
    
    Args:
        projections_p: Array of measured parallel projection data.
        n_zones: Integer, number of zones (parameters) in the model.
        regularization_method: String, one of the three proposed techniques 
                               to handle non-differentiability.
    
    Returns:
        theta: Array of n_zones parameters defining the slice.
    """
    # Initialize parameters theta with n_zones values
    theta = initialize_parameters(n_zones)
    
    # Define the forward projection operator for the 1D piecewise constant model
    def forward_project(theta):
        # Construct the 1D profile from theta
        profile = construct_piecewise_constant_profile(theta, n_zones)
        # Compute parallel projections of the profile
        # This involves line integrals along the projection angles
        return compute_parallel_projections(profile)
    
    # Define the non-differentiable quadratic criterion
    def criterion(theta):
        p_hat = forward_project(theta)
        return sum((p - p_hat)**2 for p, p_hat in zip(projections_p, p_hat))
    
    # Apply regularization to handle non-differentiability
    # The specific implementation depends on the chosen regularization_method
    if regularization_method == "technique_1":
        regularized_criterion = apply_regularization_tech1(criterion, theta)
    elif regularization_method == "technique_2":
        regularized_criterion = apply_regularization_tech2(criterion, theta)
    elif regularization_method == "technique_3":
        regularized_criterion = apply_regularization_tech3(criterion, theta)
    else:
        raise ValueError("Unknown regularization method")
    
    # Perform constrained minimization
    # Note: Standard gradient-based optimizers may fail due to non-differentiability.
    # Specialized algorithms or the specific regularization techniques must be used.
    theta_optimal = minimize_constrained(regularized_criterion, theta)
    
    return theta_optimal

def compute_covariance_matrix(theta_optimal, projections_p, n_zones):
    """
    Analyze sensitivity of the reconstructed parameter vector to noise.
    
    Args:
        theta_optimal: Optimal parameters from reconstruction.
        projections_p: Measured projection data.
        n_zones: Number of zones.
        
    Returns:
        cov_matrix: Covariance matrix of the parameter vector.
    """
    # Determine the covariance matrix of the parameter vector after reconstruction
    # This typically involves linearizing the forward model around the solution
    # and propagating the noise covariance from the projections.
    # Implementation details for covariance calculation are not specified in the source.
    cov_matrix = calculate_covariance(theta_optimal, projections_p, n_zones)
    return cov_matrix
```

**Parameters:**
*   $n$: Number of parameters in the 1D piecewise constant model, corresponding to the number of zones [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-differentiability:** The quadratic criterion is non-differentiable due to the piecewise constant model structure [S1, unofficial]. Specialized regularization techniques are required [S1, unofficial].
*   **Noise Sensitivity:** The reconstruction is sensitive to noise in the input data, which can be analyzed via the covariance matrix of the parameter vector [S1, unofficial].

## Subtopics

### Noise Sensitivity Analysis via Covariance Matrix
This variant focuses on analyzing the sensitivity of the reconstructed parameter vector to noise present in the input data [S1, unofficial]. It involves determining the covariance matrix of the parameter vector after reconstruction [S1, unofficial]. This analysis helps quantify the uncertainty and stability of the reconstruction results.

### 3D Reconstruction via Iterative 1D Slices with Smoothing
This variant extends the 1D slice reconstruction to 3D objects by iterating the 1D reconstruction approach for all slices of the 3D object [S1, unofficial]. It adds a smoothing term between adjacent slices during the iteration to enforce consistency across the volume [S1, unofficial]. This approach leverages the 1D method as a building block for volumetric reconstruction.

### Regularization of Non-Differentiable Quadratic Criterion
This variant addresses the core challenge of the method: the non-differentiability of the quadratic criterion [S1, unofficial]. It proposes three specific regularization techniques to handle this issue [S1, unofficial]. These techniques are applied during the minimization step to enable optimization despite the discontinuities in the model.

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Models the object structure directly with fewer parameters ($n$ zones) compared to voxel-based methods, potentially leading to more efficient reconstruction for objects with simple 1D profiles.
*   **Limitations:** The non-differentiability of the criterion requires specialized regularization and optimization techniques [S1, unofficial]. It is limited to 2D slice reconstruction in its basic form, requiring iteration for 3D [S1, unofficial].
*   **When to use:** Suitable for reconstructing 2D slices from parallel projections when the object can be well-approximated by a 1D piecewise constant model.
*   **When not to use:** Not suitable for complex 3D objects without iteration, or when differentiability is a strict requirement for the optimization algorithm.

## Background
This method originates from research on tomographic reconstruction from a small number of views [S1, unofficial]. The work was conducted by an author who first studied the reconstruction of a planar slice using a 1D model described by zones with $n$ parameters [S1, unofficial]. The approach was part of a broader investigation into modeling objects for reconstruction, which later extended to 3D models characterized by separating surfaces and density fields [S1, unofficial]. The specific techniques for regularization and noise analysis were proposed to address the challenges of non-differentiability and noise sensitivity in this constrained minimization framework [S1, unofficial].

## Sources
[S1] Reconstruction tomographique à partir d'un petit nombre de vues, https://openalex.org/W650467298, fetched 2026-10-29 (unofficial)
