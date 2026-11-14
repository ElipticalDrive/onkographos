# Metropolis Light Transport for Participating Media

Also called: MLT, MLT for Participating Media, Metropolis Light Transport (Pauly et al.), Metropolis Sampling for Light Transport

## Overview
Metropolis Light Transport (MLT) for participating media is a stochastic rendering algorithm that samples light transport paths in scenes with heterogeneous participating media by using Markov Chain Monte Carlo (MCMC) to target the measurement contribution function directly [S1, S3]. Unlike standard importance sampling methods that only consider factors of the contribution function, MLT generates paths proportional to their actual contribution to the image, allowing it to efficiently handle complex lighting effects such as caustics and multiple scattering in inhomogeneous media [S1, S3]. It distinguishes itself from its siblings by being an unbiased method that handles general geometric models and inhomogeneous media with multiple scattering while using little storage, though it is view-dependent [S1].

## How it works
MLT extends the Metropolis-Hastings algorithm to the space of light transport paths. The core challenge in participating media is that the transmittance along a path in heterogeneous media is analytically intractable, making the exact evaluation of the path contribution function $f(\bar{x})$ impossible [S4]. Pauly et al. [67] proposed using MLT for this purpose, relying on the ability to evaluate the contribution function (or a proxy) to guide the random walk [S1].

### Inputs and Outputs
- **Inputs**: Scene geometry, material properties (scattering coefficients $\sigma_s$, absorption coefficients $\sigma_a$, phase function $p$), light source definitions, camera parameters, and an initial set of seed paths.
- **Outputs**: An unbiased estimate of the pixel values (radiance) for the rendered image.

### Data Structures
The algorithm operates on paths $\bar{x}$ in the path space $\mathcal{P}$. A path $\bar{x}$ consists of a sequence of vertices $x_0, x_1, \dots, x_k$.
- Vertices $x_0$ (camera) and $x_{k+1}$ (light) are fixed or sampled from distributions.
- Intermediate vertices $x_i$ can be on surfaces (area integration, $d\mu(x_i) = dA(x_i)$) or in media (volume integration, $d\mu(x_i) = dV(x_i)$) [S4].
- The path probability density $p(\bar{x})$ is defined as:
  $$ p(\bar{x}) = \frac{f(\bar{x})}{b} $$
  where $b = \int_{\mathcal{P}} f(\bar{y}) d\mu(\bar{y})$ is the integral of the measurement contribution function over the path space [S3].

### Steps
1. **Initialization**:
   - Estimate the total image brightness (the constant $b$) using a suitable rendering algorithm (e.g., Bidirectional Path Tracing) with a low number of samples ($10^4$–$10^5$) [S3].
   - Generate a set of seed paths using standard path tracing techniques [S4].
   - Select one of the approximation samples as the initial state $\bar{x}_0$ for the Metropolis phase with probability proportional to $f(\bar{x})/p(\bar{x})$ [S3]. This sample is expected to be distributed according to the stationary density, avoiding start-up bias without discarding samples [S3].

2. **Mutation Loop**:
   - For each new state, generate a tentative sample $\bar{y}$ from the current sample $\bar{x}$ using a tentative transition function $T(\bar{x} \rightarrow \bar{y})$ [S3].
   - Common mutations include adding a new vertex to the path (e.g., inserting a scattering event in the medium) or perturbing existing vertices [S1].
   - Calculate the acceptance probability:
     $$ a(\bar{x} \rightarrow \bar{y}) := \min \left\{ 1, \frac{f(\bar{y})T(\bar{y} \rightarrow \bar{x})}{f(\bar{x})T(\bar{x} \rightarrow \bar{y})} \right\} $$
   - Accept or reject the proposal $\bar{y}$ based on this value [S3]. If accepted, $\bar{y}$ becomes the new current path; otherwise, the current path $\bar{x}$ is repeated.

