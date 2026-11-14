# Null-Scattering Path Integral Framework

Also called: Generalized Path Space for Volume Rendering, Path Integral with Null Scattering

## Overview
The Null-Scattering Path Integral Framework formulates light transport in participating media as a Monte Carlo integration problem over a generalized path space that explicitly includes null-scattering vertices [S1]. It solves the problem of defining a rigorous measure for path integrals in heterogeneous media where standard delta tracking or majorant sampling does not naturally fit into a path-space summation, enabling the application of variance reduction techniques like multiple importance sampling and bidirectional integrators [S1]. It differs from standard path tracing by treating null-scattering events as distinct vertex types in the path configuration, allowing the path throughput to be decomposed into majorant transmittance and scattering terms that can be sampled independently [S1].

## How it works
The framework defines a path space over sequences of vertices that can be of three types: surface locations ($A$), real scattering positions in the medium ($V$), and null-scattering positions in the medium ($V_\emptyset$) [S1].

### Inputs and Outputs
The framework takes a path $\bar{p}_n$ of length $n$ and a configuration vector $\mathbf{c} \in \{0, 1, 2\}^n$ defining vertex types as input, and outputs the path contribution estimator $\hat{P}(\bar{p}_n)$ and the throughput weight $\beta(\bar{p}_n)$ [S1].

### Data Structures
The path $\bar{p}_n$ is a sequence of $n$ vertices $p_1, \dots, p_n$ [S1]. Each vertex $p_i$ has a type determined by the configuration vector $\mathbf{c}$:
* $c_i = 0$: Surface vertex ($p_i \in A$).
* $c_i = 1$: Real scattering vertex ($p_i \in V$).
* $c_i = 2$: Null-scattering vertex ($p_i \in V_\emptyset$) [S1].

The subset of vertices representing real scattering is denoted $r_i$ [S1]. Note that the index of a real-scattering vertex in the full path $\bar{p}_n$ may differ from its index in the subset $r$ [S1].

### Formulas
The set of paths for a specific configuration $\mathbf{c}$ is the Cartesian product:
$$
\mathcal{P}_n^{\mathbf{c}} = \prod_{i=1}^{n} \begin{cases} A & \text{if } c_i = 0 \\ V & \text{if } c_i = 1 \\ V_\emptyset & \text{if } c_i = 2 \end{cases}
$$
The set of all paths of length $n$ is the union over all configuration vectors:
$$
\mathcal{P}_n = \bigcup_{\mathbf{c} \in \{0, 1, 2\}^n} \mathcal{P}_n^{\mathbf{c}}
$$
The measure $\mu_n$ for a domain $D \subseteq \mathcal{P}_n$ sums over configurations $\mathbf{c} \in \{0, 1, 2\}^n$ [S1]:
$$
\mu_n(D) = \sum_{\mathbf{c} \in \{0, 1, 2\}^n} \mu_n^{\mathbf{c}}(D \cap \mathcal{P}_n^{\mathbf{c}})
$$
where the differential measure $d\mu_n^{\mathbf{c}}$ uses $dA(p_i)$ if $c_i=0$, $dV(p_i)$ if $c_i=1$, and $dV_\emptyset(p_i)$ if $c_i=2$ [S1]. The measure $dV_\emptyset$ incorporates a Dirac delta distribution to restrict integration to the line segment between successive real-scattering vertices [S1].

The generalized path contribution function is:
$$
\hat{P}(\bar{p}_n) = \int_{\mathcal{P}_{n-1}} \hat{L}_e(p_n \to p_{n-1}) \hat{T}(\bar{p}_n) d\mu_{n-1}(p_2, \dots, p_n)
$$
where the emitted radiance term $\hat{L}_e$ is:
$$
\hat{L}_e(p_n \to p_{n-1}) = \begin{cases} L_e(\dots) & \text{if } p_n \in A \\ \sigma_a(p_n) L_e(\dots) & \text{if } p_n \in V \end{cases}
$$
The generalized path throughput $\hat{T}(\bar{p}_n)$ is:
$$
\hat{T}(\bar{p}_n) = \left( \prod_{i=1}^{n-1} \hat{f}(p_{i+1} \to p_i \to p_{i-1}) \right) \left( \prod_{i=0}^{n-1} T_{maj}(p_i \to p_{i+1}) \right) \left( \prod_{i=1}^{m-1} \hat{G}(r_i \leftrightarrow r_{i+1}) \right)
$$
where $m$ is the number of real-scattering vertices [S1].

