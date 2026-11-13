# Transient Photon Beams

Also called: PTPB, Progressive Spatio-Temporal Beam Density Estimation, Progressive Transient Photon Beams, progressive transient path tracing with temporal KDE (PTPT) comparison target

## Overview
Transient Photon Beams (TPB) is a method for rendering time-resolved light transport in participating media that extends steady-state photon beams by incorporating photon time-of-flight along continuous beam trajectories [S1, S2]. It addresses the high variance and sparse temporal sampling inherent in point-based photon methods by leveraging the spatial continuity of photon beams to densely populate the temporal domain [S1]. Unlike its sibling [Photon Beam Mapping](7874-photon-beam-mapping.md) which handles steady-state, TPB accounts for propagation and scattering delays, and unlike [Photon Marching](4775-photon-marching.md) which uses marching cubes for isosurfaces, TPB uses kernel density estimation for volumetric radiance [S1, S2]. The progressive variant, Progressive Transient Photon Beams (PTPB), further reduces bias and variance by iteratively accumulating independent beam sets with progressively shrinking kernel bandwidths, achieving an optimal asymptotic convergence rate of $O(n^{-2/3})$ [S1, S3].

## How it works
The method consists of two main phases: Photon Tracing (building the beam map) and Radiance Estimation (rendering), optionally repeated in a progressive loop.

### Inputs and Outputs
**Inputs:**
*   Scene geometry and participating media properties (scattering coefficient $\mu_s$, extinction coefficient $\mu_t$, phase function $\rho$, index of refraction $\eta_m$).
*   Light source emission function $L_e$.
*   Camera ray $r = (\mathbf{x}_r, -\vec{\omega}_r)$ and time $t$.
*   Number of iterations $N$ (for progressive mode).
*   Beams per iteration $M_b$.
*   Kernel parameters: spatial radius $R_b$, temporal width $\mathcal{T}$, and reduction exponents $\beta_R, \beta_\mathcal{T}, \alpha$.

**Outputs:**
*   Transient radiance $L(\mathbf{x}_r, \vec{\omega}_r, t)$ for each camera ray.

### Data Structures
**Photon Beam Map:**
A collection of photon beams. Each beam $b$ stores:
*   Origin position $\mathbf{x}_b \in \mathbb{R}^3$.
*   Direction $\vec{\omega}_b \in \mathbb{R}^3$ (unit vector).
*   Power $\Phi_b \in \mathbb{R}$.
*   Creation time $t_b \in \mathbb{R}$.
*   Radius $R_b \in \mathbb{R}$ (for density estimation).

### Numbered Steps

#### 1. Photon Tracing (Pass 1)
For each iteration $i$ from $0$ to $N-1$:
1.  Trace $M_b$ photon random walks (subpaths) from the light source.
2.  For each subpath $\bar{\mathbf{x}}_l = \mathbf{x}_0 \dots \mathbf{x}_k$ (where $\mathbf{x}_0$ is on the light source):
    a.  Define $k-1$ photon beams. For each vertex $j$ from $0$ to $k-2$:
        i.  Beam origin: $\mathbf{x}_{b_j} = \mathbf{x}_j$.
        ii. Beam direction: $\vec{\omega}_{b_j} = \frac{\mathbf{x}_{j+1} - \mathbf{x}_j}{\|\mathbf{x}_{j+1} - \mathbf{x}_j\|}$.
        iii. Compute beam creation time $t_{b_j}$:
        $$t_{b_j} = \sum_{m=0}^{j-1} \tau_m + \sum_{m=0}^{j-1} \Delta t(\mathbf{x}_m, \mathbf{x}_{m+1})$$
        where $\tau_m$ is the scattering delay at vertex $\mathbf{x}_m$ and $\Delta t$ is the propagation time.
        iv. Compute beam power $\Phi_{b_j}$:
        $$\Phi_{b_j} = \frac{L_e(\mathbf{x}_0 \to \mathbf{x}_j, \tau_0) T(\bar{\mathbf{x}}_j, \bar{\tau}_j)}{M \prod_{m=0}^{j} p(\mathbf{x}_m, \tau_m)}$$
        where $M$ is the total number of photon walks, $p$ is the sampling probability, and $T$ is the throughput:
        $$T(\bar{\mathbf{x}}_j, \bar{\tau}_j) = \left[ \prod_{m=1}^{j-1} \rho(\mathbf{x}_m, \tau_m) \right] \left[ \prod_{m=0}^{j-1} G(\mathbf{x}_m, \mathbf{x}_{m+1}) V(\mathbf{x}_m, \mathbf{x}_{m+1}) \right]$$
        $G$ is the geometry term, $V$ is the visibility term.
        v. Store beam $b_j$ in the map for iteration $i$.

