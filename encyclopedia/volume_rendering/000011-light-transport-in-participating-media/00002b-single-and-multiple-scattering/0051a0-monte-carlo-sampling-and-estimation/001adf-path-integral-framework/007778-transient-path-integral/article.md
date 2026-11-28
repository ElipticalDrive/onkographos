# Transient Path Integral

Also called: Time-resolved path integral, Transient light transport integral

## Overview
The transient path integral models the intensity $I$ recorded by a virtual sensor by integrating over all light paths $\bar{\mathbf{x}}$ and temporal delays $\mathbf{t}$, explicitly accounting for finite light propagation time [S1]. It extends the standard path integral framework by adding a temporal dimension to the path space, allowing for the simulation of time-of-flight phenomena where the arrival time of photons is resolved [S1].

## How it works
The method formulates light transport as a Monte Carlo integration problem over the joint space of geometric paths and time delays.

**Inputs and Outputs**
*   **Input:** A scene geometry, participating media properties (scattering function $\rho$, refractive indices $\eta$), light source emission function $L_e$, and sensor sensitivity function $W_e$.
*   **Output:** The scalar intensity $I$ recorded by the sensor.

**Data Structures**
A light path is represented as a sequence of vertices $\bar{\mathbf{x}} = \mathbf{x}_0, \dots, \mathbf{x}_k$ and a corresponding sequence of temporal delays $\mathbf{t} = t_0, \dots, t_k$ [S1].
*   $\mathbf{x}_0$: Vertex on the light source.
*   $\mathbf{x}_k$: Vertex on the sensor.
*   $\mathbf{x}_i$ ($0 < i < k$): Intermediate vertices (scattering events).
*   $t_i$: Temporal delay accumulated at vertex $\mathbf{x}_i$.

**Formulas**
The total intensity is defined as [S1]:

$$I = \int_{\Omega} \int_{\mathcal{T}} f_i(\bar{\mathbf{x}}, \mathbf{t}) d\mu(\mathbf{t}) d\mu(\bar{\mathbf{x}})$$

where:
*   $\Omega = \cup_{k=1}^{\infty} \Omega_k$ is the path space, with $\Omega_k$ being the space of all paths with $k$ scattering vertices.
*   $\mathcal{T}$ is the space of temporal delays.
*   $d\mu(\bar{\mathbf{x}})$ is the differential geometric measure.
*   $d\mu(\mathbf{t})$ denotes temporal integration at each vertex.

The path contribution function $f_i(\bar{\mathbf{x}}, \mathbf{t})$ is given by [S1]:

$$f_i(\bar{\mathbf{x}}, \mathbf{t}) = L_e(\mathbf{x}_0 \rightarrow \mathbf{x}_1, t_0) \mathfrak{T}(\bar{\mathbf{x}}, \mathbf{t}) W_e(\mathbf{x}_{k-1} \rightarrow \mathbf{x}_k, t_k + \text{tof}(\bar{\mathbf{x}}))$$

where:
*   $L_e(\mathbf{x}_0 \rightarrow \mathbf{x}_1, t_0)$ is the light source emission function.
*   $W_e(\mathbf{x}_{k-1} \rightarrow \mathbf{x}_k, t_k + \text{tof}(\bar{\mathbf{x}}))$ is the sensor sensitivity function, dependent on the total time-of-flight.
*   $\mathfrak{T}(\bar{\mathbf{x}}, \mathbf{t})$ is the light throughput.

The light throughput $\mathfrak{T}(\bar{\mathbf{x}}, \mathbf{t})$ is defined as [S1]:

$$\mathfrak{T}(\bar{\mathbf{x}}, \mathbf{t}) = \left[ \prod_{i=1}^{k-1} \rho(\mathbf{x}_i, t_i) \right] \left[ \prod_{i=0}^{k-1} G(\mathbf{x}_i \leftrightarrow \mathbf{x}_{i+1}) V(\mathbf{x}_i \leftrightarrow \mathbf{x}_{i+1}) \right]$$

