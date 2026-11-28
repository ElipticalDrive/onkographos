# Noise-to-Noise Training for Volume Rendering

Also called: Appearance Estimator Training via Noise-to-Noise, Noise2Noise for Voxel Throughput

## Overview
This method trains a neural network to approximate the average voxel throughput function $\phi_{\mathcal{V}}(\omega_o, \omega_i)$ using noise-to-noise training [Lehtinen et al. 2018] with unbiased Monte Carlo estimates [S1]. It solves the problem of computing complex light transport within a voxel by replacing expensive offline integration with a learned estimator that generalizes across incident and exitant directions. Unlike standard path tracing which computes transport per pixel, this method precomputes the transport characteristics of the voxel geometry and medium, enabling efficient correlation-aware levels of detail [S1]. It differs from [Volumetric Radiance Estimators](818-volumetric-radiance-estimators.md) by focusing on the throughput function between specific boundary points rather than full radiance, and from [Delta tracking](985-delta-tracking.md) by using neural networks to approximate the integral rather than sampling the medium directly.

## How it works
The method trains a neural network to predict the throughput $\phi_{\mathcal{V}}$ for a given voxel $\mathcal{V}$, incident direction $\omega_i$, and exitant direction $\omega_o$. The training process generates noisy reference samples using an unbiased Monte Carlo estimator derived from path tracing within the voxel.

### Inputs and Outputs
*   **Input:** Voxel geometry $\mathcal{V}$, medium properties (scattering distribution $f$, extinction), incident direction $\omega_i$, exitant direction $\omega_o$.
*   **Output:** Neural network parameters approximating $\phi_{\mathcal{V}}(\omega_o, \omega_i)$.
*   **Training Data:** Pairs of noisy estimates $(\hat{\phi}, \tilde{\phi})$ generated via the MC estimator described below.

### Data Structures
The implementation requires:
1.  **Voxel Geometry Representation:** A representation allowing ray intersection tests within the voxel boundary $\partial\mathcal{V}$ [S1].
2.  **Neural Network:** Takes $\omega_i, \omega_o$ (and potentially voxel encoding) as input, outputs throughput value.
3.  **Ray Tracing Engine:** Capable of tracing rays against voxel geometry and performing shadow tests [S1].

### Formulas
The target function is the average voxel throughput [S1]:
$$ \phi_{\mathcal{V}}(\omega_o, \omega_i) = \mathbb{E} \left[ \int_{\partial\mathcal{V}} T_{\mathcal{V}}(\mathbf{p}_o, \omega_o; \mathbf{p}_i, \omega_i) |\omega_i \cdot \mathbf{n}_i| dA(\mathbf{p}_i) \right] $$
where $T_{\mathcal{V}}$ is the transmittance/throughput from exit point $\mathbf{p}_o$ to incidence point $\mathbf{p}_i$. The point of incidence $\mathbf{p}_i$ is determined by the null-scattering distribution $f(\omega_n, \mathbf{p}_i, \omega_i) = \delta(\omega_n - \omega_i)/|\mathbf{n}_i \cdot \omega_n|$ on the voxel boundary [S1].

The Monte Carlo estimator for training is [S1]:
$$ \phi_{\mathcal{V}}(\omega_o, \omega_i) = \mathbb{E} \left[ \sum_{n=1}^{\infty} \frac{\prod_{j=1}^n f(\omega_{j-1}, \mathbf{v}_j, \omega_j) |\omega_j \cdot \mathbf{n}_j|}{\prod_{j=1}^{n-1} p(\omega_j | \omega_{j-1}, \mathbf{v}_j)} V_{\mathcal{M}}(\mathbf{v}_n, \mathbf{p}_i) \right] $$
where $V_{\mathcal{M}}$ is visibility, $\mathbf{v}_j$ are path vertices, and $p$ are BSDF importance sampling densities [S1].

The training distribution for accepted exit points is [S1]:
$$ p_{\omega_o}^{\Phi}(\mathbf{p}_o) := \frac{p_{\omega_o}(\mathbf{p}_o)}{\int_{\partial\mathcal{V}} (1 - V_{\mathcal{M}}(\mathbf{p}_o, \mathcal{R}_{\partial\mathcal{V}}(\mathbf{p}_o, -\omega_o))) p_{\omega_o}(\mathbf{p}_o) dA(\mathbf{p}_o)} $$
where $p_{\omega_o}(\mathbf{p}_o) \propto |\omega_o \cdot \mathbf{n}_o|$ is the proposal distribution [S1].

### Parameters
*   **Learning Rate:** $\eta = 0.005$ for Adam optimizer [S1].
*   **Domain Extension:** Training domain extended by 0.5% of original voxel extent to fix boundary issues; inference queries original boundary [S1].
*   **Russian Roulette:** Applied for path tracing efficiency to bound infinite scatter events [S1].

