# Neural Baking with Latent-Space In-Scattering

Also called: Latent-space baking, SH-based in-scattering approximation

## Overview
Neural Baking with Latent-Space In-Scattering is a rendering method that approximates indirect illumination (high-order scattering) by baking the in-scattering operator $\mathcal{K}$ into a compressed latent space representation of the radiance field [S1]. It avoids explicit integration of the Radiative Transfer Equation (RTE) and reduces model complexity compared to direct density-to-radiance mapping [S1]. Unlike methods that rely on complex, irregular sampling strategies for general volume mapping, this approach uses implicit baking with a simpler workload to avoid extensive irregular texel access, achieving significant performance gains [S1].

## How it works
The method approximates the in-scattering radiance $\mathcal{S}$ by decomposing the radiance field into Spherical Harmonics (SH) and applying the in-scattering operator analytically in the latent space, rather than integrating it numerically.

### Inputs and Outputs
*   **Inputs:**
    *   Camera position $p$ and view direction $\omega$.
    *   Volume density/extinction field $\sigma_t(p, \omega) = \sigma_a(p, \omega) + \sigma_s(p, \omega)$ [S1].
    *   Albedo $\alpha(p, \omega) = \sigma_s(p, \omega)/\sigma_t(p, \omega)$ [S1].
    *   Phase function parameters (e.g., Henyey-Greenstein asymmetry parameter $g$) [S1].
    *   Light direction $\omega_s$ (assumed directional; ambient light is handled by random sampling of $\omega_s$) [S1].
*   **Outputs:**
    *   Estimated radiance $\hat{L}(p, \omega)$ at the camera [S1].

### Data Structures
The method relies on multi-resolution 3D grids (Levels of Detail, LoD) and specific feature modules [S1]:

1.  **Latent Radiance Feature Grids $\mathcal{L}_l^m(p)$:** 3D grids indexed by position $p$, storing the latent SH coefficients for degree $l$ and order $m$. These are sampled locally at $p$ [S1].
2.  **Base Grid $\mathcal{B}$:** A 3D grid indexed by position $p$ solely. Used for towards-light sampling to reduce cost [S1].
3.  **Transmittance Map $\mathcal{T}$:** A mipmapped grid indexed by incoming light direction. Used for towards-light sampling [S1].
4.  **View Module $\mathcal{V}$:** A 2D feature grid indexed by view direction $\omega$, used as a condition in the decoding network [S1].
5.  **Phase Module $\mathcal{P}$:** Input module conditioning on phase parameters [S1].
6.  **Albedo Module $\mathcal{A}$:** Input module conditioning on albedo [S1].
7.  **Phase Grid $\mathcal{G}$:** Specifically for Henyey-Greenstein (HG) phase function, indexed by $(g, \cos \theta)$ where $\cos \theta = \omega \cdot \omega_s$ [S1].

### Mathematical Foundation
The Volumetric Radiative Transfer Equation (VRTE) defines the in-scattering radiance $L_s(p, \omega)$ and total radiance $L(p, \omega)$ [S1]:

$$
L_s(p, \omega) = \sigma_a(p, \omega)L_e(p, \omega) + \sigma_s(p, \omega) \int_{S^2} \phi(p, \omega_i, \omega) L(p, \omega_i) d\omega_i
$$

$$
L(p, \omega) = \int_0^\infty T_r(p' \rightarrow p) L_s(p', \omega) dt
$$

where $\phi$ is the phase function and $T_r$ is transmittance [S1]. The in-scattering integral is denoted as an angular operator $\mathcal{K}$ acting on radiance $L$ [S1]:

$$
(\mathcal{K}h)(p, \omega) := \int_{S^2} \phi(p, (\omega_i \cdot \omega)) h(p, \omega_i) d\omega_i
$$

The in-scattering field is $S = \mathcal{K}L$ [S1].

**Theorem 3.1 (SH Diagonalization):** If the radiance field $L(p, \omega)$ has an SH decomposition $L(p, \omega) = \sum_{l=0}^\infty \sum_{m=-l}^l L_l^m(p) Y_l^m(\omega)$, the in-scattering field $S(p, \omega)$ is [S1]:

$$
S(p, \omega) = \sum_{l=0}^\infty \eta_l(p) \sum_{m=-l}^l L_l^m(p) Y_l^m(\omega)
$$

