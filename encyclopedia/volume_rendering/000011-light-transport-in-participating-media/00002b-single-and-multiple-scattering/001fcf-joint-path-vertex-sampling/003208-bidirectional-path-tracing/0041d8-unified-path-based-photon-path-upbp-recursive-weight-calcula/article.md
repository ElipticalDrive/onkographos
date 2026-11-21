# Unified Path-based Photon Path (UPBP) Recursive Weight Calculation

Also called: Recursive d-quantities, UPBP Recursive MIS Weight Computation, Unified Path-Based Path Tracing recursive weight

## Overview
UPBP Recursive Weight Calculation is a method for efficiently computing Multiple Importance Sampling (MIS) weights in Bidirectional Path Tracing (BPT) and Photon Density Estimation (PDE) estimators within participating media. It solves the problem of expensive path weight re-evaluation by caching partial sums that remain constant as subpaths are extended, avoiding the storage of full path weights. Unlike methods that store full path weights, UPBP stores only the recursive $d$-quantities, which are updated incrementally as the subpath is traced.

## How it works
**Inputs and Outputs:**
*   **Input:** Light subpath vertices $\bar{\mathbf{y}}$ with $s$ vertices, camera subpath vertices $\bar{\mathbf{z}}$ with $t$ vertices, technique $v$ (BPT or PDE), and estimator-specific parameters.
*   **Output:** MIS weight $w_{v,s,t}$ for the light path formed by connecting the subpaths.

**Data Structures:**
For each vertex $i$ in a subpath, store three scalar quantities:
*   $d_i^{\text{shared}}$: Shared recursive quantity.
*   $d_i^{\text{BPT}}$: BPT-specific recursive quantity.
*   $d_i^{\text{PDE}}$: PDE-specific recursive quantity.

**Definitions:**
*   $p_i^{\rightarrow}$: PDF of vertex $i$ in the forward direction (light or camera subpath generation).
*   $g_i^{\leftarrow}$: Geometric term for the connection from vertex $i$ to the previous vertex in the reverse direction.
*   $f_i$: Phase function or scattering term at vertex $i$.
*   $p_{\tau,i-2}^{\leftarrow}$: Transition probability term for vertex $i-2$ in reverse direction.
*   $p_{\omega,i-2}^{\leftarrow}$: Solid angle term for vertex $i-2$ in reverse direction.
*   $n_{\text{BPT}}$: Number of light paths used in the BPT estimator normalization.
*   $n_v$: Estimator-specific term for technique $v$.
*   $e_{v,s}$: Estimator-specific term for technique $v$ on a light subpath with $s$ vertices.
*   $e_{v,t}$: Estimator-specific term for technique $v$ on a camera subpath with $t$ vertices.

**Numbered Steps:**

1.  **Initialization ($i=1$):**
    Compute initial values for the first stored vertex ($i=1$):
    $$d_1^{\text{shared}} = 1/p_1^{\rightarrow}$$
    $$d_1^{\text{BPT}} = g_0^{\leftarrow}/(p_0^{\rightarrow} p_1^{\rightarrow})$$
    $$d_1^{\text{PDE}} = (g_0^{\leftarrow} n_{\text{BPT}})/(p_0^{\rightarrow} p_1^{\rightarrow})$$
    Store these values. Note: The first vertices $\mathbf{y}_0$ and $\mathbf{z}_0$ are not stored [S1].

2.  **Recursive Update ($i > 1$):**
    For each subsequent vertex $i$, update the quantities using the previous vertex's values ($i-1$):
    $$d_i^{\text{shared}} = 1/p_i^{\rightarrow}$$
    $$d_i^{\text{BPT}} = \frac{g_{i-1}^{\leftarrow}}{p_i^{\rightarrow}} \left( \frac{f_i}{n_{\text{BPT}}} + d_{i-1}^{\text{shared}} + p_{\tau,i-2}^{\leftarrow} p_{\omega,i-2}^{\leftarrow} d_{i-1}^{\text{BPT}} \right)$$
    $$d_i^{\text{PDE}} = \frac{g_{i-1}^{\leftarrow}}{p_i^{\rightarrow}} \left( f_i + n_{\text{BPT}} d_{i-1}^{\text{shared}} + p_{\tau,i-2}^{\leftarrow} p_{\omega,i-2}^{\leftarrow} d_{i-1}^{\text{PDE}} \right)$$
    Store these values for vertex $i$ [S1].