where:
*   $\rho(\mathbf{x}_i, t_i)$ is the scattering function at vertex $\mathbf{x}_i$ and time $t_i$.
*   $G(\mathbf{x}_i \leftrightarrow \mathbf{x}_{i+1})$ is the geometric function between adjacent vertices.
*   $V(\mathbf{x}_i \leftrightarrow \mathbf{x}_{i+1})$ is the visibility function between adjacent vertices.

The total time-of-flight $\text{tof}(\bar{\mathbf{x}})$ is calculated as [S1]:

$$\text{tof}(\bar{\mathbf{x}}) = c^{-1} \sum_{i=1}^{k-1} \|\mathbf{x}_i - \mathbf{x}_{i-1}\| \eta_{i-1} + \sum_{i=0}^{k-1} t_i$$

where:
*   $c$ is the speed of light.
*   $\eta_{i-1}$ is the refractive index of the medium between vertices $\mathbf{x}_{i-1}$ and $\mathbf{x}_i$. Note: The source formula uses $\eta_i$ in the sum index context, but physically $\eta$ corresponds to the medium traversed. The source text specifies $\eta_i$ as the refractive index of the medium between $\mathbf{x}_i$ and $\mathbf{x}_{i+1}$ [S1]. In the summation $\sum_{i=0}^{k-1}$, the term $\|\mathbf{x}_i - \mathbf{x}_{i-1}\|$ is ambiguous for $i=0$. Standard path integral conventions imply the sum starts from the first segment. The source formula is: $\text{tof}(\bar{\mathbf{x}}) = c^{-1} \sum_{i=0}^{k-1} \|\mathbf{x}_i - \mathbf{x}_{i-1}\| \eta_i + \sum_{i=0}^{k-1} t_i$. This implies indices might be shifted or $\mathbf{x}_{-1}$ is defined. Given $\mathbf{x}_0$ is on the light, the first segment is $\mathbf{x}_0 \to \mathbf{x}_1$. The source formula likely intends $\sum_{i=1}^{k-1}$ for the distance terms or assumes $\mathbf{x}_{-1}$ is a virtual source point. However, strictly following the source [S1]:
    $$\text{tof}(\bar{\mathbf{x}}) = c^{-1} \sum_{i=0}^{k-1} \|\mathbf{x}_i - \mathbf{x}_{i-1}\| \eta_i + \sum_{i=0}^{k-1} t_i$$
    With $\eta_i$ being the refractive index between $\mathbf{x}_i$ and $\mathbf{x}_{i+1}$. This creates an index mismatch for the first term if $i=0$. We assume the standard physical interpretation: the sum covers segments $0 \to 1$ up to $k-1 \to k$. Let $j$ be the segment index. The source notation is specific. We retain the source's symbolic representation but note the potential indexing ambiguity [S1].

**Parameters**
*   $c$: Speed of light.
*   $\eta_i$: Refractive index of the medium between vertices $\mathbf{x}_i$ and $\mathbf{x}_{i+1}$ [S1].

**Pseudocode**
The following pseudocode computes the time-of-flight for a given path, a critical component for evaluating the sensor sensitivity $W_e$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_time_of_flight(path_vertices, refractive_indices, time_delays, speed_of_light):
    """
    Computes the total time-of-flight for a light path.
    
    Args:
        path_vertices: List of vertices [x_0, x_1, ..., x_k].
        refractive_indices: List of refractive indices [eta_0, ..., eta_{k-1}],
                             where eta_i is the index between x_i and x_{i+1}.
        time_delays: List of temporal delays [t_0, ..., t_k].
        speed_of_light: Scalar c.
        
    Returns:
        Total time-of-flight tof.
    """
    k = len(path_vertices) - 1
    tof = 0.0
    
    # Sum of propagation times through media
    # Source formula: c^-1 * sum_{i=0}^{k-1} ||x_i - x_{i-1}|| * eta_i
    # Note: The source formula index i=0 implies x_{-1}. 
    # Standard interpretation for path x_0...x_k involves segments (x_0,x_1) to (x_{k-1},x_k).
    # We assume the source formula intends to sum over the k segments.
    # If the source strictly means i=0 to k-1 with x_{i-1}, it is ill-defined for i=0.
    # We implement the physically consistent version: sum over segments j=0 to k-1.
    
    for j in range(k):
        # Segment from x_j to x_{j+1}
        # Distance
        dist = distance(path_vertices[j], path_vertices[j+1])
        # Refractive index for this segment
        eta = refractive_indices[j]
        # Time = distance * refractive_index / c
        tof += (dist * eta) / speed_of_light
        
    # Sum of temporal delays
    for t in time_delays:
        tof += t
        
    return tof