where $\eta_l(p) = \frac{4\pi f_l(p)}{2l+1}$ and $f_l(p)$ are the $l$-th order Legendre polynomial coefficients of the phase function $\phi$ [S1]. For the Henyey-Greenstein phase function with asymmetry parameter $g$, $\eta_l = g^l$ [S1].

**Latent-Space Representation:**
To handle dynamic lighting where only light direction $\omega_s$ varies, the method uses factorized encoding. The latent-space representation of radiance $\tilde{L}$ is [S1]:

$$
\tilde{L}(p, \omega) = \sum_{l=0}^{\infty} \sum_{m=-l}^l \mathcal{L}_l^m(p)Y_l^m(\omega_s)
$$

The latent-space in-scattering field $\tilde{S}$ with low-rank compression (truncation rank $K$) is [S1]:

$$
\tilde{S}(p, \omega) = \sum_{l=0}^K \eta_l(p) \sum_{m=-l}^l \mathcal{L}_l^m(p)Y_l^m(\omega_s)
$$

The network $\mathcal{S}$ decodes $\tilde{S}$ into the final in-scattering radiance, conditioned on view direction $\omega$ (via $\mathcal{V}$), phase (via $\mathcal{P}$), and albedo (via $\mathcal{A}$) [S1].

### Rendering Equation
The final rendered radiance $\hat{L}(p, \omega)$ is computed by accumulating samples along the camera ray [S1]:

$$
\hat{L}(p, \omega) = \frac{1}{N} \sum_{i=1}^N \frac{T_r(p'_i \rightarrow p)}{f(p'_i)} \left( L_{\text{direct}}(p'_i, \omega) + \mathcal{S}(\mathcal{F}(p'_i, \omega)) \right)
$$