3.  **Weight Construction for BPT:**
    When connecting a light subpath with $s$ vertices and a camera subpath with $t$ vertices using BPT, compute the partial weights using the stored $d$-quantities at vertices $s-1$ and $t-1$:
    $$\bar{w}_{v,s-1}(\bar{\mathbf{y}}) = p_{s-1}^{\leftarrow} \left( \frac{f_s}{n_{\text{BPT}}} + d_{s-1}^{\text{shared}} + p_{\tau,s-2}^{\leftarrow} p_{\omega,s-2}^{\leftarrow} d_{s-1}^{\text{BPT}} \right)$$
    $$\bar{w}_{v,t-1}(\bar{\mathbf{z}}) = p_{t-1}^{\leftarrow} \left( \frac{f_t}{n_{\text{BPT}}} + d_{t-1}^{\text{shared}} + p_{\tau,t-2}^{\leftarrow} p_{\omega,t-2}^{\leftarrow} d_{t-1}^{\text{BPT}} \right)$$
    These equations hold for $s > 1$ and $t > 1$ [S1].

4.  **Weight Construction for PDE:**
    When using PDE techniques, compute the partial weights:
    $$\bar{w}_{v,s-1}(\bar{\mathbf{y}}) = \frac{1}{n_v e_{v,s}} (n_{\text{BPT}} d_{s-1}^{\text{shared}} + p_{\tau,s-2}^{\leftarrow} p_{\omega,s-2}^{\leftarrow} d_{s-1}^{\text{PDE}})$$
    $$\bar{w}_{v,t-1}(\bar{\mathbf{z}}) = \frac{1}{n_v e_{v,t}} (n_{\text{BPT}} d_{t-1}^{\text{shared}} + p_{\tau,t-2}^{\leftarrow} p_{\omega,t-2}^{\leftarrow} d_{t-1}^{\text{PDE}})$$
    PDE techniques are only evaluated when $s > 1$ and $t > 1$ [S1].

