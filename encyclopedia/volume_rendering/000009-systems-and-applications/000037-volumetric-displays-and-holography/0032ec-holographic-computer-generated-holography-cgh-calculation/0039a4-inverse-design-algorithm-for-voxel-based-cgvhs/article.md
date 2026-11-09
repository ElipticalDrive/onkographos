# Inverse Design Algorithm for Voxel-based CGVHs

Also called: Iterative Propagation Algorithm (IPA) for mode conversion

## Overview
The Inverse Design Algorithm for Voxel-based Computer-Generated Volume Holograms (CGVHs) is a method for designing 3D refractive index structures that convert $N$ mutually incoherent transverse input modes $\{u_n\}$ into $N$ target modes $\{v_n\}$ with a one-to-one mapping [S1, unofficial]. It solves the problem of optimizing discrete voxel distributions in a finite volume to maximize mode conversion efficiency. It differs from direct summation methods by using an iterative forward-backward propagation scheme that selects optimal voxel types at each layer, rather than calculating continuous phase profiles [S1, unofficial]. It builds on [Holographic Computer-Generated Holography (CGH) Calculation](6884-holographic-computer-generated-holography-cgh-calculation.md).

## How it works
**Inputs:**
*   $N$ input mode fields $\{u_n(x,y, z_{in})\}_{n=1}^N$ defined at plane $z_{in} = z_1$.
*   $N$ target mode fields $\{v_n(x,y, z_{out})\}_{n=1}^N$ defined at plane $z_{out} = z_{K+1}$.
*   Wavelengths $\lambda_n$ associated with each mode pair $(u_n, v_n)$.
*   Bulk refractive index $n_n$ of the sample at wavelength $\lambda_n$.
*   A set of $D$ possible transverse voxel refractive index distributions $\{\Delta RI_d\}_{d=0}^D$, where $\Delta RI_0$ is the null modification (bulk index).
*   Sub-volume grid dimensions: $I \times J$ transverse cells, $K$ longitudinal layers.
*   Sub-volume extensions: $D_x, D_y, D_z$.
*   Simulation resolutions: $dx, dy, dz$.

**Outputs:**
*   A 3D map of selected voxel distributions $\{\Delta RI_{ijk}\}$ for each sub-volume $v_{ijk}$ ($1 \le i \le I, 1 \le j \le J, 1 \le k \le K$).

**Data Structures:**
*   **Grid:** A 3D array `voxel_selection[I][J][K]` storing integer indices $d \in \{0, \dots, D\}$.
*   **Field Buffers:** Complex 2D arrays `field_in[I][J]` and `field_out[I][J]` for storing transverse mode distributions at each longitudinal plane $z_k$.
*   **Voxel Library:** A lookup table or function `get_phase_mask(d, lambda, dz)` returning the phase factor for a given distribution index $d$.

**Parameters:**
*   $D_z = 10 \mu m$ (typical elementary length) [S1, unofficial].
*   $dx = dy = 0.25 \mu m, dz = 0.5 \mu m$ (typical simulation resolution) [S1, unofficial].
*   $K$: Number of longitudinal layers.

**Algorithm Steps:**

1.  **Initialization:** Set all `voxel_selection[i][j][k] = 0` (null distribution).
2.  **Forward Propagation:**
    *   For each mode $n$ from $1$ to $N$:
        *   Initialize current field $u^{(1)}_n = u_n$ at $z_1$.
        *   For $k$ from $1$ to $K$:
            *   Store current field as $u^{ijk}_n$ for all $i,j$.
            *   Get selected distribution index $d = \text{voxel\_selection}[i][j][k]$.
            *   Propagate field through sub-volume $v_{ijk}$ using Split-Step Fourier Beam Propagation Method (BPM):
                *   Apply homogeneous propagation of thickness $dz$ with index $n_n$.
                *   Apply local phase mask: $\phi_{ijk}(\lambda_n) = \exp\left(i \frac{2\pi}{\lambda_n} \Delta RI_d(\lambda_n) dz\right)$.
            *   Result is $u^{ijk+1}_n$ at plane $z_{k+1}$.
        *   Store final field $u^{K+1}_n$ at $z_{out}$.
