# Real-scattering vertex connection with ratio tracking

Also called: Delta Tracking for Transmittance Estimation, Ratio Tracking Volumetric Sensor, Ratio Tracking with Null Scattering, RatioVolumetricSensor, Real-scattering vertex connection

## Overview
This method estimates the transmittance between two points $p_1$ and $2$ in a participating medium using ratio tracking, which accumulates a product of null-scattering ratios rather than terminating on collision [S2, S3, unofficial]. It is used to connect real-scattering vertices on camera and light subpaths in a null-scattering path integral framework [S3, unofficial]. Unlike delta tracking for sampling which terminates on collision, ratio tracking continues until the endpoint is reached, accumulating a product of ratios to estimate the total transmittance [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   $p_1, p_2$: Start and end points of the segment.
*   `sampler`: A random number generator providing uniform samples in $[0, 1)$.
*   `majorant`: The homogenized extinction coefficient $\hat{\mu}_t$ (stored as `self.majorant`) [S1, S3, unofficial].
*   `throughput`: The current path throughput, initialized by undoing transmittance from previous interactions [S1, unofficial].
*   `attenuation`: The accumulated transmittance from previous segments [S1, unofficial].
*   `t_prev`: The distance traveled in the previous segment [S1, unofficial].
*   `n_scat`: The number of scattering events in the history [S1, unofficial].
*   `sigma_s`: The scattering coefficient [S1, unofficial].
*   `sigma_a`: The absorption coefficient (derived from `sigma_t` and `sigma_s`) [S1, unofficial].
*   `sigma_t`: The total extinction coefficient at the current position [S1, unofficial].
*   `em`: The emission coefficient [S1, unofficial].
*   `sa`: The absorption coefficient ($\sigma_a = \sigma_t - \sigma_s$) [S1, unofficial].
*   `st`: The total extinction coefficient $\sigma_t$ [S1, unofficial].
*   `weight`: The initial weight for the contribution, calculated as `throughput * exp(-st * t_prev)` [S1, unofficial].
*   `bbox`: The bounding box of the medium grid [S1, unofficial].
*   `voxel_size`: The size of each voxel in the grid [S1, unofficial].
*   `m_film`: The volumetric film to write contributions to [S1, unofficial].
*   `δL.array`: The differentiable array for gradient storage [S1, unofficial].

**Outputs:**
*   `transmittance`: The estimated transmittance product [S1, S2, unofficial].
*   `contrib`: The contribution written to the film [S1, unofficial].

### Data Structures
*   **VolumetricFilm**: A tensor of shape `(resz, resy, resx, channels)` where `channels` is 2 if `surface_aware` is true, else 1 [S1, unofficial].
*   **Container**: Stores medium properties including `sigma_t`, `albedo`, and `medium_phase` [S1, unofficial].

### Steps
1.  **Initialize Throughput and Scattering History**:
    Undo transmittance from previous interactions and adjust for scattering history [S1, unofficial]:
    ```python
    throughput = dr.detach(attenuation * dr.exp(mei.sigma_t * t_prev))
    throughput *= dr.select(mei.sigma_s != 0, dr.rcp(dr.detach(mei.sigma_s) ** n_scat), 1.)
    ```

2.  **Setup Segment**:
    Calculate the distance to the endpoint: `distToEnd = length(p1 - p2)` [S1, S2, unofficial].
    Initialize `t = 0` and `transmittance = (1, 1, 1)` [S1, S2, unofficial].
    Create a ray from $p_1$ to $p_2$ [S1, unofficial].

3.  **Ratio Tracking Loop**:
    While the ray is active:
    a.  **Sample Distance**:
        Sample a collision-free distance $s$ using the majorant [S1, S2, unofficial]:
        ```python
        # Using uniform sample u in [0, 1)
        u = sampler.next_1d(active)
        # Ensure u is not exactly 1 to avoid log(0)
        u = dr.clamp(u, 0.0, 0.9999999) 
        sampled_t = - dr.log(1.0 - u) / self.majorant
        ```
        Alternatively, using `invMajorant` [S2, unofficial]:
        ```python
        s = - dr.log(dr.max(1.0 - sampler.getNext1D(), 0.0)) * invMajorant
        sampled_t = s
        ```
    b.  **Update Position**:
        Update the ray position $t$ [S1, S2, unofficial]:
        ```python
        t[active] += sampled_t
        ```
    c.  **Check Bounds**:
        Check if the ray is still within the maximum distance `maxt` (or `distToEnd`) [S1, S2, unofficial]:
        ```python
        active &= t < maxt # or t < distToEnd
        ```
        If `t > distToEnd`, break the loop and return `transmittance` [S2, unofficial].

    d.  **Voxel Lookup**:
        Determine the current voxel index [S1, unofficial]:
        ```python
        current_voxel = mi.Vector3i(dr.floor((p - self.bbox.min) / self.voxel_size))
        is_inside_grid = dr.all(current_voxel >= 0) & dr.all(current_voxel < self.res)
        idx = compute_flat_index(current_voxel, self.res, self.surface_aware)
        ```
        Compute the local density/extinction coefficients at `ray(t)` [S2, unofficial]. Let $\sigma_t$ be the total extinction and $\sigma_s$ be the scattering coefficient at the current position.

    e.  **Update Transmittance**:
        Compute the null-scattering ratio and update the transmittance product [S2, unofficial]:
        ```python
        sigma_n = self.majorant - sigma_a - sigma_s
        # Ratio tracking accumulates the ratio of null-scattering probability
        # to the majorant sampling probability.
        # The term (sigma_n / majorant) represents the probability of a null event.
        transmittance *= (sigma_n * invMajorant)
        ```
        Note: Some implementations accumulate the product of ratios directly as the estimator for transmittance [S2, unofficial].

    f.  **Compute Contribution (for Sensor/Connection)**:
        If performing a connection (e.g., Next-Event Estimation), compute the contribution [S1, unofficial]:
        ```python
        # sa = sigma_t - sigma_s
        # st = sigma_t
        # weight = throughput * exp(-st * t_prev)
        contrib = dr.select(
            active & is_inside_grid,
            weight * em * sa / st * (1 - st / self.majorant) ** n_interactions * st / self.majorant,
            0.
        )
        ```
        Here, $n_{interactions}$ is the number of steps taken in the ratio tracking loop [S1, unofficial].

    g.  **Write to Film**:
        In primal mode, write the contribution to the film [S1, unofficial]:
        ```python
        self.m_film.write(contrib, idx, active & is_inside_grid)
        ```

4.  **Automatic Differentiation**:
    *   **Forward Mode**: Scatter the gradient [S1, unofficial]:
        ```python
        dr.scatter_reduce(dr.ReduceOp.Add, δL.array, dr.forward_to(contrib), idx, active & is_inside_grid)
        ```
    *   **Reverse Mode**: Gather the gradient and backpropagate [S1, unofficial]:
        ```python
        grad = dr.gather(mi.Spectrum, δL.array, idx, active & is_inside_grid)
        dr.backward_from(contrib * grad)
        ```

### Formulas
The path PDF for connecting real-scattering vertices on camera and light subpaths with $s$ and $t$ vertices respectively is [S3, unofficial]:
$$
p_{s,t} (x) = p_{dt} (x_{r_0} ...x_{r_{t-1}}) p_{rt}(x_{r_{t-1}}, x_{r_{r-s+1}}) p_{dt}(x_{r_r} ...x_{r_{r-s+1}})
$$
where $p_{dt}(x)$ is the delta-tracking path PDF [S3, unofficial]:
$$
p_{dt}(x) = p(x_0) \prod_{i=1}^{k} p(\omega_i |x_{i-1}, \omega_{i-1}) p(x_i |x_{i-1}, \omega_i) P(e_{x_i}) \prod_{i=1}^{k-1} G(x_{r_i}, x_{r_i+1})
$$
The transition probabilities are [S3, unofficial]:
$$
p(\omega' |x, \omega) \propto \rho(\omega, x, \omega'), \quad p(y|x, \omega) = T(x, y)\mu_t(y), \quad P(e_x) = \frac{\mu_{e_x}(x)}{\mu_t(x)}
$$
where $e_x \in \{a, s, n\}$ denotes the event type (absorption, scattering, or null-scattering) [S3, unofficial].

For the full path PDF including the connection segment [S3, unofficial]:
$$
p(x) = p_{dt}(x_0 ...x_{r_{r-1}}) p(\omega_r |x_{r_{r-1}}, \omega_{r_{r-1}}) p(x_k) p_{ea}(x_{r_{r-1}}|x_{r_{r-2}}, \omega_{r_{r-1}}, x_k) \cdot p_{rt}(x_{r_{r-2}}, x_{r_{r-1}}) p_{rt}(x_{r_{r-1}}, x_k) G(x_{r_{r-2}}, x_{r_{r-1}})
$$
The equiangular sampling PDF for the connection is [S3, unofficial]:
$$
p_{ea}(x_{r_{r-1}}|x_{r_{r-2}}, \omega_{r_{r-1}}, x_k) \propto G(x_{r_{r-1}}, x_k) = \frac{1}{\|x_{r_{r-1}} - x_k\|^2}
$$

### Parameters
*   `majorant`: The majorant density $\hat{\mu}_t$ used for ratio tracking sampling [S1, S3, unofficial].
*   `s`: Number of vertices on the camera subpath (real-scattering) [S3, unofficial].
*   `t`: Number of vertices on the light subpath (real-scattering) [S3, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: Logarithms of values close to 1 or 0 can cause precision issues. Clamp sampler outputs away from 1.0 [S2, unofficial].
*   **Boundaries**: Ensure voxel indices are within grid bounds before accessing data [S1, unofficial].
*   **Degenerate Input**: If `sigma_s` is 0, avoid division by zero in throughput initialization [S1, unofficial].
*   **Termination**: Ratio tracking must continue until the endpoint is reached, unlike standard delta tracking which stops at the first collision [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Proportional to the number of ratio tracking steps, which depends on the ratio of the majorant to the local extinction coefficient. Higher majorants lead to more steps but lower variance per step.
*   **Memory Complexity**: Depends on the grid resolution for storing medium properties and the film buffer [S1, unofficial].
*   **Strengths**: Unbiased estimation of transmittance in heterogeneous media without requiring spatial coherence or bounding volume hierarchies for acceleration structures [S2, S3, unofficial].
*   **Limitations**: Can be slow if the majorant is much larger than the local extinction coefficient, leading to many null-scattering events [S2, unofficial].
*   **When to use**: When estimating transmittance between two points in a participating medium within a null-scattering path integral framework, especially for connecting real-scattering vertices [S3, unofficial].

## Background
This method is part of the null-scattering path integral formulation of light transport [S3, unofficial]. It builds on the concept of ratio tracking introduced for transmittance estimation [S2, unofficial]. The implementation details are found in the Dr. TVAM project for inverse rendering in volumetric additive manufacturing [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
[S2] GitHub - yumcyaWiz/volpt: Volumetric path tracer written in C++, https://github.com/yumcyaWiz/volpt, fetched 2026-10-27 (unofficial)
[S3] A Null-Scattering Path Integral Formulation of Light Transport, https://www.scribd.com/document/1067625811/A-Null-scattering-Path-Integral-Formulation-of-Light-Transport, fetched 2026-10-29 (unofficial)