def evaluate_transient_path_contribution(path_vertices, time_delays, scene):
    """
    Evaluates the path contribution f_i for a transient path.
    """
    k = len(path_vertices) - 1
    
    # 1. Light Emission L_e(x_0 -> x_1, t_0)
    # Assuming x_0 is on light, x_1 is first scatter
    le = scene.light_emission(path_vertices[0], path_vertices[1], time_delays[0])
    
    # 2. Throughput T
    throughput = 1.0
    
    # Product of scattering functions rho(x_i, t_i) for i=1 to k-1
    for i in range(1, k):
        rho = scene.scattering_function(path_vertices[i], time_delays[i])
        throughput *= rho
        
    # Product of geometric G and visibility V for i=0 to k-1
    for i in range(k):
        g = scene.geometric_function(path_vertices[i], path_vertices[i+1])
        v = scene.visibility_function(path_vertices[i], path_vertices[i+1])
        throughput *= (g * v)
        
    # 3. Sensor Sensitivity W_e
    # W_e(x_{k-1} -> x_k, t_k + tof)
    tof = compute_time_of_flight(path_vertices, scene.refractive_indices, time_delays, scene.speed_of_light)
    we = scene.sensor_sensitivity(path_vertices[k-1], path_vertices[k], time_delays[k] + tof)
    
    return le * throughput * we
```

**Edge Cases and Pitfalls**
*   **Indexing Ambiguity:** The source formula for $\text{tof}$ sums from $i=0$ to $k-1$ for the distance term $\|\mathbf{x}_i - \mathbf{x}_{i-1}\|$. For $i=0$, $\mathbf{x}_{-1}$ is undefined. Implementers must clarify if $\mathbf{x}_0$ is the source point (distance 0) or if the sum starts at $i=1$ for the first segment $\mathbf{x}_0 \to \mathbf{x}_1$. The physical interpretation requires summing over all $k$ segments.
*   **Temporal Discontinuities:** The integrand may contain discontinuities due to visibility changes or edge crossings. Handling these requires specialized sampling techniques such as edge sampling or reparameterization [S1].
*   **Precision:** The time-of-flight values can be very small. Double precision is recommended for $c$ and distance calculations to avoid numerical errors in $W_e$ evaluation.

## Operating characteristics
*   **Complexity:** The complexity is exponential in the number of bounces $k$ due to the path space dimensionality, similar to standard path tracing. The temporal dimension adds a degree of freedom for each vertex.
*   **Performance:** Not given in the sources.
*   **Strengths:** Accurately models time-resolved light transport, enabling the simulation of ultrafast phenomena and time-of-flight imaging.
*   **Limitations:** High variance due to the additional temporal dimension and potential discontinuities. Requires careful handling of temporal boundaries and visibility changes [S1].

## Background
The transient path integral was introduced by Jarabo et al. [2014] as an extension of the path integral formulation [Veach 1997] to include the temporal domain [S1]. It generalizes the standard assumption that emission and sensor response are orders of magnitude larger than light propagation time. Recent work has extended this to differentiable rendering, allowing for gradient-based optimization in time-resolved scenes [S1].

## Sources
[S1] Differentiable transient rendering, https://doi.org/10.1145/3478513.3480498, fetched 2026-10-28