The generalized scattering distribution $\hat{f}$ is:
$$
\hat{f}(p_{i+1} \to p_i \to p_{i-1}) = \begin{cases} f(\dots) & \text{if } p_i \in A \\ \sigma_s(p_i) p(\dots) & \text{if } p_i \in V \\ \sigma_n(p_i) H((p_i - p_{i+1}) \cdot (p_i - p_{i-1})) & \text{if } p_i \in V_\emptyset \end{cases}
$$
where $\sigma_n$ is the null-scattering coefficient, $H$ is the Heaviside function, and $p(\dots)$ is the phase function [S1].

The generalized geometric term $\hat{G}$ is:
$$
\hat{G}(p \leftrightarrow p') = V(p \leftrightarrow p') \frac{C_p(p, p') C_{p'}(p', p)}{\|p - p'\|}
$$
where $C_p(p, p') = |n_p \cdot \frac{p - p'}{\|p - p'\|}|$ if $p \in A$, and $1$ if $p \in V$ [S1]. Note that $C_p$ is only evaluated for real-scattering vertices $r_i$ [S1].

The Monte Carlo estimator is:
$$
\hat{P}(\bar{p}_n) = \frac{\hat{T}(\bar{p}_n) \hat{L}_e(p_n \to p_{n-1})}{p(\bar{p}_n)}
$$
where $p(\bar{p}_n)$ is the probability density of sampling the path with respect to the generalized measure [S1].

The volumetric path throughput weight is:
$$
\beta(\bar{p}_n) = \frac{\hat{T}(\bar{p}_n)}{p(\bar{p}_n)}
$$ [S1].

### Pseudocode
The following pseudocode outlines the evaluation of the path throughput and emitted radiance term for a given path $\bar{p}_n$ and its configuration $\mathbf{c}$ [S1]. It assumes access to scene queries for $\sigma_a, \sigma_s, \sigma_{maj}, T_{maj}$, and surface normals [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b):
    return sum(x * y for x, y in zip(a, b))

def distance(a, b):
    return math.sqrt(sum((x - y)**2 for x, y in zip(a, b)))

def evaluate_null_scattering_path_integral(path, config):
    """
    Evaluates the path throughput and emitted radiance term for a path in the null-scattering framework.
    
    Args:
        path: List of vertices p_0, ..., p_{n-1}. Each vertex is a dict with 'pos' (vector), 'type' (int: 0=surface, 1=real, 2=null), 'normal' (vector, if surface), 'emitted_radiance' (float), 'bsdf' (float), 'phase_function' (float), 'absorption_coefficient' (float), 'scattering_coefficient' (float), 'null_scattering_coefficient' (float).
        config: List of integers c_0, ..., c_{n-1} matching vertex types.
        
    Returns:
        T_hat: Generalized path throughput.
        L_e_hat: Emitted radiance term at the endpoint.
    """
    n = len(path)
    if n < 2:
        return 1.0, 0.0

    # 1. Calculate Emitted Radiance Term L_hat_e at the last vertex p_{n-1}
    p_n = path[-1]
    c_n = config[-1]
    
    if c_n == 0: # Surface
        L_e_hat = p_n.get('emitted_radiance', 0.0)
    elif c_n == 1: # Real Volume
        sigma_a = p_n.get('absorption_coefficient', 0.0)
        L_e_hat = sigma_a * p_n.get('emitted_radiance', 0.0)
    else: # Null Volume (c_n == 2)
        L_e_hat = 0.0

    # 2. Calculate Generalized Path Throughput T_hat
    T_hat = 1.0
    
    # Product 1: Generalized Scattering Distribution f_hat over intermediate vertices
    # Corresponds to \prod_{i=1}^{n-2} \hat{f}(p_{i+1} \to p_i \to p_{i-1}) in 0-based indexing
    for i in range(1, n - 1):
        p_prev = path[i-1]
        p_curr = path[i]
        p_next = path[i+1]
        c_curr = config[i]
        
        if c_curr == 0: # Surface
            f_hat = p_curr.get('bsdf', 1.0)
        elif c_curr == 1: # Real Volume
            sigma_s = p_curr.get('scattering_coefficient', 0.0)
            phase = p_curr.get('phase_function', 1.0)
            f_hat = sigma_s * phase
        elif c_curr == 2: # Null Volume
            sigma_n = p_curr.get('null_scattering_coefficient', 0.0)
            # Heaviside function H checks if the direction is consistent (no backtracking)
            dot_product = dot(p_curr['pos'] - p_next['pos'], p_curr['pos'] - p_prev['pos'])
            H = 1.0 if dot_product >= 0 else 0.0
            f_hat = sigma_n * H
        else:
            f_hat = 0.0
            
        T_hat *= f_hat

    # Product 2: Majorant Transmittance T_maj over all segments
    # Corresponds to \prod_{i=0}^{n-2} T_{maj}(p_i \to p_{i+1})
    for i in range(n - 1):
        p_curr = path[i]
        p_next = path[i+1]
        dist = distance(p_curr['pos'], p_next['pos'])
        # Majorant extinction is typically constant or per-segment; here assumed per-vertex for simplicity [G]
        sigma_maj = p_curr.get('majorant_extinction', 0.0)
        T_hat *= math.exp(-dist * sigma_maj)

    # Product 3: Generalized Geometric Term G_hat for consecutive real scattering vertices
    real_scatter_indices = [i for i, c in enumerate(config) if c == 1]
    for j in range(len(real_scatter_indices) - 1):
        idx_curr = real_scatter_indices[j]
        idx_next = real_scatter_indices[j+1]
        p_curr = path[idx_curr]
        p_next = path[idx_next]
        
        dist = distance(p_curr['pos'], p_next['pos'])
        if dist > 0:
            # C_p and C_p' are 1 for volume vertices. Visibility V is scene-dependent [G].
            # Assuming V=1 and cosines=1 for the base framework evaluation.
            T_hat *= 1.0 / dist
        else:
            T_hat *= 0.0

    return T_hat, L_e_hat
```

### Parameters and Edge Cases
*   **Dirac Delta**: The measure $dV_\emptyset$ restricts null-scattering vertices to the line between real-scattering vertices [S1].
*   **Heaviside Function**: Ensures that null-scattering directions are consistent with the path direction (no backtracking) [S1].
*   **Configuration Vector**: The framework handles arbitrary sequences of surface, real, and null vertices [S1].

## Subtopics

### [Null-Scattering Volumetric Path Tracing](3829-null-scattering-volumetric-path-tracing.md)
An unbiased algorithm that uses the framework to sample distances from the majorant distribution and explicitly maintains the majorant transmittance factor, ensuring unbiased estimates even with significant majorant overestimation.

### [Surrounding Path Contribution (WKB Expansion)](1300-surrounding-path-contribution-wkb-expansion.md)
A sibling method that uses a WKB expansion to approximate the path contribution, differing in its approach to handling multiple scattering and variance reduction.

## Operating characteristics
*   **Complexity**: The complexity depends on the sampling strategy [S1]. The framework itself is a formulation, not a specific sampler [S1]. The number of path vertices $n$ and the number of real-scattering vertices $m$ affect the cost of evaluating $\hat{T}$ [S1].
*   **Performance**: The framework enables variance reduction techniques like MIS, which can significantly improve performance in complex scenes with heterogeneous media [S1].
*   **Strengths**: Rigorous path-space formulation allows for bidirectional and multi-bounce integrators in participating media [S1]. Handles heterogeneous media via the majorant and null-scattering terms [S1].

## Background
The framework was introduced in the context of the Path Tracing book (PBR Book), specifically in Section 14.1.4, to express the null-scattering integral equation of transfer as a sum over paths [S1]. It builds on the surface path space formulation from Section 13.1.4 and extends it to include volume and null-scattering vertices [S1].

## Sources
[S1] Light Transport II: Volume Rendering / The Equation of Transfer, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27
