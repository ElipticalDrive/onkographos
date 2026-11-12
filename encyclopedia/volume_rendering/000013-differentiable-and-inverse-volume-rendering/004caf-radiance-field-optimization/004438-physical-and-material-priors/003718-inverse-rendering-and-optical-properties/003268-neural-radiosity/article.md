# Neural Radiosity

Also called: Full-gradient method, Residual-based optimization

## Overview
Neural Radiosity is a method that represents the outgoing radiance field of a scene as a neural network and optimizes its parameters by minimizing the residual of the rendering equation [S1, unofficial][S3, unofficial]. It solves the problem of computing global illumination by treating the rendering equation as a fixed-point problem where the network output is the solution [S3, unofficial]. Unlike differentiable rendering methods that compute gradients via backpropagation through a rasterizer or path tracer, Neural Radiosity directly optimizes the radiance function itself, making it independent of specific camera viewpoints during training [S1, unofficial]. It is distinguished from its sibling [Differentiable Neural Radiosity](7410-differentiable-neural-radiosity.md) by the latter's use of a separate network to cache differential radiance, whereas Neural Radiosity uses a single network for the radiance field itself [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry (represented as a set of surfaces/meshes), material properties (BRDF $f$, emission $E$), and initial network parameters $\theta$.
*   **Output:** A neural network $L_\theta(x, \omega_o)$ that approximates the outgoing radiance at any point $x$ on the scene surface in direction $\omega_o$ [S3, unofficial].

**Data Structures:**
*   **Neural Network:** A Multi-Layer Perceptron (MLP) with parameters $\theta$. The standard implementation uses 6 fully-connected layers with 512 neurons per hidden layer [S3, unofficial].
*   **Inputs to Network:** Surface location $x$, outgoing direction $\omega_o$, surface normal, diffuse reflectance, and specular reflectance [S3, unofficial].
*   **Feature Grids:** Multi-resolution learnable feature vectors stored in sparse grids (e.g., hash grids) to encode high-frequency spatial details [S3, unofficial]. These are queried based on $x$ and concatenated with the network inputs.