#### 2. Radiance Estimation (Pass 2)
For each camera ray $r = (\mathbf{x}_r, -\vec{\omega}_r)$ at time $t$:
1.  Initialize radiance $L = 0$.
2.  Find all beams $b$ in the map that intersect the ray within their spatial kernel radius $R_b$. Let this set be $R_b$.
3.  For each beam $b \in R_b$:
    a.  Compute the ray-beam intersection interval $[s_r^-, s_r^+]$ along the ray and $[s_b^-, s_b^+]$ along the beam.
    b.  Compute the angle $\theta_b$ between $\vec{\omega}_b$ and $\vec{\omega}_r$.
    c.  Compute the temporal footprint $[t^-, t^+]$ based on the kernel type:
        *   **For 2D Kernel (Steady-state-like or non-progressive):**
            $$t^- = t_b + t_r + \frac{\eta_m}{c}(s_r^- + s_b^-)$$
            $$t^+ = t_b + t_r + \frac{\eta_m}{c}(s_r^+ + s_b^+)$$
            where $t_r$ is the camera ray start time.
            If $t \notin (t^-, t^+)$, contribution is 0.
            Otherwise, compute spatial radiance $L_{b|2D}$ and apply temporal window $W_{2D}(t)$:
            $$L_{b|2D} = K_{2D}(R_b) \Phi_b \rho(\theta_b) \mu_s \frac{e^{-\mu_t (s_c^- - s_c^+)} (|\cos \theta_b| - 1)}{e^{\mu_t (s_r^- + s_b^-)} \mu_t (|\cos \theta_b| - 1)}$$
            $$W_{2D}(t) = \begin{cases} \frac{1}{t^+ - t^-} & \text{if } t \in (t^-, t^+) \\ 0 & \text{otherwise} \end{cases}$$
            Contribution: $L_b = L_{b|2D} \cdot W_{2D}(t)$.
        *   **For 1D Kernel (Progressive):**
            Compute propagation time from beam origin to intersection:
            $$\Delta t_{prop} = \frac{\eta_m}{c} s_b$$
            Compute propagation time from intersection to camera:
            $$\Delta t_{cam} = \frac{\eta_m}{c} s_r$$
            Total time at intersection: $t_{int} = t_b + \Delta t_{prop} + \Delta t_{cam}$.
            Compute spatial radiance $L_{b|1D}$:
            $$L_{b|1D} = K_{1D}(R_b) \Phi_b \rho(\theta_b) \mu_s \frac{e^{-\mu_s s_b} e^{-\mu_s s_r}}{\sin \theta_b}$$
            Apply temporal kernel $K_\mathcal{T}$ centered at $t_b$:
            $$L_b = L_{b|1D} \cdot K_\mathcal{T}(t - t_b)$$
            *Note: In the progressive formulation, the temporal kernel is convolved with the spatial estimate.*
    d.  Add $L_b$ to $L$.

#### 3. Progressive Convergence (Optional)
If using the progressive variant:
1.  Maintain the accumulated estimate $\hat{L}_n$.
2.  After iteration $n$, update the estimate:
    $$\hat{L}_n(\mathbf{x}_r, \vec{\omega}_r, t) = \frac{1}{n} \sum_{i=0}^{n-1} \sum_{b \in B_i} L_b(\mathbf{x}_r, \vec{\omega}_r, t)$$