3. **Rendering**:
   - Deposit a certain amount of energy to the pixel each path passes through, updating the image [S1].
   - The paths are distributed proportionally to their contribution to the final image [S1].
   - The measurement contribution function $f_j(\bar{x})$ is split into a pixel filter function $h_j(\bar{x})$ and a remainder $f(\bar{x})$ common to every pixel [S3]. The estimator for pixel $j$ is:
     $$ F_{j,N} = \frac{1}{N} \sum_{i=1}^N h_j(\bar{X}_i) b $$
     where $N$ is the number of samples [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def metropolis_light_transport(scene, camera, num_chains, steps_per_chain):
    # 1. Initialization
    # Estimate b using a low-sample BDPT or similar method
    b_estimate = estimate_total_brightness(scene, camera, num_samples=10000)
    
    # Generate seed paths using standard path tracing
    seed_paths = generate_seed_paths(scene, camera, num_samples=num_chains)
    
    # Select initial paths with probability proportional to f/p
    # Note: In practice, f is approximated or evaluated via unbiased estimators
    initial_paths = select_initial_paths(seed_paths, b_estimate)
    
    image_buffer = zeros(camera.resolution)
    
    for path in initial_paths:
        current_path = path
        for step in range(steps_per_chain):
            # 2. Mutation
            # Choose mutation type (e.g., add vertex, perturb vertex)
            mutation_type = random_choice(['add_vertex', 'perturb_vertex'])
            
            if mutation_type == 'add_vertex':
                tentative_path = add_scattering_vertex(current_path, scene)
            elif mutation_type == 'perturb_vertex':
                tentative_path = perturb_vertex(current_path, scene)
            
            # 3. Acceptance Test
            # Calculate f(current_path) and f(tentative_path)
            # In heterogeneous media, this requires unbiased estimation (e.g., PSSMLT)
            f_current = evaluate_contribution(current_path, scene)
            f_tentative = evaluate_contribution(tentative_path, scene)
            
            # Calculate transition probabilities T(x->y) and T(y->x)
            # For symmetric perturbations, T(x->y) = T(y->x)
            T_current_to_tentative = calculate_transition_prob(current_path, tentative_path, mutation_type)
            T_tentative_to_current = calculate_transition_prob(tentative_path, current_path, mutation_type)
            
            # Acceptance probability
            acceptance_ratio = (f_tentative * T_tentative_to_current) / (f_current * T_current_to_tentative)
            acceptance_prob = min(1.0, acceptance_ratio)
            
            if random_uniform() < acceptance_prob:
                current_path = tentative_path
            
            # 4. Deposit Energy
            pixel = get_pixel_from_path(current_path, camera)
            image_buffer[pixel] += evaluate_pixel_contribution(current_path, camera) / b_estimate
            
    return image_buffer

def evaluate_contribution(path, scene):
    """
    Evaluates the measurement contribution function f(x).
    For participating media, this involves computing transmittance.
    In standard MLT, this is intractable for heterogeneous media.
    [G] In a full implementation, this would use an unbiased estimator like PSSMLT.
    """
    # Simplified for demonstration: assumes homogeneous or pre-computed transmittance
    transmittance = compute_transmittance(path, scene)
    scattering_weights = compute_scattering_weights(path, scene)
    return transmittance * scattering_weights

def calculate_transition_prob(x, y, mutation_type):
    """
    Calculates the probability of transitioning from x to y.
    For symmetric mutations (e.g., Gaussian perturbation), this is equal to T(y->x).
    """
    # [G] Specific implementation depends on the mutation kernel.
    # For adding a vertex, it involves the probability of choosing to add a vertex
    # and the probability of sampling the new vertex position.
    return 1.0 # Placeholder for symmetric case
```

### Parameters
- **Number of chains**: Determined by the number of seed paths.
- **Steps per chain**: Number of mutations per chain. Longer chains improve correlation but increase computation time.
- **Mutation types**: Typically include adding a vertex (inserting a scattering event) and perturbing existing vertices. A good proposal strategy exploits coherence in the scene and maintains low correlation between subsequent samples [S3].

### Edge Cases and Pitfalls
- **Ergodicity**: The mutation alone cannot ensure ergodicity and may get stuck in isolated areas. Independent paths must be generated at random intervals by the underlying path tracer (feeding fresh uniform random numbers) to guarantee the algorithm explores the entire space [S3].
- **Intractable Transmittance**: In scenes with heterogeneous participating media, computing the transmittance along a path is analytically intractable, making the MH acceptance probability impossible to evaluate in closed form [S4]. This requires the use of unbiased estimators (e.g., PSSMLT) or biased approximations (which break the theoretical guarantees of MLT) [S4].
- **View Dependency**: MLT is a view-dependent method, meaning it must be re-run for each camera position [S1].

## Subtopics

### [Pseudo-marginal Metropolis Light Transport (PSSMLT)](12276-pseudo-marginal-metropolis-light-transport-pssmlt.md)
PSSMLT is a variant that enables MLT in scenes with heterogeneous participating media by replacing the intractable exact light path contribution function with a non-negative, unbiased estimator. Unlike methods that use biased approximations (such as deterministic ray-marching) which fail to target the correct density, PSSMLT constructs an extended target distribution in an auxiliary space to ensure the marginal distribution of sampled paths remains exact. It distinguishes itself from standard PSSMLT implementations that rely on delta tracking by utilizing ratio tracking, which is significantly faster and allows for longer Markov chains and better path space exploration.

### Primary Sample Space Metropolis Light Transport (PSSMLT)
Primary Sample Space Metropolis Light Transport (PSSMLT) is an MCMC method where mutations are applied to random number vectors in primary sample space, which are then used to generate light paths via traditional path tracing methods, rather than sampling directly in path space [S4]. While easy to implement, it is limited to the underlying path tracing techniques, restricting the possible exploration of the path space [S4].

### Metropolis Light Transport
Metropolis Light Transport (MLT) is a global illumination rendering method based on Monte Carlo methods that uses Markov chain Monte Carlo (MCMC) to sample light paths $\bar{x}_i$ according to a target distribution $\pi(\bar{x})$ proportional to the image contribution of the paths [S4]. Given a current path $\bar{x}_i$, a new path $\bar{x}_{i+1}$ is proposed using a conditional proposal density $q(\bar{x}_{i+1}|\bar{x}_i)$, and accepted with probability $\min(1, \frac{L(\bar{x}_{i+1}) q(\bar{x}_i|\bar{x}_{i+1})}{L(\bar{x}_i) q(\bar{x}_{i+1}|\bar{x}_i)})$ [S4]. The path integral is estimated by the average $\hat{I}_j = \frac{1}{N} \sum_{i=1}^N W_j f(\bar{x}_i)$, where $Z = \int_\Omega L(\bar{x})d\mu(\bar{x})$ is an unknown normalizing constant [S4].

### Energy Redistribution Path Tracing
Energy Redistribution Path Tracing (ERPT) is an extension to the original MLT algorithm that samples a large set of paths with standard path tracing methods and then runs many short MCMC chains starting from each initial sample [S4]. The main benefit of ERPT over original MLT is that a well stratified initial sample set can be preserved, e.g., using one path per pixel [S4].

## Operating characteristics
- **Time Complexity**: The convergence rate of MLT is generally slower than importance sampling for simple scenes but superior for complex lighting scenarios involving caustics or multiple scattering in media. The cost is dominated by the evaluation of the contribution function and the transition probabilities.
- **Memory Complexity**: MLT uses little storage as it only needs to store the current path and a few auxiliary variables [S1].
- **Strengths**: Unbiased results, handles general geometric models and inhomogeneous media, accounts for multiple scattering, and efficiently explores path space for difficult lighting situations [S1].
- **Limitations**: View-dependent, requires careful tuning of mutation strategies, and suffers from ergodicity issues if not combined with independent resampling [S3]. The intractability of transmittance in heterogeneous media requires additional techniques like PSSMLT [S4].
- **When to use**: Scenes with complex lighting, multiple scattering in participating media, or caustics where importance sampling fails.
- **When not to use**: Simple scenes with homogeneous media or direct illumination, where standard path tracing or BDPT is more efficient.

## Background
MLT was first presented by Veach and Guibas in 1997 [VG97] and strongly modified by Kelemen et al. in 2002 [KSKAC02]. Pauly et al. [67] extended MLT to incorporate volumetric scattering, producing unbiased results for general geometric models and inhomogeneous media with multiple scattering [S1]. The method is based on the Metropolis sampling technique for handling difficult sampling problems in computational physics [S1]. It is used in research and high-end rendering for simulating light transport in complex participating media.

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S3] Unbiased Global Illumination with Participating Media, http://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/ugiwpm.pdf, fetched 2026-10-30
[S4] Pseudo-marginal metropolis light transport, https://doi.org/10.1145/2820903.2820922, fetched 2026-10-30
