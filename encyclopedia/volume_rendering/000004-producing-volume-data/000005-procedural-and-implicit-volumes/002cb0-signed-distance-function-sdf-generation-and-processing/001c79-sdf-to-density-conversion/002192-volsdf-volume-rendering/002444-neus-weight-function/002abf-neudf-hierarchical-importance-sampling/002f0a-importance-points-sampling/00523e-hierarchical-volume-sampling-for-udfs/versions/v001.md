# Hierarchical Volume Sampling for UDFs

Also called: UDF importance sampling, VRP-UDF hierarchical sampling

## Overview
Hierarchical Volume Sampling for UDFs is a technique to upsample points along a ray for Unsigned Distance Function (UDF) volume rendering, addressing the sampling bias inherent in standard importance sampling methods when applied to UDFs. Unlike Signed Distance Functions (SDFs), UDFs lack an explicit zero-level set distinction, causing standard sampling weight functions to concentrate points erroneously before the actual ray-surface intersection due to cumulative multiplicative effects [S1]. This method mitigates that bias by introducing an auxiliary point sampling prior to refine sampling densities and a uniform upsampling strategy to ensure even point distribution within intervals, thereby improving 3D perception and rendering efficiency [S1].

## How it works
**Inputs:**
- Original sampled points along a ray: $\{q_n\}_{n=1}^{N_1}$ [S1].
- UDF values at these points: $f_u(q_n)$ [S1].
- Interval lengths: $\delta_n = \|q_{n+1} - q_n\|$ [S1].
- Fixed standard deviation parameter: $s$ [S1].
- Number of points to upsample: $N_2$ [S1].
- Auxiliary point sampling prior network: $f_{wp}$ (optional, for bias mitigation) [S1].