**Formulas:**
The rendering equation is defined as:
$$
L(x, \omega_o) = E(x, \omega_o) + \int_{H^2} f(x, \omega_i, \omega_o) L(x'(x, \omega_i), -\omega_i) (\mathbf{n} \cdot \omega_i) d\omega_i
$$
where $x'(x, \omega_i)$ is the closest surface intersection of the ray originating at $x$ in direction $\omega_i$, $H^2$ is the hemisphere of incoming directions, and $\mathbf{n}$ is the surface normal [S1, unofficial][S3, unofficial].

The residual $r_\theta(x, \omega_o)$ is the difference between the left-hand side (LHS) and right-hand side (RHS) of the rendering equation when $L$ is replaced by the network prediction $L_\theta$:
$$
r_\theta(x, \omega_o) = L_\theta(x, \omega_o) - E(x, \omega_o) - \int_{H^2} f(x, \omega_i, \omega_o) L_\theta(x'(x, \omega_i), -\omega_i) (\mathbf{n} \cdot \omega_i) d\omega_i
$$
Note: In the gradient computation for inverse rendering, partial derivatives with respect to scene parameters $p$ are included [S3, unofficial]:
$$
r_\theta(x, \omega_o) = \partial_p L_\theta(x, \omega_o) - \partial_p E(x, \omega_o) - \int_{H^2} f(x, \omega_i, \omega_o) \partial_p L_\theta(x'(x, \omega_i), -\omega_i) (\mathbf{n} \cdot \omega_i) d\omega_i - \int_{H^2} \partial_p f(x, \omega_i, \omega_o) L_\theta(x'(x, \omega_i), -\omega_i) (\mathbf{n} \cdot \omega_i) d\omega_i
$$

The loss function is the squared $L_2$ norm of the residual integrated over all scene surfaces $M$ and hemispheres $H^2$:
$$
\mathcal{L}(\theta) = \int_M \int_{H^2} r_\theta(x, \omega_o)^2 d\omega_o dx
$$
This is estimated via Monte Carlo integration using $N$ samples $(x_j, \omega_{o,j})$ drawn from a distribution with density $p(x, \omega_o)$:
$$
\mathcal{L}(\theta) \approx \frac{1}{N} \sum_{j=1}^N \frac{r_\theta(x_j, \omega_{o,j})^2}{p(x_j, \omega_{o,j})}
$$

The gradient of the loss with respect to network parameters $\theta$ is:
$$
\nabla_\theta \mathcal{L}(\theta) \approx \frac{1}{N} \sum_{j=1}^N \frac{2 r_\theta(x_j, \omega_{o,j}) \nabla_\theta r_\theta(x_j, \omega_{o,j})}{p(x_j, \omega_{o,j})}
$$
where $\nabla_\theta r_\theta$ is computed by differentiating the residual expression. The incident integral in the residual is estimated using $M$ samples of incident directions $\omega_{i,j,k}$:
$$
\int_{H^2} \dots d\omega_i \approx \frac{1}{M} \sum_{k=1}^M \frac{f(x_j, \omega_{i,j,k}, \omega_{o,j}) \partial_p L_\theta(x'(x_j, \omega_{i,j,k}), -\omega_{i,j,k}) (\mathbf{n} \cdot \omega_{i,j,k})}{p(\omega_{i,j,k})}
$$
The gradient of the residual term involving the integral is approximated similarly [S3, unofficial]:
$$
\nabla_\theta r_\theta(x_j, \omega_{o,j}) \approx \nabla_\theta (\partial_p L_\theta(x_j, \omega_{o,j})) - \frac{1}{M} \sum_{k=1}^M \frac{f(x_j, \omega_{i,j,k}, \omega_{o,j}) \nabla_\theta (\partial_p L_\theta(x'(x_j, \omega_{i,j,k}), -\omega_{i,j,k})) (\mathbf{n} \cdot \omega_{i,j,k})}{p(\omega_{i,j,k})}
$$

**Parameters:**
*   **Network:** MLP, 6 layers, 512 hidden units [S3, unofficial].
*   **Sampling:** $x$ sampled uniformly on scene surfaces; $\omega_o$ sampled uniformly on the unit hemisphere (or sphere for two-sided BSDFs) [S3, unofficial].
*   **Incident Samples:** $M$ incident directions sampled uniformly over a hemisphere, often importance sampled from emitter and BSDF distributions [S3, unofficial].
*   **Optimization:** Stochastic Gradient Descent (SGD) or Adam [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Neural Radiosity Training Loop
# Inputs: scene geometry, materials, initial network parameters theta, learning rate eta
# Outputs: optimized network parameters theta

def neural_radiosity(scene, materials, theta, eta, num_samples_N, num_incident_M):
    # Initialize network parameters theta
    # Note: In practice, use an optimizer like Adam
    
    while not converged:
        # 1. Sample locations and outgoing directions
        # Sample x_j uniformly on all scene surfaces
        # Sample omega_o_j uniformly on unit hemisphere (or sphere for two-sided)
        samples = []
        for j in range(num_samples_N):
            x_j = sample_surface_uniform(scene)
            omega_o_j = sample_hemisphere_uniform(x_j.normal)
            
            # 2. Sample incident directions for the RHS integral
            # Sample omega_i_j_k uniformly over hemisphere, potentially importance sampled
            incident_samples = []
            for k in range(num_incident_M):
                omega_i_j_k = sample_incident_direction(scene, materials, x_j, omega_o_j)
                incident_samples.append(omega_i_j_k)
            
            samples.append((x_j, omega_o_j, incident_samples))
        
        # 3. Compute Loss and Gradient
        total_loss = 0.0
        total_grad = 0.0
        
        for (x_j, omega_o_j, incident_samples) in samples:
            # Evaluate L_theta(x_j, omega_o_j)
            L_theta = network_forward(theta, x_j, omega_o_j, materials.get_features(x_j))
            
            # Evaluate E(x_j, omega_o_j) and its gradient if needed
            E_val = materials.get_emission(x_j, omega_o_j)
            dE_dtheta = 0.0 # Emission is usually constant w.r.t network params
            
            # Evaluate RHS Integral: sum over incident directions
            rhs_integral = 0.0
            rhs_grad_integral = 0.0
            
            for omega_i in incident_samples:
                # Ray trace to find intersection x'
                x_prime = scene.ray_intersect(x_j, omega_i)
                
                # Evaluate L_theta at the hit point for the incoming direction
                # Note: The incoming direction to the hit point is -omega_i
                L_theta_prime = network_forward(theta, x_prime, -omega_i, materials.get_features(x_prime))
                
                # Get BRDF f(x_j, omega_i, omega_o_j)
                f_val = materials.get_brdf(x_j, omega_i, omega_o_j)
                
                # Compute weight: f * L_theta_prime * (n . omega_i) / p(omega_i)
                # p(omega_i) is the PDF of the sampling distribution
                pdf = get_sampling_pdf(omega_i)
                weight = (f_val * L_theta_prime * dot(x_j.normal, omega_i)) / pdf
                
                rhs_integral += weight
                
                # Compute gradient of the RHS term w.r.t theta
                # This requires backpropagating through L_theta_prime
                # d/dtheta (f * L_theta_prime) = f * dL_theta_prime/dtheta
                dL_prime_dtheta = network_backward(theta, x_prime, -omega_i, materials.get_features(x_prime))
                rhs_grad_integral += (f_val * dL_prime_dtheta * dot(x_j.normal, omega_i)) / pdf
            
            # Compute Residual r_theta
            # r_theta = L_theta - E - RHS_integral
            # For inverse rendering, include partial derivatives w.r.t scene params p
            # Here we assume standard radiosity optimization where p are not scene params but network params theta
            # So dE/dtheta = 0, df/dtheta = 0
            residual = L_theta - E_val - rhs_integral
            
            # Compute Gradient of Residual w.r.t theta
            # dr/dtheta = dL_theta/dtheta - d(E)/dtheta - d(RHS)/dtheta
            dL_dtheta = network_backward(theta, x_j, omega_o_j, materials.get_features(x_j))
            dRHS_dtheta = rhs_grad_integral
            
            grad_residual = dL_dtheta - dRHS_dtheta
            
            # Gradient of Loss: dL/dtheta = 2 * r_theta * dr/dtheta / N / p(x, omega_o)
            # Assuming uniform sampling, p(x, omega_o) is constant and absorbed into eta or normalized by N
            # Loss contribution
            loss_contribution = residual ** 2
            total_loss += loss_contribution
            
            # Gradient contribution
            grad_contribution = 2.0 * residual * grad_residual
            total_grad += grad_contribution
        
        # Update parameters
        # theta = theta - eta * (total_grad / N)
        update_parameters(theta, total_grad / num_samples_N, eta)
        
        # Check convergence based on loss change or max iterations
```

**Edge Cases and Pitfalls:**
*   **Bias and Variance:** The baseline gradient estimator is biased and has high variance due to the stochasticity of Monte Carlo integration in the RHS and its noisy derivatives [S2, unofficial]. This leads to slow training and occasionally darkened renders [S2, unofficial].
*   **Correlation Bias:** Using the same Monte Carlo samples for both factors in the gradient estimation (e.g., in the product $r_\theta \nabla_\theta r_\theta$) leads to $E(ab) \neq E(a)E(b)$, causing bias [S3, unofficial].
*   **Memory Intensity:** Storing the full computation graph of Monte Carlo rendering algorithms is memory intensive, making it prohibitive for complex scenes [S3, unofficial].
*   **Outdated Gradients:** Keeping $\partial_p L_\theta$ and $L_\phi$ outdated can introduce bias, though experiments show convergence improvements compared to previous methods [S3, unofficial].

## Subtopics

### [Differentiable Neural Radiosity](7410-differentiable-neural-radiosity.md)
A variant that uses a neural network to represent the *differential* radiance field, enabling view-independent gradient computation with constant time and memory complexity relative to path length. It avoids the quadratic time complexity of Radiative Backpropagation and the memory overhead of storing per-bounce data.

## Operating characteristics
*   **Time Complexity:** The training time depends on the number of samples $N$ and the number of incident samples $M$ per sample. Each iteration requires $N \times M$ ray intersections and network evaluations.
*   **Memory Complexity:** Memory usage is dominated by the storage of the neural network parameters and the feature grids. The computation graph size can be large if automatic differentiation is used for the entire Monte Carlo process, leading to high memory intensity [S3, unofficial].
*   **Performance:** The method suffers from slow convergence and high variance in gradient estimates [S2, unofficial]. It is generally slower than semi-gradient methods which ignore the partial derivatives of the RHS [S2, unofficial].
*   **Strengths:** It provides a global solution to the rendering equation, modeling global illumination effects directly. It is viewpoint-independent during training [S1, unofficial].
*   **Limitations:** High variance and bias in gradients lead to slow training and potential artifacts (darkened renders) [S2, unofficial]. Memory intensive for complex scenes [S3, unofficial].
*   **When to use:** When a global, viewpoint-independent radiance field is required and computational resources allow for slow convergence.
*   **When not to use:** When fast training or low memory usage is critical; consider semi-gradient methods or differentiable path tracing instead.

## Background
Neural Radiosity was introduced by Hadadan et al. in 2021 [S1, unofficial][S3, unofficial]. It is based on the idea of using a neural network to solve the rendering equation by minimizing the residual [S3, unofficial]. The method leverages the radiometric prior to optimize network parameters and model the global radiance field with lightweight neural networks and trainable spatial feature grids [S1, unofficial]. It has been implemented in Mitsuba2 with Python bindings [S3, unofficial].

## Sources
[S1] [2508.07852] Vertex Features for Neural Global Illumination, https://arxiv.org/abs/2508.07852, fetched 2026-10-27 (unofficial)
[S2] Fast and Accurate Neural Rendering Using Semi-Gradients, https://doi.org/10.48550/arXiv.2410.10149, fetched 2026-10-29 (unofficial)
[S3] Differentiable Neural Radiosity, https://arxiv.org/abs/2201.13190, fetched 2026-10-29 (unofficial)
