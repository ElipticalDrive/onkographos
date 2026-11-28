# Transient Nonlinear Beam Radiance Estimation

Also called: Transient beam estimation

## Overview
Transient Nonlinear Beam Radiance Estimation extends steady-state beam estimation to time-dependent light transport by reconstructing emission signals using time-of-flight (TOF) data stored in the photon map [S1]. It differs from naive 4D photon mapping by assuming quasi-static media and light sources, thereby reusing the spatial photon distribution and using TOF to reconstruct the temporal signal, which avoids the increased photon count required for temporal density estimation [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A photon map containing spatial nodes $\mathbf{u}_j$, each associated with a stored time-of-flight (TOF) value from the emitter to the node, and a light source identifier.
*   **Input:** The absolute observation time $t$ at the query point $\mathbf{x}$.
*   **Input:** The emission function of the light source, mapping absolute emission time to flux.
*   **Output:** The estimated transient basic radiance $\tilde{L}_b(\mathbf{x}, \boldsymbol{\omega}, t)$ along a curved beam direction $\boldsymbol{\omega}$ [S1].

**Data Structures**
The photon map stores nodes $\mathbf{u}_j$. Each node contains:
*   Position $\mathbf{u}_j$ [S1].
*   Time-of-flight (TOF) from the light source to $\mathbf{u}_j$, denoted as $TOF_j$ [S1].
*   Light source identifier [S1].
*   (Implicitly) The spatial flux contribution $\Delta \Phi_j$ associated with the photon, though the *time-dependent* flux is derived dynamically [S1].

**Numbered Steps**
1.  **Trace and Store TOF:** During the time integration of the curved beam trajectory from the light source to the query point $\mathbf{x}$, compute the TOF for each node $\mathbf{u}_j$ along the path. Store these TOF values together with the nodes in a list [S1].
2.  **Determine Node Time:** For each of the $k$ nearest neighbors $\mathbf{u}_j$ to $\mathbf{x}$, determine the time $t'(\mathbf{u}_j)$ at which the photon arrived at $\mathbf{u}_j$. This is calculated as the absolute observation time $t$ minus the TOF from $\mathbf{x}$ to $\mathbf{u}_j$ [S1].
    *   Note: The TOF from $\mathbf{x}$ to $\mathbf{u}_j$ is derived from the stored TOF values. If $TOF_j$ is the TOF from source to $\mathbf{u}_j$, and $TOF_x$ is the TOF from source to $\mathbf{x}$, the TOF between them is $|TOF_x - TOF_j|$ assuming straight-line propagation between nodes or integrated path length divided by speed of light. The source states $t'(\mathbf{u}_j)$ is $t$ minus TOF from $\mathbf{x}$ to $\mathbf{u}_j$ [S1].
3.  **Reconstruct Emission Time:** Using the stored TOF of the photon at $\mathbf{u}_j$ (from source to $\mathbf{u}_j$) and the calculated arrival time $t'(\mathbf{u}_j)$, reconstruct the absolute emission time $t_{emit}$ of the photon.
    *   $t_{emit} = t'(\mathbf{u}_j) - TOF_{source \to \mathbf{u}_j}$ [S1].
4.  **Lookup Time-Dependent Flux:** Using the light source identifier and the reconstructed emission time $t_{emit}$, perform a lookup in the light source's emission function to determine the time-dependent flux $\Delta \Phi_j(t'(\mathbf{u}_j))$ that the photon contributes with respect to absolute time $t$ [S1].
5.  **Estimate Radiance:** Compute the sum over the $k$ nearest neighbors using the formula below [S1].

**Formula**
The estimated transient basic radiance is:
$$
\tilde{L}_b(\mathbf{x}, \boldsymbol{\omega}, t) \approx \sum_{j=1}^k \tau(\mathbf{u}_j, \mathbf{x}) \frac{\sigma_t(\mathbf{u}_j)}{n(\mathbf{u}_j)^2} K(r_j, d_j) \Delta \Phi_j(t'(\mathbf{u}_j)) P(\boldsymbol{\omega}_j, \boldsymbol{\omega}'(\mathbf{u}_j))
$$
Where:
*   $\tau(\mathbf{u}_j, \mathbf{x})$ is the transmittance from node $\mathbf{u}_j$ to query point $\mathbf{x}$ [S1].
*   $\sigma_t(\mathbf{u}_j)$ is the extinction coefficient at $\mathbf{u}_j$ [S1].
*   $n(\mathbf{u}_j)$ is the refractive index at $\mathbf{u}_j$ [S1].
*   $K(r_j, d_j)$ is the kernel function for density estimation with radius $r_j$ and dimension $d_j$ [S1].
*   $\Delta \Phi_j(t'(\mathbf{u}_j))$ is the time-dependent flux contribution of the $j$-th photon [S1].
*   $P(\boldsymbol{\omega}_j, \boldsymbol{\omega}'(\mathbf{u}_j))$ is the phase function evaluating the scattering from the photon's incoming direction $\boldsymbol{\omega}_j$ to the beam direction $\boldsymbol{\omega}'(\mathbf{u}_j)$ at $\mathbf{u}_j$ [S1].

**Parameters**
*   $k$: Number of nearest neighbors. Not given in the sources.
*   $r_j$: Search radius for the $j$-th neighbor. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Quasi-static Assumption:** The method assumes the medium and light sources are quasi-static. If the medium properties change rapidly over time, the spatial photon distribution may not be valid for the current time step [S1].
*   **High-Frequency Emission:** If light source emission functions contain high frequencies, the assumption of reusing spatial photons might lead to inaccuracies if the temporal resolution of the TOF is insufficient. However, the method claims to avoid loss of information by using TOF [S1].
*   **Refractive Index Cancellation:** The term $n^2(\mathbf{x})$ in the radiance definition does not cancel out with $1/n(\mathbf{u}_j)^2$ in the estimator because these values depend on different positions along the ray [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the nearest neighbor search in the photon map and the emission function lookup. Not given in the sources.
*   **Memory Complexity:** Requires storing TOF values for each photon in the map, adding a small overhead to the standard spatial photon map. Not given in the sources.
*   **Strengths:** Avoids the need for a 4D photon map (spatial + time), which would require significantly more photons, especially for high-frequency emission functions. Reuses the spatial photon distribution for each point in time [S1].
*   **Limitations:** Relies on the quasi-static assumption for media and light sources. Not suitable for highly dynamic participating media [S1].

## Background
*   **Origin:** The method is an extension of the nonlinear beam estimation technique for steady-state rendering, adapted for transient rendering by incorporating time-of-flight data [S1].
*   **History:** It builds upon standard photon mapping extensions for transient rendering, such as those by Jarabo [2012] for surface-based transient rendering, but applies them to volumetric participating media with continuous refraction [S1].
*   **Implementation:** Implemented in C++ as a custom integrator plugin in the Mitsuba renderer [S1].

## Sources
[S1] Refractive radiative transfer equation, https://doi.org/10.1145/2557605, fetched 2026-10-31
