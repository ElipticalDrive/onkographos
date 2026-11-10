# Inverse Design Algorithm for Voxel-based CGVHs

Also called: Iterative Propagation Algorithm (IPA) for mode conversion

## Overview
The Inverse Design Algorithm for Voxel-based Computer-Generated Volume Holograms (CGVHs) is a method for designing volumetric optical elements that convert $N$ mutually incoherent transverse input modes $\{u_n\}$ into $N$ target modes $\{v_n\}$ with a one-to-one mapping [S1, unofficial]. It solves the problem of determining the optimal 3D refractive index distribution within a finite volume to achieve specific mode conversion efficiencies. Unlike direct forward calculation methods, this algorithm uses an iterative optimization loop that alternates between forward propagation and backpropagation to select discrete voxel distributions that maximize the overlap between the propagated fields and the target modes [S1, unofficial]. It is distinct from standard phase-only CGH generation because it operates on a volumetric grid of selectable refractive index profiles rather than a 2D phase mask, and it specifically handles multi-wavelength or multi-mode incoherent superposition [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   $N$: Number of input/target mode pairs.
*   $\{u_n\}_{n=1}^N$: Input complex field distributions at plane $z_{in} = z_1$.
*   $\{v_n\}_{n=1}^N$: Target complex field distributions at plane $z_{out} = z_{K+1}$.
*   $\lambda_n$: Wavelength associated with mode $n$ (each pair may have a distinct wavelength) [S1, unofficial].
*   $n_n$: Bulk refractive index of the sample at wavelength $\lambda_n$ [S1, unofficial].
*   $D$: Set of $D$ available transverse voxel distributions $\{\Delta RI_d\}_{d=1}^D$, representing relative refractive index changes compared to the bulk index. $\Delta RI_0$ denotes no modification [S1, unofficial].
*   Grid dimensions: $I \times J \times K$ sub-volumes $v_{ijk}$ with extensions $D_x, D_y, D_z$ [S1, unofficial].
*   Simulation resolution: $dx, dy, dz$ [S1, unofficial].

**Outputs:**
*   Selected voxel distribution $\Delta RI_{ijk}^*$ for each sub-volume $v_{ijk}$, defining the final hologram structure [S1, unofficial].

### Data Structures
*   **Grid:** A 3D array of dimensions $I \times J \times K$. Each cell $(i,j,k)$ stores an integer index $d \in \{0, \dots, D\}$ representing the selected refractive index profile.
*   **Field Arrays:** Complex-valued 2D arrays of size $(N_x \times N_y)$ for storing field distributions $u$ and $v$ at each transverse plane $z_k$. $N_x, N_y$ correspond to the spatial resolution $dx, dy$.
*   **Voxel Library:** A lookup table or function mapping index $d$ to the complex phase profile $\phi_{ijk}^d(\lambda_n)$ or the refractive index change $\Delta RI_{ijk}^d(\lambda_n)$ [S1, unofficial].

### Algorithm Steps
The algorithm consists of an iterative loop comprising a forward pass, a convergence check, and a backward pass with voxel selection.

1.  **Initialization:** Set all sub-volumes $v_{ijk}$ to the null distribution $\Delta RI_0$. Initialize the current field distributions $u_{ijk}^n$ at plane $z_1$ to the input modes $u_n$.
2.  **Forward Propagation:** Propagate the input modes $\{u_n\}$ from $z_1$ to $z_{K+1}$ using the Split-Step Fourier Beam Propagation Method (BPM).
    *   For each layer $k$ from 1 to $K$:
        *   Save the current field distributions as $u_{ijk}^n$ at plane $z_k$.
        *   Apply the phase shift corresponding to the selected voxel distribution $\Delta RI_{ijk}^d$ in sub-volume $v_{ijk}$.
        *   Propagate through the homogeneous medium of thickness $dz$ using the bulk refractive index $n_n$.
    *   Store the final fields at $z_{K+1}$ as $u_{out}^n$.
3.  **Convergence Check:** Compute the global overlap criterion $O_{max}$ (defined below) using the final fields $u_{out}^n$ and targets $v_n$. If the value has converged or stagnated, terminate and output the current voxel selection.
4.  **Backward Propagation and Voxel Selection:**
    *   Initialize the target fields at $z_{K+1}$ as $v_{out}^n = v_n$.
    *   For each layer $k$ from $K$ down to 1:
        *   For each sub-volume $v_{ijk}$ in layer $k$:
            *   Retrieve the stored forward-propagated field $u_{ijk}^n$ from plane $z_k$.
            *   For each candidate distribution $d \in \{0, \dots, D\}$:
                *   Propagate $u_{ijk}^n$ from $z_k$ to $z_{k+1}$ through the hypothetical voxel $\Delta RI_{ijk}^d$ to obtain $u_{ijk+1}^n(d)$.
                *   Compute the local criterion $O_{ijk}^{max}(d)$:
                    $$ O_{ijk}^{max}(d) = \sum_{n=1}^{N} \int_{z_{k+1}} \Re\left( (v_{ijk+1}^n)^* u_{ijk+1}^n(d) \right) dx dy $$
                    where $v_{ijk+1}^n$ is the target mode distribution at $z_{k+1}$ (initially the global target, updated after selection in previous steps of the backward pass).
            *   Select the distribution $d^*$ that maximizes $O_{ijk}^{max}(d)$.
            *   Update the field at $z_{k+1}$ for the next iteration step by backpropagating the target $v_{ijk+1}^n$ through the *selected* voxel $\Delta RI_{ijk}^{d^*}$ to plane $z_k$. This updated field becomes the new $v_{ijk}^n$ for the next sub-volume or layer processing.
        *   After processing all sub-volumes in layer $k$, the fields at $z_k$ are updated.
    *   Repeat until $k=1$.
5.  **Iteration:** Return to Step 2 with the newly selected voxel distributions.

### Formulas
**Phase Multiplication Model:**
When a mode $u_n$ crosses a layer of thickness $dz$ containing a voxel with relative refractive index change $\Delta RI_{ijk}^d(\lambda_n)$, the field is multiplied by a local phase factor [S1, unofficial]:
$$ \phi_{ijk}^d(\lambda_n) = \exp\left( i \frac{2\pi}{\lambda_n} \Delta RI_{ijk}^d(\lambda_n) dz \right) $$
The propagation through the layer is modeled as propagation through homogeneous medium $n_n$ followed by multiplication by $\phi_{ijk}^d$ [S1, unofficial].

**Global Error Metric (Minimization):**
$$ O_{min} = \min \sum_{n=1}^{N} \| u_n - v_n \|_2^2 = \min \left\{ \sum_{n=1}^{N} \int_{z_{out}} |u_n - v_n|^2 \right\} $$
[S1, unofficial]

**Maximization Criterion (Lossless Design):**
Assuming energy conservation ($\|u_n\|^2$ and $\|v_n\|^2$ are constant), the minimization is equivalent to maximization of the real part of the overlap integral [S1, unofficial]:
$$ O_{max} = \max_{\{\Delta RI_{ijk}^d\}} \left\{ \sum_{n=1}^{N} \int_{z_{out}} \Re(v_n^* u_n) \right\} $$
[S1, unofficial]

**Local Selection Criterion:**
For a specific sub-volume $v_{ijk}$, the best distribution $d$ is chosen to maximize the local contribution to the global overlap [S1, unofficial]:
$$ O_{ijk}^{max} = \max_{d} \left\{ \sum_{n=1}^{N} \int_{z_{k+1}} \Re\left( (v_{ijk+1}^n)^* u_{ijk+1}^n(d) \right) \right\} $$
where $u_{ijk+1}^n(d)$ is the field at $z_{k+1}$ resulting from propagating the stored field $u_{ijk}^n$ from $z_k$ through $\Delta RI_{ijk}^d$ [S1, unofficial].

### Parameters
*   **Elementary Length ($D_z$):** Typically $10 \mu m$ [S1, unofficial].
*   **Simulation Resolution:** $dx = dy = 0.25 \mu m$, $dz = 0.5 \mu m$ [S1, unofficial].
*   **Coordinate System:** Rectangular $(O, x, y, z)$, with $z$ as the propagation direction [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_design_cgvh(input_modes, target_modes, wavelengths, bulk_indices, 
                        voxel_library, grid_dims, resolution, max_iterations):
    """
    input_modes: List of N complex 2D arrays (Nx, Ny) at z=0
    target_modes: List of N complex 2D arrays (Nx, Ny) at z=K+1
    wavelengths: List of N floats
    bulk_indices: List of N floats
    voxel_library: Function d -> complex 2D array (Nx, Ny) representing phase mask for distribution d
    grid_dims: Tuple (I, J, K)
    resolution: Tuple (dx, dy, dz)
    """
    I, J, K = grid_dims
    N = len(input_modes)
    
    # Initialize voxel selection: all null (index 0)
    # voxel_selection[i, j, k] = 0
    voxel_selection = np.zeros((I, J, K), dtype=int)
    
    # Storage for forward fields: fields[k][n] stores u at plane z_k for mode n
    # k ranges from 0 to K (inclusive), corresponding to z_1 to z_{K+1}
    forward_fields = [None] * (K + 1)
    
    for iteration in range(max_iterations):
        # Step 1: Forward Propagation
        current_fields = list(input_modes)
        forward_fields[0] = list(current_fields)
        
        for k in range(K):
            # current_fields holds u at plane z_k
            next_fields = []
            for n in range(N):
                u_n = current_fields[n]
                # Propagate through homogeneous medium dz
                u_n = propagate_homogeneous(u_n, resolution[2], bulk_indices[n])
                
                # Apply voxel phase masks for the current layer k
                # u_n is a single field, but we need to apply different voxels to different spatial regions
                # Assuming u_n is on a grid covering the whole cross-section
                u_layered = np.zeros_like(u_n)
                for i in range(I):
                    for j in range(J):
                        d = voxel_selection[i, j, k]
                        phase_mask = voxel_library[d] # Shape (Nx, Ny) or padded to sub-region
                        # Apply mask to the corresponding sub-region of u_n
                        # Note: In practice, u_n is split into sub-volumes, processed, and reassembled
                        # For simplicity, assuming voxel_library returns a mask for the whole grid 
                        # or we slice u_n appropriately.
                        # Correct approach: u_n is the field at z_k. We apply the phase of the voxel 
                        # at (i,j,k) to the corresponding spatial patch.
                        patch = get_patch(u_n, i, j, I, J)
                        patch *= phase_mask
                        u_layered = set_patch(u_layered, patch, i, j, I, J)
                
                # Propagate to next plane z_{k+1}
                u_next = propagate_homogeneous(u_layered, resolution[2], bulk_indices[n])
                next_fields.append(u_next)
            
            current_fields = next_fields
            forward_fields[k+1] = list(current_fields)
            
        # Step 2: Convergence Check
        # Compute global overlap O_max
        final_overlap = 0.0
        for n in range(N):
            u_final = current_fields[n]
            v_target = target_modes[n]
            overlap = np.real(np.sum(np.conj(v_target) * u_final) * resolution[0] * resolution[1])
            final_overlap += overlap
            
        if is_converged(final_overlap, previous_overlap):
            break
        previous_overlap = final_overlap
        
        # Step 3: Backward Propagation and Voxel Selection
        # Start with target modes at z_{K+1}
        current_targets = list(target_modes)
        
        for k in range(K-1, -1, -1):
            # We are at plane z_{k+1}, want to select voxels for layer k to optimize overlap at z_{k+1}
            # and backpropagate to z_k.
            
            # Prepare fields at z_k for backpropagation
            # The fields at z_k are the forward fields stored in forward_fields[k]
            # But we need to backpropagate the TARGETS from z_{k+1} through the SELECTED voxels to z_k
            
            # First, select best voxels for layer k
            for i in range(I):
                for j in range(J):
                    best_d = 0
                    best_score = -np.inf
                    
                    # Get forward field at z_k for this sub-volume
                    # We need to extract the patch from forward_fields[k][n]
                    # And propagate it through each candidate d to z_{k+1}
                    
                    for n in range(N):
                        u_patch_k = get_patch(forward_fields[k][n], i, j, I, J)
                        v_patch_k1 = get_patch(current_targets[n], i, j, I, J)
                        
                        # Try all distributions
                        for d in range(len(voxel_library)):
                            # Propagate u_patch_k through homogeneous dz
                            u_prop = propagate_homogeneous(u_patch_k, resolution[2], bulk_indices[n])
                            # Apply phase mask d
                            phase_mask = voxel_library[d]
                            u_prop *= phase_mask
                            # Propagate to z_{k+1}
                            u_prop_k1 = propagate_homogeneous(u_prop, resolution[2], bulk_indices[n])
                            
                            # Compute local criterion contribution for this mode
                            score_n = np.real(np.sum(np.conj(v_patch_k1) * u_prop_k1) * resolution[0] * resolution[1])
                            
                            # Accumulate score for all modes (simplified: assuming we pick d based on sum)
                            # Note: The criterion is sum over n. We must compute the sum for each d.
                            pass 
                    
                    # Recompute sum over n for each d properly
                    for d in range(len(voxel_library)):
                        total_score = 0.0
                        for n in range(N):
                            u_patch_k = get_patch(forward_fields[k][n], i, j, I, J)
                            v_patch_k1 = get_patch(current_targets[n], i, j, I, J)
                            
                            u_prop = propagate_homogeneous(u_patch_k, resolution[2], bulk_indices[n])
                            phase_mask = voxel_library[d]
                            u_prop *= phase_mask
                            u_prop_k1 = propagate_homogeneous(u_prop, resolution[2], bulk_indices[n])
                            
                            total_score += np.real(np.sum(np.conj(v_patch_k1) * u_prop_k1) * resolution[0] * resolution[1])
                            
                        if total_score > best_score:
                            best_score = total_score
                            best_d = d
                    
                    voxel_selection[i, j, k] = best_d
            
            # Now backpropagate targets through the SELECTED voxels
            next_targets = []
            for n in range(N):
                v_k1 = current_targets[n]
                v_k = np.zeros_like(v_k1)
                
                for i in range(I):
                    for j in range(J):
                        d = voxel_selection[i, j, k]
                        v_patch_k1 = get_patch(v_k1, i, j, I, J)
                        
                        # Backpropagate through homogeneous dz
                        v_prop = backpropagate_homogeneous(v_patch_k1, resolution[2], bulk_indices[n])
                        # Apply inverse phase mask of selected voxel
                        phase_mask = voxel_library[d]
                        v_prop *= np.conj(phase_mask) # Inverse phase
                        # Backpropagate to z_k
                        v_patch_k = backpropagate_homogeneous(v_prop, resolution[2], bulk_indices[n])
                        
                        v_k = set_patch(v_k, v_patch_k, i, j, I, J)
                
                next_targets.append(v_k)
            
            current_targets = next_targets

    return voxel_selection
```

### Edge Cases and Pitfalls
*   **Lossy Systems:** The derivation of the maximization criterion $O_{max}$ assumes lossless design where $\|u_n\|^2$ and $\|v_n\|^2$ are conserved. If the medium is lossy, the equivalence between minimization of squared error and maximization of overlap breaks down, and the algorithm may not converge to the global optimum [S1, unofficial].
*   **Discrete Voxel Library:** The algorithm is restricted to the pre-defined set of $D$ transverse distributions. If the optimal refractive index profile is not well-approximated by these distributions, the efficiency will be limited [S1, unofficial].
*   **Resolution vs. Voxel Size:** The simulation resolution $dz$ ($0.5 \mu m$) is finer than the elementary voxel length $D_z$ ($10 \mu m$). The BPM must correctly handle the phase accumulation over the discrete steps within each voxel [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The algorithm is iterative. Each iteration involves $K$ forward propagation steps and $K$ backward propagation steps. The voxel selection step requires evaluating $D$ candidates for each of the $I \times J$ sub-volumes in each layer. Thus, the cost per iteration is proportional to $K \cdot I \cdot J \cdot D \cdot (\text{cost of propagation})$. Propagation itself is $O(N_x N_y \log(N_x N_y))$ per mode per layer using FFTs. Total complexity per iteration: $O(K \cdot I \cdot J \cdot D \cdot N \cdot N_x N_y \log(N_x N_y))$ [S1, unofficial].
*   **Memory Complexity:** Requires storing the grid $I \times J \times K$, the forward fields at each plane $K \times N \times N_x \times N_y$, and the target fields. Memory scales linearly with $K$ and $N$ [S1, unofficial].
*   **Strengths:** Handles multi-mode and multi-wavelength conversion simultaneously. Provides a physically realizable design based on available fabrication constraints (discrete voxels).
*   **Limitations:** Computationally expensive due to the exhaustive search over $D$ candidates at each voxel. Convergence is not guaranteed to be global. Limited by the resolution and diversity of the voxel library [S1, unofficial].

## Background
The Inverse Design Algorithm for Voxel-based CGVHs was developed in the context of designing efficient computer-generated volume holograms in glass for mode sorting and optical communications [S1, unofficial]. It builds upon the Iterative Propagation Algorithm (IPA) concept, which utilizes the conservation of overlap integrals in lossless systems to iteratively optimize optical elements [S1, unofficial]. The method was presented in research focusing on the design and manufacturing of these holograms, leveraging femtosecond laser writing capabilities to create the discrete voxel structures [S1, unofficial].

## Sources
[S1] Design and manufacturing of efficient computer-generated volume holograms in glass, https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
