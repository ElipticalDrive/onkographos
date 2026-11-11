# Light Propagation Maps

Also called: Fattal's LPM, Fattal's Original LPM Algorithm, Fattal's method, Original LPM iteration

## Overview
Light Propagation Maps (LPM) is a Discrete Ordinate Method (DOM) technique for computing multiple scattering in participating media [S1]. It approximates the radiative transfer equation by decoupling spatial and directional sampling: it uses high-resolution 2D maps for iterative light propagation to minimize false scattering and ray effects, while using coarse 3D grids for storing accumulated radiance [S1]. This approach reduces memory usage and artifacts compared to standard DOM techniques that require fine sampling in both space and direction [S1].

## How it works
**Inputs:**
*   3D voxel grid representing the medium, with constant absorption ($K_a$), scattering ($K_s$), and self-emission coefficients per voxel [S1].
*   Boundary conditions (incoming radiance) [S1].
*   Phase function $p(\boldsymbol{\omega}, \boldsymbol{\omega}')$ [S1].
*   Termination threshold $\varepsilon$ [S1].
*   Number of propagation directions $m$ (where $m \ll n$, the total solid angle samples) [S1].

**Outputs:**
*   Average scattered radiance $I_{xyz}^d$ for each voxel $xyz$ and storage direction $d$ [S1].

**Data Structures:**
*   **Light Propagation Maps (LPMs):** A set of 2D arrays. Each LPM corresponds to a face of the coarse 3D grid. Each element $(r,s)$ in an LPM stores a set of rays originating at $(r,s)$ with directions sampled from the unit sphere $S^2$ [S1].
*   **Radiance Buffers:**
    *   $I_{xyz}^d$: Stores the average scattered radiance in voxel $xyz$ along direction $\Omega_d$ [S1].
    *   $U_{xyz}^{i,p}$: Stores the unscattered radiance contribution for propagation direction $i$ at iteration $p$ [S1].
    *   In practice, only two $U$ buffers are kept (for iterations $p-1$ and $p$) using a swap mechanism to reuse memory [S1].

**Definitions:**
*   $V_{xyz}^d = \Delta V |\Omega_d|$: Product of voxel volume $\Delta V$ and solid angle $|\Omega_d|$ [S1].
*   $\omega_z^i$: The z-component of propagation direction $i$ [S1].
*   $\omega_s^i$: The component of direction $i$ along the propagation axis of the current LPM plane [S1].
*   $\Delta l_q$: Distance traversed by a ray in the $t$-th voxel [S1].
*   $A_{r,s}$: Area of the LPM sample element $(r,s)$ [S1].
*   $F^{i,d}$: Precomputed phase function integral for propagation direction $i$ and storage direction $d$ (where $d$ is the direction closest to $i$) [S1].

**Formulas:**
1.  **Radiance Update (Ray Marching):**
    The radiance $L_{r,s}^i$ leaving the $t$-th voxel encountered by a ray is:
    $$ L_{r,s}^i = L_{r,s}^{i,t-1} e^{-\Delta l_q(K_a + K_s)/\omega_z^i} + U_{xyz}^{i,p} \times \frac{1 - e^{-\Delta l_q(K_a + K_s)/\omega_z^i}}{K_a + K_s} \quad (3) $$
    where $L_{r,s}^{i,t-1}$ is the radiance entering the voxel, and $U_{xyz}^{i,p}$ is the unscattered radiance from the previous iteration stored in the voxel [S1].

2.  **Scattering Contribution:**
    When a ray adds radiance to a voxel, the contribution $R$ scattered into all directions is:
    $$ R = (V_{xyz}^d)^{-1} A_{r,s} F^{i,d} L_{r,s}^{i,t} \frac{1 - e^{-\Delta l_q K_s}}{\omega_s^i} \quad (4) $$
    This value updates both the average radiance and the unscattered radiance for the next iteration:
    $$ I_{xyz}^d = R $$
    $$ U_{xyz}^d = R $$
    [S1]

3.  **Phase Function Integral:**
    $$ F^{i,d} = \frac{1}{4\pi} \int_{\Omega^i} \int_{\Omega^d} p(\boldsymbol{\omega}, \boldsymbol{\omega}') d\boldsymbol{\omega} d\boldsymbol{\omega}' \quad (5) $$
    where superscripts $i$ and $d$ are for the propagation direction and storage direction respectively, and $d$ is the direction closest to $i$ [S1].

**Algorithm Steps:**
1.  Initialize iteration counter $p_{prev} = 0$ [S1].
2.  Initialize $U_{d,p_{prev},xyz}$ and $I_{d,xyz}$ with the medium's self-emission radiance (if any) [S1].
3.  While $\max_{xyz}|U_{d,p_{prev},xyz}| \ge \varepsilon$:
    a.  Set $p_{cur} = (p_{prev} + 1) \% 2$ [S1].
    b.  Initialize $U_{d,p_{cur},xyz}$ to 0 [S1].
    c.  For each LPM:
        i.  For each propagation direction $i$:
            1.  For each element $(r,s)$ of the current map:
                *   If $p == 0$, initialize $L_{r,s}^{i,0}$ with boundary conditions (incoming radiance).
                *   Else, set $L_{r,s}^{i,0}$ to 0 [S1].
            2.  For each intersected voxel $t$ along the ray path:
                *   Update ray radiance $L_{r,s}^{i,t}$ using Eq. (3) and $U_{d,p_{prev},xyz}$ [S1].
                *   Calculate scattering contribution $R$ using Eq. (4) [S1].
                *   Update $U_{d,p_{cur},xyz}$ and $I_{d,xyz}$ with $R$ [S1].
                *   **Boundary Handling:** When a ray leaves the medium at boundary point A, generate a new ray of the same direction with zero radiance from the opposite point B to maintain a constant number of rays [S1].
    d.  Set $p_{prev} = p_{cur}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def light_propagation_maps(volume, boundary_conditions, phase_function, epsilon):
    # volume: 3D list of voxel objects with attributes: Ka, Ks, emission, volume, delta_l
    # boundary_conditions: callable(pos, direction) -> radiance
    # phase_function: callable(omega, omega_prime) -> value
    # epsilon: float, termination threshold

    # Helper: standard numerical integration over solid angles [G]
    def integrate_phase(i, d):
        # Approximate integral of phase function over propagation direction i and storage direction d
        return (1.0 / (4.0 * math.pi)) * sum(phase_function(w, wp) for w, wp in sample_solid_angles(i, d))

    # Precompute phase integrals F[i][d] for all propagation directions i and storage directions d [S1]
    F = {}
    for i in propagation_directions:
        F[i] = {}
        for d in storage_directions:
            F[i][d] = integrate_phase(i, d)

    # Initialize buffers [S1]
    U_prev = [[[0.0 for _ in range(Z)] for _ in range(Y)] for _ in range(X)]
    U_curr = [[[0.0 for _ in range(Z)] for _ in range(Y)] for _ in range(X)]
    I = [[[0.0 for _ in range(Z)] for _ in range(Y)] for _ in range(X)]

    # Initialize with self-emission radiance [S1]
    for x in range(X):
        for y in range(Y):
            for z in range(Z):
                U_prev[x][y][z] = volume[x][y][z].emission
                I[x][y][z] = volume[x][y][z].emission

    buf_prev = 0
    buf_curr = 1
    it = 1

    # Get LPM faces (2D arrays corresponding to volume faces) [S1]
    lpm_faces = get_lpm_faces(volume)

    while max(max(max(row) for row in buf) for buf in U_prev) >= epsilon:
        buf_curr = 1 - buf_prev
        for x in range(X):
            for y in range(Y):
                for z in range(Z):
                    U_curr[x][y][z] = 0.0

        for lpm in lpm_faces:
            for i in propagation_directions:
                for r, s in lpm.elements():
                    # Initialize ray radiance: boundary conditions on first iteration, else zero [S1]
                    L = boundary_conditions(r, s, i) if it == 1 else 0.0

                    for t in intersected_voxels(lpm, r, s, i):
                        voxel = volume[t]
                        delta_l = voxel.delta_l_in_voxel
                        omega_z = i.z
                        omega_s = i.component_along_plane_normal(lpm)

                        # Eq. (3): Update ray radiance [S1]
                        exp_term = math.exp(-delta_l * (voxel.Ka + voxel.Ks) / omega_z)
                        L = L * exp_term + U_prev[t] * (1.0 - exp_term) / (voxel.Ka + voxel.Ks)

                        # Eq. (4): Calculate scattering contribution [S1]
                        V_d = voxel.volume * solid_angle(storage_directions[0])
                        A_rs = lpm.element_area(r, s)
                        d = storage_directions[argmin_distance(i, storage_directions)]
                        F_val = F[i][d]
                        exp_s_term = math.exp(-delta_l * voxel.Ks)
                        R = (1.0 / V_d) * A_rs * F_val * L * (1.0 - exp_s_term) / omega_s

                        # Update buffers [S1]
                        U_curr[t] += R
                        I[t] += R

                        # Boundary handling: ray exits volume [S1]
                        if ray_exited_volume(lpm, r, s, i, t):
                            L = 0.0
                            r_new, s_new = opposite_point(lpm, r, s, i)
                            # Continue marching from new point (handled by loop continuation)

        # Swap buffers for next iteration [S1]
        U_prev, U_curr = U_curr, U_prev
        it += 1

    return I
```

**Edge Cases and Pitfalls:**
*   **Ray Effects and False Scattering:** Standard DOM suffers from these. LPM mitigates them by using fine spatial sampling (LPMs) for propagation and coarse sampling for storage [S1].
*   **Boundary Conditions:** Rays leaving the volume must be handled by generating new rays from the opposite side with zero radiance to maintain ray count [S1].

## Subtopics

### [Parallel and Scalable LPM](9951-parallel-and-scalable-lpm.md)
Parallel and Scalable LPM is a GPU-optimized implementation of Light Propagation Maps (LPM) that solves the synchronization bottlenecks inherent in naive parallel ray tracing within participating media. Unlike standard parallel approaches that assign one thread per ray and rely on expensive atomic operations or synchronization barriers to handle concurrent writes to the same voxel, this method decouples the process into a **propagation step** and a **collecting step**. By grouping rays by propagation direction and using temporary buffers, it guarantees that each voxel is traversed by only one ray at a time within a specific direction block, thereby eliminating race conditions and leveraging data locality for higher efficiency.

### [Streaming LPM](9952-streaming-lpm.md)
Streaming LPM is a memory-management technique for Light Propagation Maps (LPM) that overcomes GPU memory limitations by processing the volume in small, streamed sub-volumes rather than the entire grid at once. It sets itself apart from standard LPM by decoupling the memory footprint from the total volume size, allowing high-resolution volumes (e.g., $256^3$) to be rendered on hardware with limited VRAM, whereas standard LPM requires storing all radiance buffers for the entire volume simultaneously.

## Operating characteristics
*   **Time Complexity:** The method is iterative. Each iteration involves marching rays through the volume. The number of iterations depends on the scattering order and the termination threshold $\varepsilon$ [S1].
*   **Memory Complexity:** LPMs use high spatial resolution but coarse directional sampling for storage. This reduces memory compared to full DOM [S1]. Only two $U$ buffers are needed [S1].
*   **Performance:** LPM reduces artifacts like false scattering and ray effects compared to standard DOM [S1].
*   **Strengths:** Efficient handling of multiple scattering; reduced artifacts; lower memory usage than full DOM [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When multiple scattering is important and artifacts from standard DOM are unacceptable, and memory is a constraint compared to full DOM [S1].
*   **When not to use:** Not given in the sources.

## Background
Light Propagation Maps were introduced by Fattal as an iterative algorithm for computing multiple scattering in participating media [S1]. The method is based on the Discrete Ordinate Method (DOM) but improves upon it by using fine sampling for propagation and coarse sampling for storage [S1]. The technique is described in the paper "Light Propagation Maps on Parallel Graphics Architectures" [S1].

## Sources
[S1] Light Propagation Maps on Parallel Graphics Architectures, https://doi.org/10.2312/egpgv/egpgv12/081-088, fetched 2026-10-26