3.  **Convergence Check:**
    *   Compute global overlap criterion:
        $$ O_{max} = \sum_{n=1}^N \int_{z_{out}} \Re(v_n^* u^{K+1}_n) \, dx dy $$
    *   If $O_{max}$ has converged (change below threshold) or stagnated, exit and return `voxel_selection`.
4.  **Backward Propagation and Voxel Selection:**
    *   For each mode $n$ from $1$ to $N$:
        *   Initialize target field $v^{K+1}_n = v_n$ at $z_{out}$.
    *   For $k$ from $K$ down to $1$:
        *   For each transverse cell $(i,j)$:
            *   Retrieve stored forward field $u^{ijk}_n$ for all $n$.
            *   Retrieve current backward field $v^{ijk+1}_n$ for all $n$.
            *   Find optimal distribution index $d^*$ that maximizes the local criterion:
                $$ O^{ijk}_{max} = \max_{d \in \{0,\dots,D\}} \sum_{n=1}^N \int_{z_{k+1}} \Re\left( (v^{ijk+1}_n)^* u^{ijk+1}_n(d) \right) \, dx dy $$
                where $u^{ijk+1}_n(d)$ is the field $u^{ijk}_n$ propagated through $v_{ijk}$ using distribution $d$.
            *   Set `voxel_selection[i][j][k] = d^*`.
        *   Backpropagate the fields $v^{ijk+1}_n$ from $z_{k+1}$ to $z_k$ through the *selected* voxels (using the inverse of the forward propagation step) to obtain $v^{ijk}_n$.
5.  **Repeat:** Go to Step 2.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_design_cgvh(N, K, I, J, D, u_in, v_target, lambda_n, n_bulk, dz, dx, dy):
    # Initialize voxel selection to null (0)
    voxel_selection = zeros((I, J, K), dtype=int)
    
    # Precompute phase masks for all distributions and wavelengths
    # phi[d][n] is a 2D complex array representing the phase mask for distribution d at wavelength lambda_n
    phi = compute_phase_masks(D, lambda_n, dz) 
    
    for iteration in range(MAX_ITERATIONS):
        # Step 1: Forward Propagation
        # Store fields at each plane k
        u_stored = zeros((K+1, N, I, J), dtype=complex)
        u_current = copy(u_in) # Shape (N, I, J)
        
        for k in range(1, K+1):
            u_stored[k] = copy(u_current)
            
            # Propagate through each sub-volume v_ijk
            u_next = zeros((N, I, J), dtype=complex)
            for n in range(N):
                for i in range(I):
                    for j in range(J):
                        d = voxel_selection[i, j, k-1] # 0-indexed array, k is 1-indexed
                        # Homogeneous propagation (simplified as identity for BPM step if dz is small, 
                        # but typically involves FFT-based propagation)
                        u_prop = propagate_homogeneous(u_current[n, i, j], n_bulk[n], dz, dx, dy)
                        # Apply local phase mask
                        u_prop = u_prop * phi[d, n]
                        u_next[n, i, j] = u_prop
            u_current = u_next
            
        u_stored[K+1] = copy(u_current)
        
        # Step 2: Convergence Check
        O_max = 0.0
        for n in range(N):
            # Integral of Re(v* u) over the plane
            O_max += integral_re_conjugate_product(v_target[n], u_current[n])
            
        if check_convergence(O_max, previous_O_max):
            return voxel_selection
        previous_O_max = O_max
        
        # Step 3 & 4: Backward Propagation and Selection
        v_current = copy(v_target) # Shape (N, I, J)
        
        for k in range(K, 0, -1):
            v_next = zeros((N, I, J), dtype=complex)
            
            for i in range(I):
                for j in range(J):
                    # Find best d for this sub-volume
                    best_d = 0
                    best_O_local = -infinity
                    
                    for d in range(D+1):
                        O_local = 0.0
                        for n in range(N):
                            # Propagate forward stored field u_stored[k][n][i][j] through distribution d
                            u_prop = propagate_homogeneous(u_stored[k, n, i, j], n_bulk[n], dz, dx, dy)
                            u_prop = u_prop * phi[d, n]
                            # Compute overlap with current backward field v_current[n][i][j]
                            # Note: v_current here is v^{ijk+1}_n
                            O_local += integral_re_conjugate_product(v_current[n, i, j], u_prop)
                            
                        if O_local > best_O_local:
                            best_O_local = O_local
                            best_d = d
                            
                    voxel_selection[i, j, k-1] = best_d
                    
                    # Backpropagate v_current[n][i][j] through the selected voxel to get v^{ijk}_n
                    # This is the inverse of the forward step
                    # Inverse phase mask
                    u_back = v_current[n, i, j] / phi[best_d, n]
                    # Inverse homogeneous propagation
                    v_next[n, i, j] = propagate_homogeneous_inverse(u_back, n_bulk[n], dz, dx, dy)
            
            v_current = v_next
            
    return voxel_selection
