# Differentiable Transient Rendering

Also called: Transient Path Integral Differentiable Rendering

## Overview
Differentiable Transient Rendering computes the gradients of the transient path integral with respect to scene parameters, enabling inverse rendering tasks that require high physical accuracy in time-resolved light transport [S1]. Unlike steady-state differentiable rendering where scattering events are independent, this method accounts for the total time of flight $\text{tof}(\bar{\mathbf{x}})$, which couples scattering events and requires treating them as a multidimensional, evolving manifold [S1]. It approximates the material form of the differential transient path integral using two separate Monte Carlo estimators for interior and boundary integrals, allowing for efficient optimization of parameters such as geometry, light position, and material properties in transient scenarios [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   Scene description $\boldsymbol{\theta}$ (parameters to optimize, e.g., geometry, light position, index of refraction).
*   Light source distribution $L$.
*   Sensor response function $S_e$, which acts as a temporal smoothing kernel and defines the non-zero time window for detection [S1].
*   Number of interior samples $N_i$ and boundary samples $N_b$ [S1].
*   Temporal domain discretization (histogram bins) [S1].

**Outputs:**
*   Gradient of the transient image intensity $I$ with respect to parameters $\boldsymbol{\theta}$, denoted $\frac{dI}{d\theta}$ [S1].
*   Transient image sequence (binned contributions) [S1].

### Data Structures
*   **Path $\bar{\mathbf{p}}$**: A sequence of vertices $\mathbf{x}_0, \dots, \mathbf{x}_k$. Each vertex stores position, normal, and material properties.
*   **Time of Flight $\text{tof}(\bar{\mathbf{x}})$**: A scalar value associated with the path, representing the total propagation delay from emission to detection [S1].
*   **Throughput $\hat{\mathbf{x}}(\bar{\mathbf{p}})$**: The radiance contribution of the path, including the derivative with respect to parameters $(\hat{\mathbf{x}})^\top (\bar{\mathbf{p}})$ [S1].
*   **Temporal Histogram**: An array of bins representing the temporal domain. Contributions are binned based on $\text{tof}$ [S1].

### Algorithm Steps
1.  **Path Tracing**: Trace a path $\hat{\mathbf{x}}(\bar{\mathbf{p}}_j, \boldsymbol{\theta})$ using a bidirectional path tracer [S1].
2.  **Time of Flight Tracking**: Compute the total time of flight $\text{tof}(\hat{\mathbf{x}}(\bar{\mathbf{p}}_j, \boldsymbol{\theta}))$ for the path [S1].
3.  **Throughput and Derivative Computation**: Compute the path throughput $\hat{\mathbf{x}}(\bar{\mathbf{p}}_j)$ and its parameter derivative $(\hat{\mathbf{x}})^\top (\bar{\mathbf{p}}_j)$ during the tracing process [S1]. This is done using automatic differentiation on top of next-event estimation and radiance-based multiple importance sampling for the interior integral [S1].
4.  **Temporal Footprint Selection**: Determine the time window for the path based on the non-zero regions of the sensor response function $S_e(\hat{\mathbf{x}}(\bar{\mathbf{p}}_j, \boldsymbol{\theta}))$ [S1].
5.  **Interior Integral Estimation**:
    *   Compute the interior contribution $(\hat{f}_{\mathcal{T}})^\top (\bar{\mathbf{p}}_j)$ [S1].
    *   Convolve the throughput and its derivative by $(\hat{S}_e)^\top (\bar{\mathbf{p}}_j)$ and $\hat{S}_e(\bar{\mathbf{p}}_j)$ respectively [S1].
    *   Bin the convolved contribution into the temporal histogram using $\text{tof}$ [S1].
6.  **Boundary Integral Estimation**:
    *   Sample boundary vertices $\bar{\mathbf{p}}_l$ using multidirectional sampling, next-event estimation, and grid-based importance sampling [S1].
    *   Compute the boundary contribution $\Delta \hat{f}_{\mathcal{T}}(\bar{\mathbf{p}}_l)$ and the normal velocity $\mathcal{V}_{\partial\bar{\mathcal{B}}}(\bar{\mathbf{p}}_l)$ [S1].
    *   Assume $S_e$ is continuous, so the discontinuity term $\Delta\Omega_k[S_e]$ vanishes [S1].
    *   Bin the contribution into the temporal histogram using $\text{tof}(\hat{\mathbf{x}}(\bar{\mathbf{p}}_l, \boldsymbol{\theta}))$ [S1].
7.  **Gradient Aggregation**: Combine the interior and boundary estimators to compute the final gradient [S1].

### Formulas
The gradient is approximated as the sum of two Monte Carlo estimators [S1]:

$$
\frac{dI}{d\theta} \approx \frac{1}{N_i} \sum_{j=1}^{N_i} \frac{(\hat{f}_{\mathcal{T}})^{\cdot}(\bar{\mathbf{p}}_j)}{p_i(\bar{\mathbf{p}}_j)} + \frac{1}{N_b} \sum_{l=1}^{N_b} \frac{\Delta \hat{f}_{\mathcal{T}}(\bar{\mathbf{p}}_l) \mathcal{V}_{\partial\bar{\mathcal{B}}}(\bar{\mathbf{p}}_l)}{p_b(\bar{\mathbf{p}}_l)}
$$

Where:
*   $N_i, N_b$: Number of interior and boundary samples [S1].
*   $p_i(\bar{\mathbf{p}}), p_b(\bar{\mathbf{p}})$: Sampling probabilities for the material path $\bar{\mathbf{p}}$ for each estimator [S1].
*   $(\hat{f}_{\mathcal{T}})^{\cdot}(\bar{\mathbf{p}}_j)$: The derivative of the transient path integral function for interior samples [S1].
*   $\Delta \hat{f}_{\mathcal{T}}(\bar{\mathbf{p}}_l)$: The boundary contribution function for vertex $\mathbf{x}_i \in \partial\mathcal{M}$ [S1].
*   $\mathcal{V}_{\partial\bar{\mathcal{B}}}(\bar{\mathbf{p}}_l)$: The scalar normal velocity at the boundary, defined as $\mathcal{V}_{\partial\mathcal{M}}(\mathbf{x}_i) = \mathbf{n}(\mathbf{x}_i) \cdot v(\mathbf{x}_i)$, where $\mathbf{n}$ is the unit normal and $v$ is the velocity of the manifold evolution [S1].

### Parameters
*   **Learning Rate**: Scene-dependent value used with the ADAM optimizer. Typical values: 0.07 for Coffee, Stars, and Wavy scenes; 40.0 for Corners scene [S1].
*   **Samples per Pixel (spp)**: Varies by scene. For example, Egg scene uses 410+919 spp (interior + boundary) [S1].

### Edge Cases and Pitfalls
*   **Reciprocity**: The current implementation lacks reciprocity, sharing this limitation with Zhang et al. [2020] [S1].
*   **Memory Bounds**: The implementation is memory-bounded, aggravated by the need to compute a full transient sequence, limiting scalability to thousands of parameters [S1].
*   **Discontinuities**: The method handles discontinuous importance functions for light and sensor using a correlated importance function [S1]. If $S_e$ is not continuous, the term $\Delta\Omega_k[S_e]$ does not vanish and must be accounted for [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differentiable_transient_rendering(scene_params, N_i, N_b, S_e, temporal_bins):
    """
    Computes the gradient of the transient image with respect to scene_params.
    
    Args:
        scene_params: Parameters theta to optimize.
        N_i: Number of interior samples.
        N_b: Number of boundary samples.
        S_e: Sensor response function (temporal smoothing kernel).
        temporal_bins: Array of bins for temporal reconstruction.
        
    Returns:
        gradient: Gradient of the transient image intensity.
        transient_image: Binned transient image sequence.
    """
    gradient_interior = 0.0
    gradient_boundary = 0.0
    transient_image = zeros_like(temporal_bins)
    
    # Interior Integral Estimator
    for j in range(N_i):
        # Trace path and compute throughput and derivative using AD
        path, throughput, throughput_deriv = trace_path(scene_params)
        
        # Compute time of flight
        tof = compute_time_of_flight(path)
        
        # Determine temporal footprint based on S_e
        time_window = get_non_zero_regions(S_e, tof)
        
        # Compute interior contribution derivative
        f_T_dot = compute_interior_contribution_derivative(path, throughput_deriv)
        
        # Convolve with sensor response
        # Note: S_e and its derivative are evaluated at the path's tof
        weighted_f_T_dot = f_T_dot * derivative_of_S_e(S_e, tof)
        weighted_throughput = throughput * S_e(S_e, tof)
        
        # Bin contributions
        bin_index = map_to_bin(tof, temporal_bins)
        transient_image[bin_index] += weighted_throughput
        gradient_interior += weighted_f_T_dot / sampling_probability_interior(path)
        
    # Boundary Integral Estimator
    for l in range(N_b):
        # Sample boundary vertex
        path_l, boundary_vertex = sample_boundary(scene_params)
        
        # Compute time of flight for boundary path
        tof_l = compute_time_of_flight(path_l)
        
        # Compute boundary contribution and normal velocity
        # Assume S_e is continuous, so Delta_Omega_k[S_e] vanishes
        delta_f_T = compute_boundary_contribution(path_l)
        V_partial_B = compute_normal_velocity(boundary_vertex)
        
        # Bin contributions
        bin_index_l = map_to_bin(tof_l, temporal_bins)
        transient_image[bin_index_l] += delta_f_T
        gradient_boundary += (delta_f_T * V_partial_B) / sampling_probability_boundary(path_l)
        
    # Aggregate gradients
    gradient = (gradient_interior / N_i) + (gradient_boundary / N_b)
    
    return gradient, transient_image

def trace_path(scene_params):
    """
    Traces a path using bidirectional path tracing with automatic differentiation.
    Returns path, throughput, and derivative of throughput w.r.t scene_params.
    """
    # Implementation details of BDPT with AD are assumed to be handled by the framework
    # extending Zhang et al. [2020]
    pass

def compute_time_of_flight(path):
    """
    Computes the total time of flight for a path.
    """
    tof = 0.0
    for vertex in path:
        tof += distance_to_previous_vertex(vertex) / speed_of_light
    return tof

def compute_interior_contribution_derivative(path, throughput_deriv):
    """
    Computes the derivative of the interior path integral function.
    """
    # Uses automatic differentiation on top of next-event estimation
    # and radiance-based multiple importance sampling
    pass

def compute_boundary_contribution(path):
    """
    Computes the boundary contribution function Delta f_T.
    """
    pass

def compute_normal_velocity(vertex):
    """
    Computes the scalar normal velocity V = n . v.
    """
    n = vertex.normal
    v = vertex_velocity(vertex) # Velocity of manifold evolution
    return dot(n, v)

def sampling_probability_interior(path):
    """
    Returns the probability of sampling the material path for the interior estimator.
    """
    pass

def sampling_probability_boundary(path):
    """
    Returns the probability of sampling the material path for the boundary estimator.
    """
    pass
```

## Subtopics

### Steady-State Differential Rendering
Handles independent scattering events using the 2D Reynolds transport theorem. It fails to converge for time-of-flight effects and transparent objects where transient effects are significant [S1].

### Finite Differences
Scales linearly with the number of optimized parameters and requires rendering multiple perturbed scenes. It is slower and noisier than differentiable transient rendering for scenes with multiple parameters [S1].

## Operating characteristics
*   **Time Complexity**: The overhead of computing derivatives compared to standard transient rendering is scene-dependent. It ranges from 10% for NLOS results (Wavy and Corners scenes) to 72% in the worst case (Tower scene) [S1].
*   **Memory Complexity**: The method is memory-bounded due to the need to compute a full transient sequence, limiting scalability to thousands of parameters [S1].
*   **Performance**: In the Wavy scene, differentiable transient rendering takes 36 minutes per iteration, compared to 61 minutes for finite differences [S1].
*   **Strengths**: Provides less noisy results than finite differences for equal computation time. Converges faster for index of refraction optimization in transparent objects and converges where steady-state methods fail [S1].
*   **Limitations**: Lacks reciprocity. Memory-bounded, limiting scalability.

## Background
Differentiable Transient Rendering was introduced by Zhang et al. [2020] as an extension of their steady-state differentiable path tracing method [S1]. It generalizes the steady-state differential path integral to higher-dimensional domains to handle time-of-flight effects [S1]. The method is implemented on top of the code provided by Zhang et al. [2020], extending the bidirectional path tracer [S1].

## Sources
[S1] Differentiable transient rendering, https://doi.org/10.1145/3478513.3480498, fetched 2026-10-28