**Outputs:**
- Set of $N_2$ upsampled points $\{q'_m\}_{m=1}^{N_2}$ distributed along the ray [S1].

**Data Structures:**
- Array of original points $Q = [q_1, ..., q_{N_1}]$ [S1].
- Array of sampling densities $\tau = [\tau_1, ..., \tau_{N_1}]$ [S1].
- Array of cumulative probabilities $\Omega = [\Omega_1, ..., \Omega_{N_1}]$ [S1].
- Array of uniform random numbers $U = [u_1, ..., u_{N_2}]$ where $u_i \sim \text{Uniform}[0, 1]$ [S1].

**Parameters:**
- $s$: Fixed standard deviation for the sampling density function. Typical value not specified in sources [S1].

**Steps:**

1. **Compute Sampling Densities ($\tau_n$):**
   For each original point $q_n$, calculate the sampling density $\tau_n$ using the UDF value $f_u(q_n)$ and the fixed standard deviation $s$:
   $$ \tau_n = \frac{s \cdot e^{-s \cdot f_u(q_n)}}{(1 + e^{-s \cdot f_u(q_n)})^2} $$
   [S1]

2. **Refine Densities with Auxiliary Prior (Optional but Recommended):**
   If using the auxiliary point sampling prior $f_{wp}$ to mitigate bias:
   a. Input a window of UDF values around $q_n$ into $f_{wp}$ to obtain a mask value $m_n \in [0, 1]$, representing the probability that the window contains a ray-surface intersection [S1].
   b. Update the sampling density: $\tau_n \leftarrow \tau_n \times m_n$ [S1].
   *Note:* $f_{wp}$ is trained separately using mask images as optimizing objectives to produce binary-like outputs, avoiding circular dependency with the volume rendering prior training [S1].

3. **Compute Cumulative Probabilities ($\Omega_n$):**
   Calculate the cumulative probability $\Omega_n$ that a new point falls between $q_n$ and $q_{n+1}$:
   $$ \Omega(n) = (1 - e^{-\tau_n \delta_n}) \times \prod_{n'=1}^{n-1} e^{-\tau_{n'} \delta_{n'}} $$
   where $\Omega(n)$ is the sampling weight for the interval $[q_n, q_{n+1}]$ [S1].
   *Implementation Note:* Compute the product term iteratively to avoid numerical underflow. Let $P_0 = 1$. For $n=1$ to $N_1-1$: $P_n = P_{n-1} \times e^{-\tau_{n-1} \delta_{n-1}}$, then $\Omega(n) = (1 - e^{-\tau_n \delta_n}) \times P_n$.

4. **Inverse Transform Sampling:**
   Generate $N_2$ uniform random numbers $u_m \sim \text{Uniform}[0, 1]$ [S1].
   Determine the initial positions of the upsampled points by finding the interval index $k$ for each $u_m$ such that $\sum_{i=1}^{k-1} \Omega(i) < u_m \le \sum_{i=1}^{k} \Omega(i)$, and then interpolating within that interval based on the inverse CDF $F^{-1}_{\Omega}$ [S1].
   *Note:* The exact interpolation formula within the interval is not explicitly detailed in the sources, but standard inverse transform sampling for piecewise constant densities implies linear interpolation within the selected interval $[q_k, q_{k+1}]$ based on the relative position of $u_m$ within the cumulative probability mass of that interval [G].

5. **Uniform Upsampling Strategy:**
   To correct the non-smooth distribution caused by discrete inverse transform sampling:
   a. Group the upsampled points that fall within the same interval $[q_n, q_{n+1}]$ into sets $\{q'_{n,1}, q'_{n,2}, ..., q'_{n,m}\}$, where $m$ is the number of points in that interval [S1].
   b. Re-order these $m$ points to be evenly distributed within the interval. The new positions are calculated such that the distance between consecutive points is $\frac{\delta_n}{m+1}$ [S1].
   Specifically, for $j=1$ to $m$:
   $$ q'_{n,j} = q_n + \frac{j}{m+1} \delta_n $$
   [S1]

**Edge Cases and Pitfalls:**
- **Sampling Bias:** Standard hierarchical sampling without the auxiliary prior concentrates points erroneously before the intersection due to the cumulative multiplicative effect of high sampling densities in non-intersection regions [S1].
- **Non-Uniform Distribution:** Inverse transform sampling on discrete intervals results in a non-smooth step function for the inverse CDF, leading to highly uneven point distribution and perception degeneration [S1]. The uniform upsampling strategy addresses this.
- **Circular Dependency:** Cannot directly use the trained volume rendering prior for $\tau_n$ because its training relies on hierarchical sampling, which itself suffers from bias. The auxiliary prior $f_{wp}$ is trained independently using mask images [S1].

## Subtopics

### Auxiliary Point Sampling Prior
An auxiliary neural network $f_{wp}$ that acts as a mask function for sampling densities. It takes a window of UDFs as input and outputs the probability that the window contains a ray-surface intersection. It is trained similarly to the volume rendering prior but uses mask images as optimizing objectives to produce binary outputs. The mask value is multiplied with the refined sampling densities $\{\tau_n\}$ for subsequent hierarchical sampling. Unlike standard hierarchical sampling which concentrates points erroneously before intersection, this prior effectively concentrates upsampled points near the actual intersection [S1].

### Uniform Upsampling Strategy
A post-processing step for inverse transform sampling results. It groups upsampled points falling within the same interval formed by original sampled points and re-orders them to evenly distribute them within the interval. This prevents perception degeneration caused by the non-smooth nature of the inverse CDF in discrete sampling [S1].

## Operating characteristics
- **Time Complexity:** The method involves computing densities, cumulative probabilities, and sorting/grouping points. The dominant cost is likely the evaluation of the UDF and the prior network $f_{wp}$ if used, and the sorting/grouping step for uniform upsampling. Not given in the sources.
- **Memory Complexity:** Requires storing original points, densities, cumulative probabilities, and upsampled points. Not given in the sources.
- **Measured Performance:** The method improves 3D perception and rendering efficiency by mitigating sampling bias and ensuring uniform point distribution [S1]. Specific numerical performance metrics (FPS, memory usage) are not provided in the sources.
- **Strengths:** Mitigates bias in non-intersection regions; ensures uniform distribution of upsampled points; improves 3D perception.
- **Limitations:** Requires training an auxiliary network $f_{wp}$; relies on UDF values; standard hierarchical sampling without this method suffers from bias.
- **When to use:** When rendering with UDFs and needing to upsample points along rays while avoiding the bias of standard importance sampling.
- **When not to use:** When SDFs are available and standard SDF sampling methods are sufficient; when computational resources for training $f_{wp}$ are not available.

## Background
Hierarchical volume sampling for UDFs was introduced in the VRP-UDF paper [S1]. It addresses the limitations of NeuS [1], which uses S-density functions with fixed standard deviations for importance sampling, a method that becomes complex and biased when applied to UDFs due to the lack of an explicit zero-level set. Recent solutions [2, 31] proposed tailored weight functions, but VRP-UDF identified significant bias in these approaches and introduced the auxiliary point sampling prior and uniform upsampling strategy to mitigate it [S1].

## Sources
[S1] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