```

**Edge Cases and Pitfalls:**
*   **Lossy Systems:** The derivation of $O_{max}$ assumes lossless design where $\|u_n\|^2$ and $\|v_n\|^2$ are conserved. If losses are significant, the criterion may not be optimal [S1, unofficial].
*   **Discretization:** The voxel distributions are discrete. If the optimal continuous phase profile is not well-approximated by the set $\{\Delta RI_d\}$, performance will degrade [S1, unofficial].
*   **Convergence:** The algorithm is iterative and may get stuck in local minima. Monitoring $O_{max}$ is critical [S1, unofficial].
*   **Resolution:** $dx, dy, dz$ must be small enough to accurately resolve the phase variations introduced by $\Delta RI_d$. Typical values are $dx=dy=0.25 \mu m, dz=0.5 \mu m$ [S1, unofficial].

## Operating characteristics
*   **Complexity:** The algorithm requires $K$ forward and $K$ backward passes. Each pass involves $N \times I \times J$ voxel evaluations. For each voxel, $D$ candidate distributions are tested. Thus, complexity is roughly $O(K \cdot N \cdot I \cdot J \cdot D)$. The BPM propagation itself is $O(M \log M)$ per mode per layer, where $M$ is the number of pixels in a transverse plane. Total complexity is $O(K \cdot N \cdot I \cdot J \cdot D \cdot M \log M)$.
*   **Memory:** Requires storing fields at each longitudinal plane for all modes: $O(K \cdot N \cdot I \cdot J)$.
*   **Strengths:** Handles multiple incoherent modes simultaneously. Optimizes discrete voxel structures suitable for fabrication (e.g., femtosecond laser writing).
*   **Limitations:** Computationally expensive due to the search over $D$ distributions at each voxel. Assumes a specific set of available voxel types.
*   **When to use:** When designing CGVHs for fabrication with discrete voxel capabilities, especially for mode conversion tasks.
*   **When not to use:** For continuous phase-only holograms where a simpler algorithm like Gerchberg-Saxton is sufficient, or when real-time calculation is required.

## Background
The algorithm was proposed in the context of designing efficient computer-generated volume holograms in glass for mode conversion [S1, unofficial]. It relies on the Split-Step Fourier Beam Propagation Method (BPM) for accurate modeling of light propagation in inhomogeneous media. The method is also known as the Iterative Propagation Algorithm (IPA) for mode conversion [S1, unofficial]. It is particularly relevant for applications involving femtosecond laser writing of ferroelectric nanodomains or refractive index modifications in glass [S1, unofficial].

## Sources
[S1] Design and manufacturing of efficient computer-generated volume holograms in glass, https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
