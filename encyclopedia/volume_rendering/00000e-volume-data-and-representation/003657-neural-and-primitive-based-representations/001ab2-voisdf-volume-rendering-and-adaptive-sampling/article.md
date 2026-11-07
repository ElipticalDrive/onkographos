# VoISDF Volume Rendering and Adaptive Sampling

Also called: VolSDF, VolSDF Training, Volume Rendering of Neural Implicit Surfaces, Volume Rendering of Neural Implicit Surfaces training, Volume SDF

## Overview
VoISDF is a neural volume rendering method that models volume density as a function of a Signed Distance Function (SDF) to improve geometry reconstruction and enable adaptive sampling [S3, unofficial]. Unlike prior methods that extract geometry from an arbitrary level set of a generic density MLP, VoISDF extracts the surface as the zero level-set of the learned SDF, providing a well-defined surface and inductive bias [S3, unofficial]. It avoids the salt-and-pepper artifacts of random sampling and the density mismatch of second-network inverse sampling by using a theoretical error bound for opacity approximation to drive adaptive sampling [S3, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of images with camera parameters. For each pixel $p$, the input is a triplet $(I_p, \mathbf{c}_p, \mathbf{v}_p)$, where $I_p \in \mathbb{R}^3$ is the intensity, $\mathbf{c}_p \in \mathbb{R}^3$ is the camera location, and $\mathbf{v}_p \in \mathbb{R}^3$ is the viewing direction [S3, unofficial].
*   **Outputs:** Rendered image intensity $I(\mathbf{c}, \mathbf{v})$ and a reconstructed 3D surface defined by the zero level-set of the SDF [S3, unofficial].

**Data Structures**
The system consists of two Multi-Layer Perceptrons (MLPs) and two scalar learnable parameters [S3, unofficial]:
1.  **Geometry Network ($f_\varphi$):** Approximates the SDF $d(\mathbf{x})$ and a global geometry feature $\mathbf{z} \in \mathbb{R}^{256}$. Output: $(d(\mathbf{x}), \mathbf{z}(\mathbf{x})) \in \mathbb{R}^{1+256}$.
2.  **Radiance Network ($L_\psi$):** Predicts color. Input: position $\mathbf{x}$, normal $\mathbf{n}$, view direction $\mathbf{v}$, and geometry feature $\mathbf{z}$. Output: color $\in \mathbb{R}^3$.
3.  **Parameters:** $\alpha, \beta \in \mathbb{R}$ (scalar learnable parameters). In implementation, $\alpha = \beta^{-1}$ is often chosen [S3, unofficial].
4.  **Positional Encoding:** Exploited for position $\mathbf{x}$ and view direction $\mathbf{v}$ in both networks [S3, unofficial].

**Formulas**
*   **SDF Definition:** $d_\Omega(\mathbf{x}) = (-1)^{\mathbf{1}_\Omega(\mathbf{x})} \min_{\mathbf{y} \in \mathcal{M}} \|\mathbf{x} - \mathbf{y}\|$, where $\mathbf{1}_\Omega$ is the indicator function for object space $\Omega$ and $\mathcal{M}$ is its boundary [S3, unofficial].
*   **Density from SDF:** $\sigma(\mathbf{x}) = \alpha \Psi_\beta(-d_\Omega(\mathbf{x}))$, where $\Psi_\beta$ is the CDF of the Laplace distribution [S3, unofficial]:
    $$
    \Psi_\beta(s) = \begin{cases} \frac{1}{2} \exp\left(-\frac{s}{\beta}\right) & \text{if } s \leq 0 \\ 1 - \frac{1}{2} \exp\left(-\frac{s}{\beta}\right) & \text{if } s > 0 \end{cases}
    $$
*   **Transmittance and Opacity:** $T(t) = \exp\left(-\int_0^t \sigma(\mathbf{x}(s)) ds\right)$ and $O(t) = 1 - T(t)$ [S3, unofficial].
*   **Rendering Equation:** $I(\mathbf{c}, \mathbf{v}) = \int_0^\infty L(\mathbf{x}(t), \mathbf{n}(t), \mathbf{v}) \tau(t) dt$, where $\tau(t) = \sigma(\mathbf{x}(t))T(t)$ is the PDF of opacity and $\mathbf{n}(t) = \nabla_{\mathbf{x}} d_\Omega(\mathbf{x}(t))$ [S3, unofficial].
*   **Rectangle Rule Approximation:** For samples $\mathcal{T} = \{t_i\}_{i=1}^n$, the integral of density is approximated as $\hat{R}(t) = \sum_{i=1}^{k-1} \delta_i \sigma_i + (t - t_k) \sigma_k$ for $t \in [t_k, t_{k+1}]$, where $\delta_i = t_{i+1} - t_i$ and $\sigma_i = \sigma(\mathbf{x}(t_i))$ [S3, unofficial].
*   **Approximated Opacity:** $\widehat{O}(t) = 1 - \exp\left(-\widehat{R}(t)\right)$ [S3, unofficial].
*   **Error Bound:** The bound on the derivative of density allows bounding the rectangle rule error $E(t)$:
    $$
    |E(t)| \leq \widehat{E}(t) = \frac{\alpha}{4\beta} \left( \sum_{i=1}^{k-1} \delta_i^2 e^{-\frac{d_i^*}{\beta}} + (t - t_k)^2 e^{-\frac{d_k^*}{\beta}} \right)
    $$
    where $d_i^* = \min_{s \in [t_i, t_{i+1}], \mathbf{y} \notin B_i \cup B_{i+1}} \|\mathbf{x}(s) - \mathbf{y}\|$ and $B_i = \{\mathbf{x} \mid \|\mathbf{x} - \mathbf{x}(t_i)\| < |d_i|\}$ [S3, unofficial].
*   **Opacity Error Bound:** $|O(t) - \widehat{O}(t)| \leq \exp\left(-\widehat{R}(t)\right) \left( \exp\left(\widehat{E}(t)\right) - 1 \right)$ [S3, unofficial].
*   **Global Error Bound:** $B_{\mathcal{T}, \beta} = \max_{k \in [n-1]} \left\{ \exp\left(-\widehat{R}(t_k)\right) \left( \exp(\widehat{E}(t_{k+1})) - 1 \right) \right\}$ [S3, unofficial].

**Parameters**
*   $\beta$: Scale parameter of the Laplace distribution. Typical value: 0.001 [S3, unofficial]. High $\beta$ makes training harder but potentially more accurate; low $\beta$ makes training easier but less accurate [S1, unofficial].
*   $\epsilon$: Error threshold for the opacity approximation bound. Typical value: 0.1 [S3, unofficial].
*   $n$: Number of initial uniform samples. Default: 128 [S3, unofficial].
*   $m$: Number of final samples for rendering via inverse CDF. Default: 64 [S3, unofficial].
*   $M$: Large constant representing the maximum ray length [S3, unofficial].

**Algorithm: Adaptive Sampling**
The following pseudocode implements the adaptive sampling algorithm described in [S3, unofficial]. It ensures the opacity approximation error stays within $\epsilon$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sampling(ray_origin, ray_direction, geometry_network, density_params, epsilon, max_iter=5):
    """
    Performs adaptive sampling along a ray based on the VoISDF error bound.
    
    Args:
        ray_origin: Camera position c.
        ray_direction: Viewing direction v.
        geometry_network: MLP f_phi predicting (d, z).
        density_params: Tuple (alpha, beta).
        epsilon: Target error bound.
        max_iter: Maximum number of upsampling iterations.
        
    Returns:
        samples: Set of m samples S for rendering.
    """
    # 1. Initialize uniform sampling T_0 with n=128 points
    n = 128
    M = 10.0  # Large constant for max ray length, typical value
    t = [(k - 1) * M / (n - 1) for k in range(1, n + 1)]
    T = t
    
    # 2. Initialize beta_plus > beta such that B_{T, beta_plus} <= epsilon
    # Using Lemma 2: beta >= alpha * M^2 / (4 * (n-1) * log(1 + epsilon))
    alpha, beta = density_params
    beta_plus = max(beta, alpha * M**2 / (4 * (n - 1) * np.log(1 + epsilon)))
    
    # Helper to compute global error bound B_{T, beta}
    def compute_global_error_bound(samples_T, beta_val):
        # Compute density sigma at each sample point
        sigma_vals = []
        for t_i in samples_T:
            x = ray_origin + t_i * ray_direction
            d, _ = geometry_network(x)
            # sigma(x) = alpha * Psi_beta(-d)
            s = -d
            if s <= 0:
                sigma = alpha * 0.5 * np.exp(-s / beta_val)
            else:
                sigma = alpha * (1 - 0.5 * np.exp(-s / beta_val))
            sigma_vals.append(sigma)
            
        # Compute R_hat and E_hat for each interval
        # R_hat(t_k) is cumulative sum of delta_i * sigma_i
        # E_hat(t_{k+1}) involves sum of delta_i^2 * exp(-d_i*/beta)
        
        # Note: d_i* requires finding min distance to surface within interval.
        # For implementation simplicity, we approximate d_i* using the SDF values at endpoints
        # or a finer sub-sampling if exact calculation is too costly.
        # The source defines d_i* precisely but defers the formula to supplementary.
        # We assume a helper function get_d_star(t_i, t_{i+1}, ray_origin, ray_direction, geometry_network) exists.
        
        max_bound = 0.0
        R_hat_prev = 0.0
        
        for k in range(len(samples_T) - 1):
            t_k = samples_T[k]
            t_k1 = samples_T[k+1]
            delta = t_k1 - t_k
            sigma_k = sigma_vals[k]
            
            # R_hat(t_k) is accumulated before this interval
            # R_hat(t_{k+1}) = R_hat(t_k) + delta * sigma_k
            R_hat_curr = R_hat_prev + delta * sigma_k
            
            # Compute E_hat(t_{k+1})
            # E_hat(t) = (alpha / (4*beta)) * ( sum_{i=1}^{k-1} delta_i^2 exp(-d_i*/beta) + (t-t_k)^2 exp(-d_k*/beta) )
            # We need to maintain the sum term.
            # For simplicity, we recompute or maintain running sum.
            # Let's assume we have a function to compute the error bound term for the interval end.
            
            # Approximation: The source gives the bound formula. 
            # We need d_i* for each interval i <= k.
            # d_i* = min distance to surface in [t_i, t_{i+1}] excluding neighborhoods of endpoints.
            
            # To keep pseudocode self-contained and valid, we define a placeholder for the complex d_i* calculation
            # which is deferred in the source. In practice, this requires ray marching or optimization.
            # We will use the SDF values at endpoints as a proxy if exact d_i* is not available, 
            # but strictly following the source, we must compute d_i*.
            
            # Let's assume we have a function `compute_error_bound_term` that returns the contribution 
            # of interval i to the E_hat sum.
            
            # Re-evaluating the bound formula structure:
            # E_hat(t_{k+1}) = (alpha / (4*beta)) * ( Sum_{i=1}^{k} delta_i^2 * exp(-d_i*/beta) )
            # Note: The source formula (12) has sum up to k-1 and a term for (t-t_k). 
            # For t_{k+1}, the last term is delta_k^2 * exp(-d_k*/beta).
            
            # We need to track the sum of delta_i^2 * exp(-d_i*/beta) for all i <= k.
            
            # For the purpose of this pseudocode, we assume `compute_interval_error_contribution` 
            # calculates delta_i^2 * exp(-d_i*/beta).
            
            # Calculate R_hat(t_k) for the bound calculation
            # R_hat(t_k) is the integral approximation up to t_k.
            
            # Calculate the bound for interval [t_k, t_{k+1}]
            # Bound_k = exp(-R_hat(t_k)) * (exp(E_hat(t_{k+1})) - 1)
            
            # We need to compute E_hat(t_{k+1}).
            # Let S_k = sum_{i=1}^{k} delta_i^2 * exp(-d_i*/beta)
            # E_hat(t_{k+1}) = (alpha / (4*beta)) * S_k
            
            # We need to compute S_k.
            
            # This part is computationally heavy. We assume efficient implementation.
            
            # For the sake of the algorithm structure, we compute the bound.
            
            # 1. Compute R_hat(t_k)
            # R_hat(t_k) = sum_{j=1}^{k-1} delta_j * sigma_j
            
            # 2. Compute E_hat(t_{k+1})
            # We need d_i* for all i <= k.
            
            # Let's simplify: The source says "details are in the supplementary".
            # We will implement the logic assuming we can compute the bound.
            
            # Compute R_hat(t_k)
            R_hat_tk = sum([samples_T[j+1] - samples_T[j] for j in range(k)]) # This is wrong, need sigma
            
            # Correct R_hat(t_k)
            R_hat_tk = 0.0
            for j in range(k):
                R_hat_tk += (samples_T[j+1] - samples_T[j]) * sigma_vals[j]
                
            # Compute E_hat(t_{k+1})
            # We need the sum of delta_i^2 * exp(-d_i*/beta) for i=1 to k.
            # We assume a function `get_d_star` is available.
            
            sum_term = 0.0
            for i in range(k):
                t_i = samples_T[i]
                t_i1 = samples_T[i+1]
                delta_i = t_i1 - t_i
                # d_i* calculation
                d_star = get_d_star(t_i, t_i1, ray_origin, ray_direction, geometry_network)
                sum_term += delta_i**2 * np.exp(-d_star / beta_val)
                
            E_hat_tk1 = (alpha / (4 * beta_val)) * sum_term
            
            # Bound for this interval
            bound_k = np.exp(-R_hat_tk) * (np.exp(E_hat_tk1) - 1)
            
            if bound_k > max_bound:
                max_bound = bound_k
                
            R_hat_prev = R_hat_tk + delta * sigma_k # R_hat(t_{k+1})
            
        return max_bound

    # Initial check
    B_T_beta = compute_global_error_bound(T, beta)
    
    # 3. While B_{T, beta} > epsilon and not max_iter
    iteration = 0
    while B_T_beta > epsilon and iteration < max_iter:
        # 4. Upsample T
        # Add n points, proportional to current error bound
        # We need to compute error bound for each interval to determine sampling density
        interval_bounds = []
        for k in range(len(T) - 1):
            t_k = T[k]
            t_k1 = T[k+1]
            # Compute R_hat(t_k) and E_hat(t_{k+1}) for this interval
            # Similar to above, but for a single interval
            # R_hat(t_k)
            R_hat_tk = 0.0
            for j in range(k):
                R_hat_tk += (T[j+1] - T[j]) * sigma_vals[j] # Need to recompute sigma_vals for current T
            
            # Recompute sigma_vals for current T
            sigma_vals = []
            for t_i in T:
                x = ray_origin + t_i * ray_direction
                d, _ = geometry_network(x)
                s = -d
                if s <= 0:
                    sigma = alpha * 0.5 * np.exp(-s / beta_plus)
                else:
                    sigma = alpha * (1 - 0.5 * np.exp(-s / beta_plus))
                sigma_vals.append(sigma)
                
            # Recompute R_hat(t_k) with new sigma_vals
            R_hat_tk = 0.0
            for j in range(k):
                R_hat_tk += (T[j+1] - T[j]) * sigma_vals[j]
                
            # Compute E_hat(t_{k+1})
            sum_term = 0.0
            for i in range(k):
                t_i = T[i]
                t_i1 = T[i+1]
                delta_i = t_i1 - t_i
                d_star = get_d_star(t_i, t_i1, ray_origin, ray_direction, geometry_network)
                sum_term += delta_i**2 * np.exp(-d_star / beta_plus)
            E_hat_tk1 = (alpha / (4 * beta_plus)) * sum_term
            
            bound_k = np.exp(-R_hat_tk) * (np.exp(E_hat_tk1) - 1)
            interval_bounds.append(bound_k)
            
        # Normalize bounds to get sampling weights
        total_bound = sum(interval_bounds)
        if total_bound == 0:
            weights = [1.0 / len(interval_bounds)] * len(interval_bounds)
        else:
            weights = [b / total_bound for b in interval_bounds]
            
        # Add n points proportional to weights
        new_points = []
        for k, weight in enumerate(weights):
            num_points = int(round(weight * n))
            if num_points < 1:
                num_points = 1
            t_k = T[k]
            t_k1 = T[k+1]
            for p in range(num_points):
                # Insert points between t_k and t_k1
                # Uniformly distributed within the interval
                new_t = t_k + (p + 1) * (t_k1 - t_k) / (num_points + 1)
                new_points.append(new_t)
                
        T = sorted(T + new_points)
        
        # 5. If B_{T, beta_plus} < epsilon
        B_T_beta_plus = compute_global_error_bound(T, beta_plus)
        if B_T_beta_plus < epsilon:
            # 6. Find beta_star in (beta, beta_plus) such that B_{T, beta_star} = epsilon
            # Using bisection
            low = beta
            high = beta_plus
            for _ in range(10):
                mid = (low + high) / 2
                B_mid = compute_global_error_bound(T, mid)
                if B_mid > epsilon:
                    low = mid
                else:
                    high = mid
            beta_star = (low + high) / 2
            beta_plus = beta_star
            
        iteration += 1
        B_T_beta = compute_global_error_bound(T, beta)
        
    # 10. Estimate O_hat using final T and beta_plus
    # Recompute sigma_vals with beta_plus
    sigma_vals = []
    for t_i in T:
        x = ray_origin + t_i * ray_direction
        d, _ = geometry_network(x)
        s = -d
        if s <= 0:
            sigma = alpha * 0.5 * np.exp(-s / beta_plus)
        else:
            sigma = alpha * (1 - 0.5 * np.exp(-s / beta_plus))
        sigma_vals.append(sigma)
        
    # Compute R_hat for all t in T
    R_hat_vals = [0.0]
    for k in range(len(T) - 1):
        delta = T[k+1] - T[k]
        R_hat_vals.append(R_hat_vals[-1] + delta * sigma_vals[k])
        
    # O_hat(t) = 1 - exp(-R_hat(t))
    O_hat_vals = [1 - np.exp(-R) for R in R_hat_vals]
    
    # 11. Return m=64 fresh samples using inverse transform sampling with O_hat^{-1}
    # Generate m uniform random numbers
    u = np.random.uniform(0, 1, m)
    samples = []
    for u_val in u:
        # Find t such that O_hat(t) = u_val
        # Since O_hat is monotonic, we can use binary search or interpolation
        # Find index k such that O_hat_vals[k] <= u_val < O_hat_vals[k+1]
        k = np.searchsorted(O_hat_vals, u_val, side='right') - 1
        if k < 0:
            k = 0
        if k >= len(O_hat_vals) - 1:
            k = len(O_hat_vals) - 2
            
        t_k = T[k]
        t_k1 = T[k+1]
        O_k = O_hat_vals[k]
        O_k1 = O_hat_vals[k+1]
        
        # Linear interpolation
        if O_k1 == O_k:
            t_sample = t_k
        else:
            t_sample = t_k + (t_k1 - t_k) * (u_val - O_k) / (O_k1 - O_k)
            
        samples.append(t_sample)
        
    return samples

def get_d_star(t_i, t_i1, ray_origin, ray_direction, geometry_network):
    """
    Computes d_i* = min distance to surface in [t_i, t_{i+1}] excluding neighborhoods of endpoints.
    This is a placeholder for the complex calculation deferred in the source.
    In practice, this might involve ray marching or optimization.
    """
    # Simplified approximation: use the minimum SDF magnitude in the interval
    # excluding small neighborhoods of endpoints.
    num_samples = 10
    min_d = float('inf')
    for j in range(1, num_samples - 1):
        t = t_i + (j / num_samples) * (t_i1 - t_i)
        x = ray_origin + t * ray_direction
        d, _ = geometry_network(x)
        if abs(d) < min_d:
            min_d = abs(d)
    return min_d
```

**Edge Cases and Pitfalls**
*   **Random Sampling Artifacts:** Random sampling in NeRF causes salt-and-pepper artifacts [S3, unofficial].
*   **Second Network Mismatch:** Using a second network for inverse sampling may result in density mismatch between the opacity network and the density network used for integration [S3, unofficial].
*   **Beta Bias:** High $\beta$ biases the learned SDF differently than low $\beta$. Low $\beta$ makes it easier to train the SDF with volume rendering but potentially less accurate surface learning, while high $\beta$ makes it harder to train but potentially more accurate [S1, unofficial].
*   **Geometry vs Density Ranges:** Density and distance have different possible ranges, so the MLP for predicting distance may need different configuration than the one used for predicting density in previous parts [S2].

## Operating characteristics
*   **Time Complexity:** The adaptive sampling algorithm iteratively upsamples the ray, with each iteration requiring computation of error bounds for all intervals. The complexity depends on the number of samples $n$ and the number of iterations. The final rendering uses $m=64$ samples [S3, unofficial].
*   **Memory Complexity:** Requires storing the MLP parameters and the sample points along the ray. The global geometry feature $\mathbf{z}$ is 256-dimensional [S3, unofficial].
*   **Performance:** On the BlendedMVS dataset, VolSDF improves Chamfer distance by a mean of 51.8% compared to NeRF++. VolSDF's mean PSNR is 27.08, compared to NeRF++'s 27.55 [S3, unofficial].
*   **Strengths:** Disentangles geometry and appearance; avoids artifacts from random or uniform sampling; does not require object masks for training (unlike IDR) [S3, unofficial].
*   **Limitations:** Training with high $\beta$ can be difficult; the error bound calculation is computationally intensive [S1, S3, unofficial].

## Background
VoISDF was introduced in the paper "Volume Rendering of Neural Implicit Surfaces" [S3, unofficial]. It builds on the idea of modeling density as a function of SDF, contrasting with previous work that modeled geometry as a function of density. It addresses the limitations of NeRF (artifacts, lack of explicit geometry) and IDR (need for masks, local minima) [S3, unofficial].

## Sources
[S1] omkarchittar/Neural_Surfaces, https://github.com/omkarchittar/Neural_Surfaces, fetched 2026-10-26 (unofficial)
[S2] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S3] Volume Rendering of Neural Implicit Surfaces, https://proceedings.neurips.cc/paper_files/paper/2021/file/25e2a30f44898b9f3e978b1786dcd85c-Paper.pdf, fetched 2026-10-26 (unofficial)