5.  **Final MIS Weight:**
    Compute the total weight as:
    $$w_{v,s,t} = 1/(w_{v,s}^{\text{light}} + w_{v,s}^{\text{local}} + w_{v,s}^{\text{camera}})$$
    where $w_{v,s}^{\text{light}}$ depends only on light subpath vertices, $w_{v,s}^{\text{local}}$ depends on local connection/merge information, and $w_{v,s}^{\text{camera}}$ depends only on camera subpath vertices. For BPT, $w_{v,s}^{\text{local}} = 1$. For PDE, $w_{v,s}^{\text{local}}$ is given by specific estimator formulas [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_d_quantities(p_0_forward, p_1_forward, g_0_reverse):
    """
    Initialize d-quantities for the first stored vertex (i=1).
    """
    d_shared = 1.0 / p_1_forward
    d_bpt = g_0_reverse / (p_0_forward * p_1_forward)
    d_pde = (g_0_reverse * n_BPT) / (p_0_forward * p_1_forward)
    return d_shared, d_bpt, d_pde

def update_d_quantities(i, p_i_forward, g_i_minus_1_reverse, f_i, 
                        p_tau_i_minus_2_reverse, p_omega_i_minus_2_reverse,
                        d_shared_prev, d_bpt_prev, d_pde_prev):
    """
    Update d-quantities for vertex i based on previous vertex i-1.
    """
    d_shared = 1.0 / p_i_forward
    d_bpt = (g_i_minus_1_reverse / p_i_forward) * (
        (f_i / n_BPT) + d_shared_prev + 
        (p_tau_i_minus_2_reverse * p_omega_i_minus_2_reverse * d_bpt_prev)
    )
    d_pde = (g_i_minus_1_reverse / p_i_forward) * (
        f_i + (n_BPT * d_shared_prev) + 
        (p_tau_i_minus_2_reverse * p_omega_i_minus_2_reverse * d_pde_prev)
    )
    return d_shared, d_bpt, d_pde

def compute_bpt_weights(s, t, p_s_minus_1_reverse, p_t_minus_1_reverse, 
                        f_s, f_t, d_shared_s_minus_1, d_bpt_s_minus_1,
                        d_shared_t_minus_1, d_bpt_t_minus_1,
                        p_tau_s_minus_2_reverse, p_omega_s_minus_2_reverse,
                        p_tau_t_minus_2_reverse, p_omega_t_minus_2_reverse):
    """
    Compute partial weights for BPT estimator.
    Assumes s > 1 and t > 1.
    """
    w_light = p_s_minus_1_reverse * (
        (f_s / n_BPT) + d_shared_s_minus_1 + 
        (p_tau_s_minus_2_reverse * p_omega_s_minus_2_reverse * d_bpt_s_minus_1)
    )
    w_camera = p_t_minus_1_reverse * (
        (f_t / n_BPT) + d_shared_t_minus_1 + 
        (p_tau_t_minus_2_reverse * p_omega_t_minus_2_reverse * d_bpt_t_minus_1)
    )
    return w_light, w_camera

def compute_pde_weights(s, t, n_v, e_v_s, e_v_t,
                        d_shared_s_minus_1, d_pde_s_minus_1,
                        d_shared_t_minus_1, d_pde_t_minus_1,
                        p_tau_s_minus_2_reverse, p_omega_s_minus_2_reverse,
                        p_tau_t_minus_2_reverse, p_omega_t_minus_2_reverse):
    """
    Compute partial weights for PDE estimator.
    Assumes s > 1 and t > 1.
    """
    w_light = (1.0 / (n_v * e_v_s)) * (
        (n_BPT * d_shared_s_minus_1) + 
        (p_tau_s_minus_2_reverse * p_omega_s_minus_2_reverse * d_pde_s_minus_1)
    )
    w_camera = (1.0 / (n_v * e_v_t)) * (
        (n_BPT * d_shared_t_minus_1) + 
        (p_tau_t_minus_2_reverse * p_omega_t_minus_2_reverse * d_pde_t_minus_1)
    )
    return w_light, w_camera

def compute_mis_weight(w_light, w_local, w_camera):
    """
    Compute final MIS weight.
    """
    return 1.0 / (w_light + w_local + w_camera)
```

**Edge Cases and Pitfalls:**
*   **First Vertices:** The first vertices $\mathbf{y}_0$ and $\mathbf{z}_0$ are not stored. Equations (43)-(46) only hold for $s > 1$ and $t > 1$. Special handling is required for $s \leq 1$ or $t \leq 1$ [S1].
*   **PDF Zero Conditions:**
    *   BPT PDF must be zero when $\mathbf{y}_{s-1}$ or $\mathbf{z}_{t-1}$ is on a specular surface [S1].
    *   B-B1D, P-B2D, and P-P3D PDFs must be zero when $\mathbf{y}_{s-1}$ or $\mathbf{z}_{t-1}$ are not in media [S1].
    *   P-P2D PDF must be zero when $\mathbf{y}_{s-1}$ or $\mathbf{z}_{t-1}$ are in media or on a specular surface [S1].

## Operating characteristics
*   **Time Complexity:** The recursive update is $O(1)$ per vertex, making the overall complexity linear with respect to the number of vertices in the subpaths. This is more efficient than recomputing full path weights for each connection.
*   **Memory Complexity:** Stores three scalar quantities per vertex per subpath, significantly less than storing full path weights.
*   **Strengths:** Efficient computation of MIS weights for BPT and PDE in participating media. Avoids redundant calculations by caching partial sums.
*   **Limitations:** Requires careful handling of edge cases for short subpaths ($s \leq 1$ or $t \leq 1$). Specific PDF zero conditions must be enforced for different estimators.
*   **When to Use:** When implementing bidirectional path tracing or photon density estimation in participating media, especially when multiple connection techniques are used.
*   **When Not to Use:** For simple path tracing without bidirectional connections or photon density estimation.

## Background
UPBP Recursive Weight Calculation was introduced in the paper "Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation" [S1]. It is part of the Unified Path-based Photon Path (UPBP) framework, which aims to efficiently simulate light transport in participating media using hardware-accelerated ray tracing. The method builds on [Bidirectional Path Tracing](6666-bidirectional-path-tracing.md) by providing an efficient way to compute MIS weights for various connection techniques in media.

## Sources
[S1] Efficient Hardware Acceleration of Robust Volumetric Light Transport Simulation, https://doi.org/10.1111/cgf.14802, fetched 2026-10-27