### Steps
1.  **Sample Directions:** For each voxel, sample incident direction $\omega_i$ and exitant direction $\omega_o$ uniformly [S1].
2.  **Sample Exit Point:** Generate a point $\mathbf{p}_o$ on the projected voxel boundary along rays with direction $\omega_o$ using proposal distribution $p_{\omega_o}(\mathbf{p}_o) \propto |\omega_o \cdot \mathbf{n}_o|$ [S1].
3.  **Visibility Test:** Trace a ray against geometry within the voxel to test if $\mathbf{p}_o$ lies within the visible geometric area. Accept if visible, reject if occluded [S1].
4.  **Path Tracing:** Realize the estimator by recursive ray tracing starting from $\mathbf{p}_o$ in direction $\omega_o$. Accumulate throughput for light incident from $\omega_i$ if unshadowed. Importance sample new direction proportional to scattering distribution at hit point. Repeat until ray crosses voxel boundary [S1].
5.  **Shadow Testing:** Trace an additional shadow ray from each encountered interaction within the voxel to the voxel boundary in direction $\omega_i$ to detect blockers [S1].
6.  **Training:** Use the generated noisy estimates as both input and target for noise-to-noise training [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_throughput_sample(voxel, geometry, omega_i, omega_o):
    """
    Generate a noisy Monte Carlo estimate of voxel throughput phi_V(omega_o, omega_i).
    """
    # Step 1: Sample exit point p_o on voxel boundary
    # Proposal distribution p_omega_o(p_o) proportional to |omega_o . n_o|
    p_o = sample_point_on_projected_boundary(voxel, omega_o)
    
    # Step 2: Visibility test for p_o
    # Trace ray from p_o in direction -omega_o to check if visible from outside
    # Or trace against geometry within voxel to check if p_o is on visible surface
    # The source states: "Ray tracing against the geometry within the voxel tests, 
    # if the sampled point lies within the visible geometric area"
    is_visible = trace_ray_against_geometry(geometry, p_o, -omega_o)
    
    if not is_visible:
        return None  # Rejected sample
    
    # Step 3: Path tracing from p_o in direction omega_o
    throughput = 0.0
    current_pos = p_o
    current_dir = omega_o
    throughput_weight = 1.0
    
    while True:
        # Trace ray from current_pos in current_dir against voxel geometry
        hit, hit_point, hit_normal, hit_material = trace_ray(voxel.geometry, current_pos, current_dir)
        
        if hit is None:
            # Ray crossed voxel boundary without hitting geometry
            # Accumulate throughput for light incident from omega_i if unshadowed
            # Shadow test: trace ray from current_pos to voxel boundary in direction omega_i
            shadow_hit = trace_shadow_ray(voxel.geometry, current_pos, omega_i)
            if shadow_hit is None:
                # Unshadowed, accumulate throughput
                # The throughput contribution depends on the transmittance and scattering
                # For the first interaction, it's determined by the null-scattering distribution
                # Simplified: accumulate based on visibility and transmittance
                throughput += throughput_weight * compute_transmittance(voxel, current_pos, omega_i)
            break
        
        # Hit geometry within voxel
        # Shadow test for this hit point
        shadow_hit = trace_shadow_ray(voxel.geometry, hit_point, omega_i)
        
        if shadow_hit is None:
            # Unshadowed, accumulate throughput
            throughput += throughput_weight * compute_transmittance(voxel, hit_point, omega_i)
        
        # Importance sample new direction based on BSDF
        new_dir = sample_bsdf_direction(hit_material, hit_normal, current_dir)
        pdf = compute_bsdf_pdf(hit_material, hit_normal, current_dir, new_dir)
        
        # Update weight
        throughput_weight *= hit_material.scattering_phase_function(current_dir, new_dir) * abs(dot(new_dir, hit_normal)) / pdf
        
        # Russian Roulette for path termination
        if russian_roulette(throughput_weight) == 0:
            break
            
        current_pos = hit_point
        current_dir = new_dir
        
    return throughput

def train_appearance_network(voxel, geometry, network, optimizer):
    """
    Train the appearance network using noise-to-noise training.
    """
    # Sample directions uniformly
    omega_i = sample_uniform_direction()
    omega_o = sample_uniform_direction()
    
    # Generate noisy estimates
    # In noise-to-noise, we generate two independent noisy estimates
    # and use them as input and target
    estimate1 = generate_throughput_sample(voxel, geometry, omega_i, omega_o)
    estimate2 = generate_throughput_sample(voxel, geometry, omega_i, omega_o)
    
    if estimate1 is None or estimate2 is None:
        return  # Skip rejected samples
    
    # Forward pass
    predicted = network(omega_i, omega_o)
    
    # Loss: MSE between two noisy estimates
    loss = mse(predicted, estimate2)
    
    # Backpropagate
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

### Edge Cases and Pitfalls
*   **Boundary Issues:** The training domain is extended by 0.5% of the original voxel extent to fix boundary issues, but inference queries the original boundary [S1].
*   **Infinite Paths:** The sum in the estimator is theoretically infinite, but paths cross the voxel boundary after a finite number of interactions, reducing subsequent terms to zero [S1]. Russian Roulette is used to bound scatter events [S1].
*   **Rejection Sampling:** Points without ray intersections are rejected for appearance training, affecting the training distribution [S1].

## Subtopics

### Visibility Training
Trains a separate network for visibility within the voxel. Slower to converge due to higher number of encoding parameters [S1].

### Appearance Training
Trains the throughput function. Faster convergence than visibility training [S1].

## Operating characteristics
*   **Training Time:** Minutes for both visibility and appearance representations, compared to hours or days for previous work [S1].
*   **Memory Footprint:** Neural representations have a small memory footprint compared to storing full transport data [S1].
*   **Performance:** Compresses light transport, reducing average path length compared to brute-force path tracing [S1].
*   **Strengths:** Efficient preprocessing, practical for real-world asset pipelines [S1].
*   **Limitations:** Requires training, which adds complexity to the pipeline [S1].

## Background
The method is based on noise-to-noise training [Lehtinen et al. 2018] applied to volume rendering. It was introduced in "Neural Prefiltering for Correlation-Aware Levels of Detail" [S1]. The approach moves preprocessing to neural representations, making it more practical for real-world asset pipelines compared to previous methods requiring hours to days of training [S1].

## Sources
[S1] Neural Prefiltering for Correlation-Aware Levels of Detail, https://zenodo.org/records/8420313, fetched 2026-10-26