3.  Reduce kernel bandwidths for the next iteration $n+1$:
    $$\frac{R_{b|n+1}}{R_{b|n}} = \left( \frac{n+\alpha}{n+1} \right)^{\beta_R}$$
    $$\frac{\mathcal{T}_{n+1}}{\mathcal{T}_n} = \left( \frac{n+\alpha}{n+1} \right)^{\beta_\mathcal{T}}$$
    where $\beta_\mathcal{T} = 1 - \beta_R$.

### Parameters
*   $\eta_m$: Index of refraction of the medium. Default: 1.0 (vacuum/air) unless specified otherwise [S2].
*   $c$: Speed of light in vacuum ($\approx 3 \times 10^8$ m/s) [S1].
*   $\alpha$: Variance increase control parameter. Range $[0, 1]$. Optimal value: $2/3$ [S1, S3].
*   $\beta_\mathcal{T}$: Temporal kernel reduction exponent. Optimal value: $1/2$ [S1, S3].
*   $\beta_R$: Spatial kernel reduction exponent. Derived as $1 - \beta_\mathcal{T} = 1/2$ [S1].

### Edge Cases and Pitfalls
*   **Temporal Footprint Size:** The temporal footprint of density estimation can be arbitrarily small, making the probability of finding a beam at a specific time very low. This is mitigated by using path reuse via density estimation or the progressive scheme [S1, S3].
*   **Temporal Bias:** Evenly distributing radiance across the temporal interval $[t^-, t^+]$ in 2D kernels introduces temporal bias because photon energy varies due to transmittance across the blur region [S1, S2].
*   **Varying Index of Refraction:** The straight-line distance formula for $\Delta t$ is only valid for constant $\eta$. In media with varying $\eta$, light follows the Eikonal equation, and $\Delta t$ must be computed via integration $\int \frac{\eta(\mathbf{x})}{c} d\mathbf{x}$ [S1, S3].
*   **Standard KDE Bias:** Standard kernel density estimation does not converge to the actual solution even with infinite photons due to bias. The progressive scheme is required for consistency [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_time_delay(x0, x1, eta_m, c):
    """Compute time delay for constant index of refraction."""
    dist = math.sqrt(sum((a-b)**2 for a, b in zip(x0, x1)))
    return (eta_m / c) * dist

def compute_beam_power(subpath_vertices, subpath_times, light_emission, M, p_sampling):
    """Compute flux for a photon beam segment."""
    # T is throughput
    # Product of scattering events rho
    rho_prod = 1.0
    for i in range(1, len(subpath_vertices) - 1):
        rho_prod *= subpath_times[i] # rho is modeled as time delay/scattering prob here for simplicity in formula context
    
    # Product of G and V terms
    gv_prod = 1.0
    for i in range(len(subpath_vertices) - 1):
        gv_prod *= geometry_term(subpath_vertices[i], subpath_vertices[i+1])
        gv_prod *= visibility_term(subpath_vertices[i], subpath_vertices[i+1])
        
    # Le term
    le = light_emission(subpath_vertices[0], subpath_vertices[-1], subpath_times[0])
    
    # Product of probabilities
    p_prod = 1.0
    for i in range(len(subpath_vertices)):
        p_prod *= p_sampling(subpath_vertices[i], subpath_times[i])
        
    return (le * rho_prod * gv_prod) / (M * p_prod)

def kernel_2d_temporal_window(t, t_minus, t_plus):
    if t_minus < t < t_plus:
        return 1.0 / (t_plus - t_minus)
    return 0.0

def kernel_1d_temporal(t, t_b, T_width):
    # Simplified Gaussian or box kernel for temporal domain
    # Assuming a normalized kernel K_T(t - t_b)
    # For implementation, use a standard PDF kernel like Gaussian
    return math.exp(-0.5 * ((t - t_b) / T_width)**2) / (T_width * math.sqrt(2 * math.pi))

def transient_photon_beams(scene, camera_rays, num_iterations, beams_per_iter, alpha, beta_T):
    beta_R = 1.0 - beta_T
    R_b = initial_spatial_radius()
    T = initial_temporal_width()
    
    # Map to store beams for each iteration
    # In progressive mode, we might store all beams or just accumulate radiance
    # Here we assume we store beams to re-evaluate with shrinking kernels
    beam_maps = [] 
    
    eta_m = scene.medium_index_of_refraction
    c = 299792458.0 # Speed of light
    
    for n in range(num_iterations):
        current_beams = []
        
        # Pass 1: Photon Tracing
        for _ in range(beams_per_iter):
            subpath = trace_photon_random_walk(scene)
            # subpath is list of (vertex, time_delay)
            
            # Create beams from subpath vertices
            for j in range(len(subpath) - 1):
                x_j, tau_j = subpath[j]
                x_j1, tau_j1 = subpath[j+1]
                
                # Beam origin and direction
                x_b = x_j
                omega_b = normalize(vector_sub(x_j1, x_j))
                
                # Beam creation time
                t_b = 0.0
                for m in range(j):
                    x_m, tau_m = subpath[m]
                    x_m1, _ = subpath[m+1]
                    t_b += tau_m
                    t_b += compute_time_delay(x_m, x_m1, eta_m, c)
                
                # Beam power
                # Simplified power calculation based on formula 9/10
                # Requires full subpath history for probabilities and throughput
                phi_b = compute_beam_power(
                    [v for v, _ in subpath[:j+1]],
                    [t for _, t in subpath[:j+1]],
                    scene.light_emission,
                    beams_per_iter, # M
                    scene.sampling_prob
                )
                
                current_beams.append({
                    'x': x_b,
                    'omega': omega_b,
                    'phi': phi_b,
                    't_b': t_b,
                    'R_b': R_b
                })
        
        beam_maps.append(current_beams)
        
        # Update kernel sizes for next iteration
        if n < num_iterations - 1:
            ratio = ((n + alpha) / (n + 1))
            R_b *= ratio ** beta_R
            T *= ratio ** beta_T

    # Pass 2: Radiance Estimation
    results = {}
    for ray in camera_rays:
        x_r, omega_r, t_cam = ray
        total_L = 0.0
        
        # Accumulate over all iterations for progressive estimate
        for n in range(num_iterations):
            iteration_L = 0.0
            current_beams = beam_maps[n]
            
            for b in current_beams:
                # Check intersection
                # Find s_r and s_b for ray-beam intersection
                # This is a geometric line-line intersection problem
                s_r_min, s_r_max, s_b_min, s_b_max = ray_beam_intersection(
                    x_r, omega_r, b['x'], b['omega'], b['R_b']
                )
                
                if s_r_min is None:
                    continue
                
                # Angle between beam and ray
                cos_theta = dot(b['omega'], omega_r)
                theta_b = math.acos(abs(cos_theta)) # Use absolute for phase function symmetry usually
                
                # Temporal Footprint Calculation
                # t_minus and t_plus for 2D kernel logic, or direct t for 1D
                # Using 1D progressive kernel as per formula 16
                
                # Propagation time from beam origin to intersection point on beam
                # s_b is distance along beam. Let's pick a representative point or integrate.
                # Formula 16 uses K_T(t - t_b). 
                # However, formula 13/14 in S1/S2 suggests t depends on s_r and s_b.
                # The progressive 1D kernel formulation (Eq 16) simplifies this to K_T(t - t_b).
                # But Eq 16 is derived from 1D spatial kernel. 
                # Let's use the explicit temporal footprint for accuracy if 2D-like, 
                # or the simplified 1D if strictly following Eq 16.
                # The text says "reformulate the 1D kernel ... by convolving with a 1D temporal kernel".
                # So we use K_T(t - t_b).
                
                # Spatial Radiance L_b|1D
                # mu_s is scattering coefficient
                mu_s = scene.medium_scattering_coeff
                mu_t = scene.medium_extinction_coeff
                
                # Exponential terms
                # e^(-mu_s * s_b) and e^(-mu_s * s_r)
                # Note: s_b and s_r in the formula refer to distances from origin/intersection
                # We need to integrate or pick a point. The formula implies a specific evaluation.
                # For implementation, we often evaluate at the closest point or integrate.
                # Given the complexity, we approximate with the midpoint or the start of the interval.
                
                # Let's use the start of the intersection interval for s_b and s_r
                s_b = s_b_min
                s_r = s_r_min
                
                # Transmittance terms
                # e^(-mu_t * (s_c^- - s_c^+)) is part of 2D. 
                # For 1D: e^(-mu_s s_b) e^(-mu_s s_r) / sin(theta_b)
                
                if sin(theta_b) == 0:
                    continue
                    
                spatial_part = (math.exp(-mu_s * s_b) * math.exp(-mu_s * s_r)) / math.sin(theta_b)
                
                # Kernel values
                K_1D = spatial_kernel_value(b['R_b']) # Canonical 1D kernel
                K_T = kernel_1d_temporal(t_cam, b['t_b'], T)
                
                # Phase function
                rho = scene.phase_function(theta_b)
                
                L_b = K_1D * b['phi'] * rho * mu_s * spatial_part * K_T
                iteration_L += L_b
            
            # Progressive Accumulation
            # L_hat_n = (1/n) * sum_{i=0}^{n-1} sum L_b
            # We accumulate the sum, then divide at the end or incrementally
            if n == 0:
                total_L = iteration_L
            else:
                # Update running average
                # L_hat_n = ((n-1)*L_hat_{n-1} + L_n) / n
                total_L = ((n * total_L) + iteration_L) / (n + 1)
                
        results[ray.id] = total_L
        
    return results
```

## Subtopics

### Photon Beam Diffusion
Uses diffusion approximation for dense media, whereas TPB uses ray tracing and density estimation for sparse to moderate media [Sibling Link].

### Photon Marching
Marches through the medium to find isosurfaces, whereas TPB estimates volumetric radiance via kernel density [Sibling Link].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M_b \cdot R_b)$, where $N$ is iterations, $M_b$ is beams per iteration, and $R_b$ is the number of beams intersecting a ray.
*   **Memory Complexity**: $O(N \cdot M_b)$ to store all beam sets for progressive convergence.
*   **Convergence Rate**: The asymptotic mean squared error (AMSE) converges at $O(n^{-2/3})$ with optimal parameters $\alpha=2/3$ and $\beta_\mathcal{T}=1/2$ [S1, S3]. This is slower than Progressive Transient Path Tracing with Temporal KDE (PTPT) which converges at $O(n^{-4/5})$, but TPB exhibits better temporal behavior with less variance on later timings and requires fewer iterations to achieve similar MSE [S1, S3].
*   **Strengths**: Robust for complex caustic light transport in media; consistent under finite memory; reduces variance compared to 0D point samples [S1].
*   **Limitations**: 3D kernel density estimation is impractical due to costly convolutions; temporal bias introduced by even distribution of radiance in 2D kernels; small temporal footprints can lead to low probability of beam hits [S1, S2].
*   **When to Use**: Scenes with complex light paths (caustics) in participating media where transient effects are important and standard path tracing is too noisy.
*   **When Not to Use**: Homogeneous media with no complex scattering where simpler methods suffice; scenes requiring very high temporal resolution where the bias from even distribution is unacceptable.

## Background
Transient Photon Beams was introduced by Julio Marco, Ibón Guillén, Wojciech Jarosz, Diego Gutierrez, and Adrian Jarabo from Universidad de Zaragoza and Dartmouth College [S1]. It builds on the steady-state Photon Beams algorithm by Jarosz et al. [JNSJ11] and the progressive photon beams method [JNT*11]. The method was developed to address the challenges of transient rendering in participating media, specifically the need to account for time-of-flight and the high variance of point-based methods. It was presented in the Computer Graphics Forum (CGF) in 019 [S1] and earlier in the Computer Graphics International (CEIG) workshop in 2017 [S2]. The method has been tested on scenes such as CORNELL SPHERES, MIRRORS, PUMPKIN, SOCCER, and JUICE [S1, S2].

## Sources
[S1] Progressive Transient Photon Beams, https://doi.org/10.1111/cgf.13600, fetched 2026-10-27
[S2] Transient Photon Beams, https://doi.org/10.2312/ceig.20171216, fetched 2026-10-26
[S3] Progressive Transient Photon Beams, https://arxiv.org/abs/1805.09562v1, fetched 2026-10-27