where:
*   $N$ is the number of volumetric samples [S1].
*   $p'_i$ is the position of the $i$-th sample, drawn from distribution $f$, starting from camera position $p$ and view direction $\omega$ [S1].
*   $T_r(p'_i \rightarrow p) = e^{-\int_0^d \sigma_t(p, \omega) dt}$ is the transmittance, with $d = \|p'_i - p\|_2$ [S1].
*   $L_{\text{direct}}$ is explicit direct illumination [S1].
*   $\mathcal{F}$ is the multi-scale feature extraction function retrieving features from the baking grids at $p'_i$ [S1].
*   $\mathcal{S}$ is the in-scattering prediction network [S1].

### Algorithm Steps

1.  **Initialize Rendering:** Set camera position $p$ and view direction $\omega$. Define light direction $\omega_s$.
2.  **Sample Camera Ray:** Draw $N$ samples $p'_i$ along the ray from $p$ in direction $\omega$ using distribution $f$ [S1].
3.  **For each sample $p'_i$:**
    a.  **Compute Direct Illumination:** Calculate $L_{\text{direct}}(p'_i, \omega)$ explicitly [S1].
    b.  **Extract Features $\mathcal{F}(p'_i, \omega)$:**
        i.  **Local Sampling:** Query latent radiance feature grids $\mathcal{L}_l^m$ at position $p'_i$ across different LoDs [S1].
        ii. **Towards-Light Sampling:** Query base grid $\mathcal{B}$ and transmittance map $\mathcal{T}$ towards the light source $\omega_s$ from $p'_i$. Use cone tracing with exponentially increasing step size and raising feature grid level [S1].
        iii. **Conditioning Inputs:** Prepare view direction $\omega$ for module $\mathcal{V}$, phase parameters for $\mathcal{P}$, and albedo $\alpha$ for $\mathcal{A}$. If using HG phase, compute $\cos \theta = \omega \cdot \omega_s$ and query phase grid $\mathcal{G}$ indexed by $(g, \cos \theta)$ [S1].
    c.  **Predict In-Scattering:** Compute $\mathcal{S}_{pred} = \mathcal{S}(\mathcal{F}(p'_i, \omega))$ using the neural network [S1].
    d.  **Accumulate:** Add contribution $\frac{T_r(p'_i \rightarrow p)}{f(p'_i)} (L_{\text{direct}}(p'_i, \omega) + \mathcal{S}_{pred})$ to the total radiance [S1].
4.  **Finalize:** Divide the accumulated sum by $N$ to get $\hat{L}(p, \omega)$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def neural_baking_in_scattering(camera_pos, view_dir, light_dir, volume_data, network, num_samples):
    """
    Renders a single pixel using Neural Baking with Latent-Space In-Scattering.
    
    Args:
        camera_pos: 3D vector, camera position p.
        view_dir: 3D vector, view direction omega.
        light_dir: 3D vector, light direction omega_s.
        volume_data: Object providing methods to query sigma_t, sigma_a, sigma_s, alpha, phase_params.
        network: Object providing method predict_in_scattering(features, view_cond, phase_cond, albedo_cond).
        num_samples: Integer N.
        
    Returns:
        float, estimated radiance L_hat.
    """
    total_radiance = 0.0
    
    # 1. Sample along camera ray
    # Note: Distribution f is not specified in sources, assumed standard stochastic sampling
    samples = sample_ray(camera_pos, view_dir, num_samples, volume_data)
    
    for i in range(num_samples):
        p_prime = samples[i]
        
        # 2. Compute Transmittance from sample to camera
        # T_r(p1 -> p2) = exp(-integral_0^d sigma_t(p, omega) dt)
        # Approximated by volume data query or precomputed map
        transmittance = compute_transmittance(p_prime, camera_pos, volume_data)
        
        # 3. Compute Direct Illumination
        # L_direct is explicit, assumed available from volume data or light source model
        l_direct = compute_direct_illumination(p_prime, view_dir, light_dir, volume_data)
        
        # 4. Extract Features for In-Scattering Prediction
        # 4a. Local Sampling: Query latent grids L_l^m at p_prime
        # Note: K is truncation rank, not given in sources, assumed fixed or derived from network config
        latent_features = []
        for l in range(K + 1):
            for m in range(-l, l + 1):
                # Query multi-resolution grid L_l^m at position p_prime
                val = query_grid_L(l, m, p_prime, volume_data)
                latent_features.append(val)
                
        # 4b. Towards-Light Sampling: Query Base Grid B and Transmittance Map T
        # Cone tracing towards light_dir from p_prime
        base_features = cone_trace_base_grid(p_prime, light_dir, volume_data)
        transmittance_features = cone_trace_transmittance_map(p_prime, light_dir, volume_data)
        
        # Combine features
        all_features = latent_features + base_features + transmittance_features
        
        # 4c. Conditioning Inputs
        # View module V: indexed by view_dir
        view_cond = query_grid_V(view_dir, volume_data)
        
        # Albedo module A: indexed by albedo alpha
        alpha = volume_data.get_albedo(p_prime, view_dir)
        albedo_cond = query_grid_A(alpha, volume_data)
        
        # Phase module P and Grid G (if HG)
        # Compute cos_theta = omega . omega_s
        cos_theta = dot(view_dir, light_dir)
        phase_params = volume_data.get_phase_params(p_prime)
        
        if volume_data.is_henyey_greenstein():
            g = phase_params['g']
            # Phase grid G indexed by (g, cos_theta)
            phase_cond = query_grid_G(g, cos_theta, volume_data)
        else:
            phase_cond = query_grid_P(phase_params, volume_data)
            
        # 5. Predict In-Scattering Radiance
        # S predicts the in-scattering contribution S(F)
        s_pred = network.predict_in_scattering(all_features, view_cond, phase_cond, albedo_cond)
        
        # 6. Accumulate
        # Contribution = (T_r / f(p'_i)) * (L_direct + S_pred)
        # f(p'_i) is the sampling distribution density. 
        # If uniform sampling over length D, f = 1/D. Not given in sources, assumed 1.0 for simplicity or normalized by N later.
        # The formula divides by N at the end, implying f is handled or uniform.
        # We assume f(p'_i) is incorporated into the sampling or is 1 for uniform distribution over the ray segment.
        # Standard Monte Carlo: sum( (1/N) * (1/f) * ... ) -> if f=1/N (uniform over N samples), term is 1.
        # The formula has 1/N outside and 1/f inside. If f is PDF of single sample, and we sum N samples, 
        # standard estimator is (1/N) sum( (1/f) * ... ).
        
        contribution = transmittance * (l_direct + s_pred)
        total_radiance += contribution
        
    # 7. Final Radiance
    # The formula has 1/N sum. 
    # If f is the PDF of the distribution from which p'_i is drawn, and we draw N samples,
    # the estimator is (1/N) sum( (1/f(p'_i)) * ... ).
    # If uniform sampling over a domain of size V, f = 1/V.
    # The source does not specify f. We assume f is normalized or handled by the sampling function.
    # For implementation, if samples are uniform, f is constant.
    
    # Note: The source formula is: (1/N) sum( (T_r / f) * (L_direct + S) )
    # If f is not provided, we cannot implement the exact division. 
    # However, in standard volumetric rendering with uniform sampling, f is often absorbed into the step size or N.
    # We will assume f=1 for the purpose of this pseudocode structure, or that f is returned by sample_ray.
    
    # Re-reading source: "p'_i is position sampled from distribution f".
    # We assume sample_ray returns samples and their PDF values if non-uniform.
    
    # Let's assume sample_ray returns (pos, pdf).
    # If uniform, pdf = 1/length.
    
    # Corrected accumulation based on standard Monte Carlo:
    # L = (1/N) sum_{i=1}^N (1/f(p'_i)) * T_r * (L_direct + S)
    
    # Since f is not given, we assume uniform sampling over the ray length D, so f = 1/D.
    # But D is not fixed. 
    # We will leave f as a variable that must be provided by the sampler.
    
    # For the sake of a complete, runnable pseudocode, we assume f is 1.0 (normalized) or handled by the sampler returning weighted samples.
    # The source formula explicitly includes 1/f.
    
    # Let's assume the sampler provides the importance weight w_i = 1/f(p'_i).
    
    # Final calculation
    # We need to sum the weighted contributions.
    
    # Resetting accumulation to include 1/f
    total_weighted_radiance = 0.0
    for i in range(num_samples):
        p_prime = samples[i]
        pdf = samples[i].pdf # Assuming sample object has pdf
        if pdf == 0:
            continue
            
        transmittance = compute_transmittance(p_prime, camera_pos, volume_data)
        l_direct = compute_direct_illumination(p_prime, view_dir, light_dir, volume_data)
        
        # Feature extraction (same as above)
        latent_features = [query_grid_L(l, m, p_prime, volume_data) for l in range(K+1) for m in range(-l, l+1)]
        base_features = cone_trace_base_grid(p_prime, light_dir, volume_data)
        transmittance_features = cone_trace_transmittance_map(p_prime, light_dir, volume_data)
        all_features = latent_features + base_features + transmittance_features
        
        view_cond = query_grid_V(view_dir, volume_data)
        alpha = volume_data.get_albedo(p_prime, view_dir)
        albedo_cond = query_grid_A(alpha, volume_data)
        cos_theta = dot(view_dir, light_dir)
        
        if volume_data.is_henyey_greenstein():
            g = volume_data.get_phase_params(p_prime)['g']
            phase_cond = query_grid_G(g, cos_theta, volume_data)
        else:
            phase_cond = query_grid_P(volume_data.get_phase_params(p_prime), volume_data)
            
        s_pred = network.predict_in_scattering(all_features, view_cond, phase_cond, albedo_cond)
        
        contribution = (transmittance / pdf) * (l_direct + s_pred)
        total_weighted_radiance += contribution
        
    return total_weighted_radiance / num_samples

def sample_ray(origin, direction, num_samples, volume_data):
    """
    Samples points along the ray. Returns list of Sample objects with pos and pdf.
    """
    # Not given in sources. Assume uniform sampling over a bounding box or max distance.
    # We assume a max distance D.
    D = volume_data.get_max_distance()
    samples = []
    for i in range(num_samples):
        t = (i + 0.5) / num_samples * D
        pos = origin + direction * t
        # PDF for uniform sampling over length D is 1/D
        pdf = 1.0 / D
        samples.append(Sample(pos, pdf))
    return samples

def compute_transmittance(p1, p2, volume_data):
    """
    Computes T_r(p1 -> p2) = exp(-integral sigma_t dt).
    """
    # Not given in sources. Assume volume_data provides a transmittance query or integration.
    # For simplicity, assume volume_data has a method to get optical depth or transmittance.
    return volume_data.get_transmittance(p1, p2)

def compute_direct_illumination(p, view_dir, light_dir, volume_data):
    """
    Computes explicit direct illumination.
    """
    # Not given in sources. Standard lighting calculation.
    # L_direct = sigma_a * L_e + sigma_s * integral(phi * L_incoming)
    # But L_direct usually refers to direct light from sources.
    # Assume volume_data provides this or it's a simple function.
    return volume_data.get_direct_light(p, view_dir, light_dir)

def query_grid_L(l, m, pos, volume_data):
    """
    Queries the latent radiance feature grid L_l^m at position pos.
    """
    # Not given in sources. Assume trilinear interpolation in a 3D grid.
    return volume_data.grid_L[l][m].trilinear_query(pos)

def cone_trace_base_grid(pos, light_dir, volume_data):
    """
    Cone traces the base grid B towards light_dir.
    """
    # Not given in sources. Assume exponential step size.
    # Returns aggregated features.
    features = []
    current_pos = pos
    step = 0.1 # Initial step size
    max_steps = 10
    for _ in range(max_steps):
        val = volume_data.grid_B.trilinear_query(current_pos)
        features.append(val)
        current_pos = current_pos + light_dir * step
        step *= 2 # Exponential increase
    return features

def cone_trace_transmittance_map(pos, light_dir, volume_data):
    """
    Cone traces the transmittance map T towards light_dir.
    """
    # Not given in sources.
    features = []
    current_pos = pos
    step = 0.1
    max_steps = 10
    for _ in range(max_steps):
        # T is indexed by direction, but cone trace is along direction.
        # Source says T is indexed by incoming light direction.
        # This implies T might be a function of direction, not position?
        # "mipmapped transmittance map T with respect to incoming light direction"
        # This is ambiguous. Usually transmittance is spatial.
        # Assuming T is a 3D grid indexed by position, but optimized for light direction queries?
        # Or T is a 2D grid indexed by direction?
        # Source: "indexed by incoming light direction".
        # If T is indexed by direction, how is it cone traced?
        # Perhaps T is a precomputed transmittance for specific directions?
        # We assume T is queried based on the current cone trace position and direction.
        # Given the ambiguity, we assume T is a 3D grid like B.
        val = volume_data.grid_T.trilinear_query(current_pos)
        features.append(val)
        current_pos = current_pos + light_dir * step
        step *= 2
    return features

def query_grid_V(view_dir, volume_data):
    """
    Queries view module V indexed by view_dir.
    """
    # Not given in sources. Assume 2D grid lookup.
    return volume_data.grid_V.lookup(view_dir)

def query_grid_A(alpha, volume_data):
    """
    Queries albedo module A indexed by alpha.
    """
    # Not given in sources.
    return volume_data.grid_A.lookup(alpha)

def query_grid_G(g, cos_theta, volume_data):
    """
    Queries phase grid G indexed by (g, cos_theta).
    """
    # Not given in sources.
    return volume_data.grid_G.lookup(g, cos_theta)

def query_grid_P(phase_params, volume_data):
    """
    Queries phase module P.
    """
    # Not given in sources.
    return volume_data.grid_P.lookup(phase_params)

class Sample:
    def __init__(self, pos, pdf):
        self.pos = pos
        self.pdf = pdf
```

### Parameters
*   $N$: Number of volumetric samples. Typical values not given in sources.
*   $K$: Truncation rank in SH. Typical values not given in sources.
*   $\sigma_t(p, \omega)$: Extinction coefficient.
*   $\alpha(p, \omega)$: Albedo.
*   $\eta_l(p)$: Diagonal scaling factor.
*   $f_l(p)$: Legendre polynomial coefficients of the phase function.

### Edge Cases and Pitfalls
*   **Feature Extraction Cost:** Naively sampling all $(K+1)^2$ components of the SH grid $\mathcal{L}_l^m$ both locally and towards light amplifies feature extraction cost. This is mitigated by using only $\mathcal{B}$ and $\mathcal{T}$ for towards-light sampling [S1].
*   **Per-Volume Tuning:** Unlike MRPNN which operates as a general mapping, this method requires per-volume tuning to find a suitable solution [S1].
*   **Light Direction Assumption:** The method assumes a directional light. For ambient light, light direction must be drawn randomly [S1].

## Operating characteristics
*   **Time Complexity:** Avoids the $O(N^5)$ space complexity associated with explicit integration before factorization [S1].
*   **Memory Complexity:** Reduced compared to explicit integration due to latent-space compression [S1].
*   **Performance:** Achieves a 10% performance gain over MRPNN by avoiding extensive irregular texel access [S1].
*   **Strengths:** Accelerates volumetric rendering by approximating high-order scattering via implicit baking [S1].
*   **Limitations:** Requires per-volume tuning [S1]. Assumes directional light (ambient requires random sampling) [S1].

## Background
*   **Origin:** The method is described in "Multi-feature Radiance Baking Neural Networks for Instant Volumetric Rendering" [S1].
*   **History:** It builds on the RPNN series and addresses problems of explicit integration and irregular texture lookups found in methods like MRPNN [S1].
*   **Usage:** Used for instant volumetric rendering [S1].

## Sources
[S1] Multi-feature Radiance Baking Neural Networks for Instant Volumetric Rendering, https://doi.org/10.1145/3799902.3811179, fetched 2026-10-31
