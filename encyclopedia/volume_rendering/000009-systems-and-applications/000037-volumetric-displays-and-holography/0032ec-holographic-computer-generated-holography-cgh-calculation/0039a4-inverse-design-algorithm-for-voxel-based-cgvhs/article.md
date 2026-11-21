# Inverse Design Algorithm for Voxel-based CGVHs

Also called: Iterative Propagation Algorithm (IPA) for mode conversion

## Overview
The Inverse Design Algorithm for Voxel-based Computer-Generated Volume Holograms (CGVHs) is a method for designing holographic structures that convert $N$ mutually incoherent transverse input modes $\{u_n\}$ into $N$ target modes $\{v_n\}$ with a one-to-one mapping [S1, unofficial]. It optimizes the refractive index distribution within a 3D volume to achieve specific mode conversion efficiencies [S1, unofficial]. Unlike direct forward calculation methods, this algorithm uses an iterative approach to select voxel distributions that maximize the overlap between propagated fields and target modes [S1, unofficial]. It is distinguished from sibling methods by its reliance on a split-step Fourier Beam Propagation Method (BPM) and a local multi-objective criterion to select from a discrete set of pre-defined transverse voxel profiles [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   $N$: Number of input/target mode pairs [S1, unofficial].
*   $\{u_n\}_{n=1}^N$: Input mode field distributions defined at plane $z_{in} = z_1$ [S1, unofficial].
*   $\{v_n\}_{n=1}^N$: Target mode field distributions defined at plane $z_{out} = z_{K+1}$ [S1, unofficial].
*   $\lambda_n$: Wavelength associated with mode pair $n$ [S1, unofficial].
*   $n_n$: Bulk refractive index of the sample at wavelength $\lambda_n$ [S1, unofficial].
*   $D$: Number of available transverse voxel distributions $\{\Delta RI_d\}_{d=0}^D$ [S1, unofficial]. $\Delta RI_0$ is the null modification [S1, unofficial].
*   $I, J, K$: Grid dimensions of the sample volume [S1, unofficial].
*   $D_x, D_y, D_z$: Spatial extensions of sub-volumes $v_{ijk}$ [S1, unofficial].
*   $dx, dy, dz$: Simulation spatial resolutions [S1, unofficial].

**Outputs:**
*   Selected voxel distribution index $d^*_{ijk}$ for each sub-volume $v_{ijk}$ ($1 \le i \le I, 1 \le j \le J, 1 \le k \le K$) [S1, unofficial].

### Data Structures
The sample volume is virtually split into sub-volumes $v_{ijk}$ acting as placeholders for elementary voxel distributions [S1, unofficial].
*   **Grid:** 3D array of indices $(i, j, k)$ [S1, unofficial].
*   **Sub-volume $v_{ijk}$:** Defined by spatial extent $D_x \times D_y \times D_z$ [S1, unofficial].
*   **Voxel Distributions:** A discrete set of $D+1$ transverse refractive index change profiles $\Delta RI_d(x,y)$ [S1, unofficial]. These are relative to the bulk index $n_n$ [S1, unofficial].
*   **Field Storage:** Complex 2D arrays storing transverse field distributions $u_{ijk}^n$ and $v_{ijk}^n$ at each plane $z_k$ [S1, unofficial].

### Parameters
*   **Elementary Length:** $D_z = 10 \mu m$ (typical) [S1, unofficial].
*   **Simulation Resolution:** $dx = dy = 0.25 \mu m$, $dz = 0.5 \mu m$ (typical) [S1, unofficial].
*   **Coordinate System:** Rectangular $(O, x, y, z)$, with $z$ as the propagation direction [S1, unofficial].

### Formulas
*   **Phase multiplication model:** Crossing a layer of thickness $dz$ through a homogeneous medium $n_n$, followed by a local phase $\phi_{ijk}$, is modeled as:
    $$ \phi_{ijk}(\lambda_n) = \exp\left(i \frac{2\pi}{\lambda_n} \Delta RI_{ijk}^d(\lambda_n) dz\right), \quad 0 \le d \le D $$
    [S1, unofficial].
*   **Maximization criterion:** The squared $l_2$ distance minimization $O_{min} = \min \sum_{n=1}^N \|u_n - v_n\|_2^2$ is transformed into a maximization problem for lossless designs (where $\|u_n\|^2$ and $\|v_n\|^2$ are conserved):
    $$ O_{max} = \max_{\Delta RI_{ijk}^d} \sum_{n=1}^N \int_{z_{out}} \Re(v_n^* u_n) $$
    [S1, unofficial].
*   **Local selection criterion:** The optimal distribution for sub-volume $v_{ijk}$ maximizes:
    $$ O_{ijk}^{max} = \max_d \sum_{n=1}^N \int_{z_{k+1}} \Re\left((v_{ijk+1}^n)^* u_{ijk+1}^n(d)\right) $$
    where $u_{ijk+1}^n(d)$ is the input mode propagated from $z_k$ to $z_{k+1}$ through $\Delta RI_{ijk}^d$ [S1, unofficial].

### Algorithm Steps
1.  **Initialization:** Set all sub-volumes $v_{ijk}$ to the null distribution $\Delta RI_0$ [S1, unofficial].
2.  **Forward Propagation:** Propagate input modes $\{u_n\}$ from $z_1$ to $z_{K+1}$ using split-step Fourier BPM with resolutions $dx, dy, dz$ [S1, unofficial]. Cross sub-volumes $v_{ijk}$ layer by layer for $k$ from 1 to $K$ [S1, unofficial]. During propagation, save the transverse distributions $\{u_n\}$ before crossing each $v_{ijk}$, noting them $\{u_{ijk}^n\}$ for reference [S1, unofficial]. Note $\{z_k\}$ as the common transverse planes where distributions are saved [S1, unofficial].
3.  **Convergence Check:** Compute the final overlap criterion $O_{max}$ to monitor convergence [S1, unofficial]. If satisfactory or stagnant, exit and consider the current selection optimal [S1, unofficial].
4.  **Backward Propagation and Voxel Selection:** Iteratively backpropagate target modes $\{v_n\}$ from $z_{out}=z_{K+1}$ to $z_{in}=z_1$ [S1, unofficial]. For a given $k$, select the distribution $d$ that maximizes the local multi-objective criterion $O_{ijk}^{max}$ [S1, unofficial]. To find $d$, try all distributions $\{\Delta RI_{ijk}^d\}$ for each sub-volume $v_{ijk}$ ($k$ fixed) [S1, unofficial]. After selecting $\{\Delta RI_{ijk}^d\}$ for a given $k$, backpropagate target modes $\{v_{ijk+1}^n\}$ to plane $z_k$, and repeat iteratively until $k=1$ [S1, unofficial].
5.  **Iteration:** Repeat steps 2-4 [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def split_step_bpm(field_in, dz, n_bulk, wavelength):
    """Standard split-step Fourier BPM propagation through homogeneous medium."""
    k0 = 2 * np.pi / wavelength
    # Propagation phase in frequency domain
    kx = np.fft.fftfreq(field_in.shape[1], d=0.25e-6) * 2 * np.pi
    ky = np.fft.fftfreq(field_in.shape[0], d=0.25e-6) * 2 * np.pi
    KX, KY = np.meshgrid(kx, ky)
    kz = np.sqrt((n_bulk * k0)**2 - KX**2 - KY**2)
    phase = np.exp(1j * kz * dz)
    field_fft = np.fft.fft2(field_in)
    field_fft *= phase
    return np.fft.ifft2(field_fft)

def backpropagate_layer(field_in, dz, delta_ri, wavelength, n_b):
    """Reverse phase multiplication and reverse homogeneous propagation."""
    phase = np.exp(-1j * 2 * np.pi / wavelength * delta_ri * dz)
    field_prop = field_in * phase
    return split_step_bpm(field_prop, -dz, n_b, wavelength)

def inverse_design_CGVH(N, modes_u, modes_v, wavelengths, n_bulk, 
                        voxel_distributions, I, J, K, Dz, dx, dy, dz):
    """
    Inverse design algorithm for voxel-based CGVHs.
    """
    H, W = modes_u[0].shape
    selected_voxels = np.zeros((I, J, K), dtype=int)
    
    # u_fields[k][n] stores field at plane z_k for mode n
    u_fields = [np.zeros((I, J, N, H, W), dtype=complex) for _ in range(K + 1)]
    # v_fields[k][n] stores field at plane z_k for mode n
    v_fields = [np.zeros((I, J, N, H, W), dtype=complex) for _ in range(K + 1)]
    
    prev_overlap = 0.0
    max_iterations = 100
    
    for iteration in range(max_iterations):
        # Step 1: Forward Propagation
        for n in range(N):
            u_fields[0][:, :, n, :, :] = modes_u[n]
            
        for k in range(K):
            for i in range(I):
                for j in range(J):
                    d = selected_voxels[i, j, k]
                    delta_ri = voxel_distributions[d]
                    for n in range(N):
                        # Propagate from z_k to z_{k+1}
                        u_fields[k+1][i, j, n, :, :] = split_step_bpm(
                            u_fields[k][i, j, n, :, :], dz, n_bulk, wavelengths[n]
                        )
                        # Local phase multiplication
                        phase = np.exp(1j * 2 * np.pi / wavelengths[n] * delta_ri * dz)
                        u_fields[k+1][i, j, n, :, :] *= phase
        
        # Step 2: Convergence Check
        total_overlap = 0.0
        for n in range(N):
            overlap = np.sum(np.conj(modes_v[n]) * u_fields[K][:, :, n, :, :])
            total_overlap += np.real(overlap)
            
        if iteration > 0 and abs(total_overlap - prev_overlap) < 1e-6:
            break
        prev_overlap = total_overlap
        
        # Step 3: Backward Propagation and Voxel Selection
        for n in range(N):
            v_fields[K][:, :, n, :, :] = modes_v[n]
            
        for k in range(K - 1, -1, -1):
            for i in range(I):
                for j in range(J):
                    best_d = 0
                    best_criterion = -np.inf
                    u_in = u_fields[k][i, j, :, :, :]
                    v_target = v_fields[k+1][i, j, :, :, :]
                    
                    for d in range(len(voxel_distributions)):
                        delta_ri = voxel_distributions[d]
                        local_criterion = 0.0
                        for n in range(N):
                            u_next = split_step_bpm(u_in[n, :, :], dz, n_bulk, wavelengths[n])
                            phase = np.exp(1j * 2 * np.pi / wavelengths[n] * delta_ri * dz)
                            u_next *= phase
                            overlap = np.real(np.sum(np.conj(v_target[n, :, :]) * u_next))
                            local_criterion += overlap
                            
                        if local_criterion > best_criterion:
                            best_criterion = local_criterion
                            best_d = d
                    
                    selected_voxels[i, j, k] = best_d
                    delta_ri_best = voxel_distributions[best_d]
                    for n in range(N):
                        v_fields[k][i, j, n, :, :] = backpropagate_layer(
                            v_target[n, :, :], dz, delta_ri_best, wavelengths[n], n_bulk
                        )

    return selected_voxels
```

### Edge Cases and Pitfalls
*   **Precision:** The simulation resolution $dx, dy, dz$ must be much smaller than the feature size to ensure numerical accuracy [S1, unofficial]. Typical values are $dx=dy=0.25 \mu m$ and $dz=0.5 \mu m$ [S1, unofficial].
*   **Wavelength Dependence:** The voxel distributions $\Delta RI_d$ may be wavelength-dependent, requiring the algorithm to use the correct $\Delta RI_d(\lambda_n)$ for each mode $n$ [S1, unofficial].
*   **Convergence Monitoring:** The algorithm monitors stagnation of the overlap metric to determine termination [S1, unofficial].

## Operating characteristics
*   **Complexity:** Each iteration involves $K$ forward propagation steps and $K$ backward propagation steps [S1, unofficial]. In each step, for each sub-volume, all $D$ voxel distributions are tested [S1, unofficial]. The complexity per iteration is $O(K \cdot I \cdot J \cdot D \cdot N \cdot \text{BPM\_cost})$ [S1, unofficial].
*   **Memory:** Requires storing fields at all $K+1$ planes for all $N$ modes, plus the voxel distributions [S1, unofficial]. Memory complexity is $O(K \cdot I \cdot J \cdot N \cdot H \cdot W)$ [S1, unofficial].
*   **Strengths:** Allows for the design of complex 3D structures for multi-mode conversion using a discrete set of transverse voxel profiles [S1, unofficial].
*   **Limitations:** The solution depends on the initial guess (all null) and the set of available voxel distributions [S1, unofficial]. The maximization criterion is derived specifically for lossless designs where $\|u_n\|^2$ and $\|v_n\|^2$ are conserved [S1, unofficial].

## Background
The algorithm was proposed in the context of designing computer-generated volume holograms (CGVHs) in glass for mode conversion [S1, unofficial]. It is based on the Iterative Propagation Algorithm (IPA) [50], which maximizes the overlap integral between propagated fields and target fields [S1, unofficial].

## Sources
[S1] Design and manufacturing of efficient computer-generated volume holograms in glass, https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
