# Inverse Design Algorithm for Voxel-based CGVHs

Also called: Iterative Propagation Algorithm (IPA) for mode conversion

## Overview
The Inverse Design Algorithm for Voxel-based Computer-Generated Volume Holograms (CGVHs) is a method for designing finite-extent holographic samples that convert $N$ mutually incoherent transverse input modes $\{u_n\}$ into $N$ target modes $\{v_n\}$ with a one-to-one mapping [S1, unofficial]. It solves the problem of determining the optimal 3D distribution of refractive index voxels within a bulk medium to achieve specific mode conversions across multiple wavelengths. It sets itself apart from sibling methods by using an iterative forward-backward propagation scheme that maximizes local overlap integrals at each voxel layer, rather than relying on direct diffraction calculations or phase-only modulation [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $N$ input mode fields $\{u_n\}$ defined at plane $z_{in} = z_1$.
    *   $N$ target mode fields $\{v_n\}$ defined at plane $z_{out} = z_{K+1}$.
    *   Wavelengths $\{\lambda_n\}$ associated with each mode pair $(u_n, v_n)$.
    *   Bulk refractive index $n_n$ at each wavelength.
    *   A library of $D$ possible transverse voxel refractive index distributions $\{\Delta RI_d\}_{1 \le d \le D}$, where $\Delta RI_0$ is the null distribution [S1, unofficial].
    *   Sub-volume grid dimensions $I, J, K$ and elementary voxel size $D_z$ (with $D_x, D_y$ defining the transverse extent of sub-volumes) [S1, unofficial].
*   **Outputs:**
    *   A 3D map of selected voxel distributions $\{\Delta RI_{ijk}\}$ for each sub-volume $v_{ijk}$ ($1 \le i \le I, 1 \le j \le J, 1 \le k \le K$) [S1, unofficial].

**Data Structures**
*   **Grid:** The sample volume is virtually split into sub-volumes $v_{ijk}$. The index $k$ corresponds to the longitudinal layer (propagation direction $z$).
*   **Fields:** Complex field arrays $u_{ijk}^n$ and $v_{ijk}^n$ representing the transverse distributions of the $n$-th mode at the boundaries of sub-volume $v_{ijk}$.
*   **Voxel Library:** A discrete set of 2D refractive index change patterns $\Delta RI_d(x,y)$ [S1, unofficial].

**Parameters**
*   **Elementary Length:** $D_z = 10 \mu m$ [S1, unofficial].
*   **Simulation Resolution:** $dx = dy = 0.25 \mu m$, $dz = 0.5 \mu m$ [S1, unofficial].
*   **Propagation Method:** Split-step Fourier Beam Propagation Method (BPM) [S1, unofficial].

**Formulas**
1.  **Phase Multiplication Model:** Crossing a layer of thickness $dz$ through a homogeneous medium $n_n$ with a local phase shift induced by voxel distribution $d$ is modeled as:
    $$ \phi_{ijk}(\lambda_n) = \exp\left(i \frac{2\pi}{\lambda_n} \Delta RI_{ijk}^d(\lambda_n) dz\right), \quad 0 \le d \le D $$
    [S1, unofficial]

2.  **Global Error Metric (Minimization):**
    $$ O_{min} = \min \sum_{n=1}^{N} \|u_n - v_n\|_2^2 = \min \left\{ \sum_{n=1}^{N} \int_{z_{out}} |u_n - v_n|^2 \right\} $$
    [S1, unofficial]

3.  **Global Overlap Criterion (Maximization):** Assuming lossless design (conservation of $\|u_n\|_2^2$ and $\|v_n\|_2^2$), the minimization is equivalent to:
    $$ O_{max} = \max_{\Delta RI_{ijk}^d} \left\{ \sum_{n=1}^{N} \int_{z_{out}} \Re(v_n^* u_n) \right\} $$
    [S1, unofficial]

4.  **Local Selection Criterion:** For a specific sub-volume $v_{ijk}$, the optimal distribution index $d$ maximizes the local overlap between the backpropagated target mode and the forward-propagated input mode:
    $$ O_{ijk}^{max} = \max_d \left\{ \sum_{n=1}^{N} \int_{z_{k+1}} \Re\left( (v_{ijk+1}^n)^* u_{ijk+1}^n(d) \right) \right\} $$
    where $u_{ijk+1}^n(d)$ is the input mode propagated from plane $z_k$ to $z_{k+1}$ through the candidate distribution $\Delta RI_{ijk}^d$ [S1, unofficial].

**Algorithm Steps**
1.  **Initialization:** Set all sub-volumes $v_{ijk}$ to the null distribution $\Delta RI_0$.
2.  **Forward Propagation:** Propagate input modes $\{u_n\}$ from $z_1$ to $z_{K+1}$ using BPM. Store the transverse field distributions $\{u_{ijk}^n\}$ at each plane $z_k$ before crossing sub-volume $v_{ijk}$ [S1, unofficial].
3.  **Convergence Check:** Compute the global overlap criterion $O_{max}$ (Eq. 6). If satisfactory or stagnant, exit. Otherwise, proceed [S1, unofficial].
4.  **Backward Iteration (Voxel Selection):**
    *   Start from $k=K$ down to $1$.
    *   For each sub-volume $v_{ijk}$, iterate through all candidate distributions $d \in \{1, \dots, D\}$.
    *   For each $d$, propagate the stored forward field $u_{ijk}^n$ from $z_k$ to $z_{k+1}$ using the phase model (Eq. 4) to get $u_{ijk+1}^n(d)$.
    *   Calculate the local criterion $O_{ijk}^{max}$ (Eq. 7) for all $d$.
    *   Select the $d$ that maximizes $O_{ijk}^{max}$ and update the voxel in $v_{ijk}$ [S1, unofficial].
    *   Backpropagate the target modes $\{v_n\}$ from $z_{k+1}$ to $z_k$ through the *selected* voxel distribution to update the fields for the next iteration step ($k-1$) [S1, unofficial].
5.  **Repeat:** Return to Step 2 with the updated voxel map [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_design_cgvh(input_modes, target_modes, wavelengths, bulk_indices, 
                        voxel_library, grid_dims, dz_elem, dx, dy, dz_sim, 
                        max_iterations, convergence_threshold):
    """
    input_modes: List of N complex 2D arrays (Nx, Ny) at z_in
    target_modes: List of N complex 2D arrays (Nx, Ny) at z_out
    wavelengths: List of N floats
    bulk_indices: List of N floats
    voxel_library: List of D complex 2D arrays representing Delta RI patterns
    grid_dims: Tuple (I, J, K)
    dz_elem: Elementary length (e.g., 10e-6)
    dx, dy, dz_sim: Simulation resolutions
    """
    N = len(input_modes)
    D = len(voxel_library)
    I, J, K = grid_dims
    
    # Initialize voxel map with null distribution (index 0)
    # voxel_map[i, j, k] stores the index d of the selected voxel
    voxel_map = np.zeros((I, J, K), dtype=int)
    
    # Precompute phase factors for all voxels and wavelengths
    # phi[d, n] is a 2D array of phase shifts for voxel d and wavelength n
    # Assuming voxel_library contains Delta RI values
    phase_factors = np.zeros((D, N, Nx, Ny), dtype=complex)
    for d in range(D):
        for n in range(N):
            # Eq 4: phi = exp(i * 2pi/lambda * Delta_RI * dz_sim)
            # Note: voxel_library might need scaling to match dz_sim if defined per unit length
            delta_ri = voxel_library[d] 
            phase_factors[d, n] = np.exp(1j * 2 * np.pi / wavelengths[n] * delta_ri * dz_sim)

    for iteration in range(max_iterations):
        # Step 1: Forward Propagation
        # Current fields at z_k
        current_fields = [np.copy(u) for u in input_modes]
        stored_fields = [] # List of lists: stored_fields[k][n]
        
        # Propagate layer by layer from k=1 to K
        for k in range(K):
            stored_fields.append([np.copy(u) for u in current_fields])
            
            # Get current voxel selection for this layer
            # The voxel selection is uniform across the transverse plane for the sub-volume?
            # Or per (i,j)? The text says "sub-volume v_ijk". 
            # Assuming we select one d for the whole sub-volume v_ijk or iterate i,j?
            # Text: "select the distribution d that maximizes... for each sub-volume v_ijk"
            # This implies d can vary with i,j.
            
            # However, BPM is usually done on the whole plane. 
            # If d varies with i,j, we must apply the phase mask locally.
            
            # Create phase mask for this layer k
            # We need to know the selected d for each (i,j) in this layer
            # But we are in the forward pass. The selection happens in the backward pass.
            # So in forward pass, we use the CURRENT voxel_map.
            
            phase_mask = np.zeros((Nx, Ny), dtype=complex)
            # Map grid indices to simulation grid indices
            # This requires interpolation/resampling logic not fully detailed in source
            # [G] Assuming a simple block mapping for pseudocode clarity
            for i in range(I):
                for j in range(J):
                    d_idx = voxel_map[i, j, k]
                    # Apply phase factor for this voxel to the corresponding spatial region
                    # [G] Placeholder for spatial mapping logic
                    pass 
            
            # Propagate current_fields through this layer
            # Split-step BPM: Propagate in homogeneous medium, then apply phase
            # [G] Simplified to just applying phase for the "crossing" model described
            # Real BPM involves FFT propagation steps. 
            # The source says "crossing each layer... modeled as small propagation... followed by multiplication"
            # We assume the homogeneous propagation is handled by the BPM engine.
            # Here we just apply the local phase modulation.
            
            for n in range(N):
                # Apply phase mask
                current_fields[n] = current_fields[n] * phase_mask
                # [G] Call BPM propagation step for dz_sim in homogeneous medium n_n
                current_fields[n] = bpm_propagate(current_fields[n], wavelengths[n], bulk_indices[n], dz_sim)
                
        stored_fields.append([np.copy(u) for u in current_fields]) # Store at z_{K+1}
        
        # Step 2: Convergence Check
        # Compute global overlap O_max
        # [G] Implement integration of Re(v* u) over the output plane
        global_overlap = 0.0
        for n in range(N):
            global_overlap += np.sum(np.real(np.conj(target_modes[n]) * current_fields[n]))
            
        if iteration > 0 and abs(global_overlap - prev_overlap) < convergence_threshold:
            break
        prev_overlap = global_overlap
        
        # Step 3: Backward Iteration and Voxel Selection
        # Start from k=K down to 1
        # We need the target modes at z_{k+1} to backpropagate to z_k
        
        # Initialize backpropagated fields with target modes at z_{K+1}
        back_fields = [np.copy(v) for v in target_modes]
        
        for k in range(K-1, -1, -1): # k goes K-1 down to 0 (indices of stored_fields)
            # stored_fields[k] holds fields at z_{k+1} BEFORE crossing v_{i,j,k}?
            # Text: "saved before crossing each sub-volume v_ijk ... note them {u_ijk^n}"
            # "note {z_k} the common transverse planes where distributions are saved"
            # So stored_fields[k] is at z_k? Or z_{k+1}?
            # "Propagate ... from z_in=z_1 to z_out=z_{K+1} ... store ... at planes {z_k}"
            # Usually z_1 is input. So stored_fields[0] is at z_1?
            # Let's align with the selection criterion:
            # O_ijk_max uses u_ijk+1^n(d) propagated from z_k to z_{k+1}
            # and v_ijk+1^n at z_{k+1}.
            
            # Let's assume stored_fields[k] corresponds to plane z_{k+1} (after crossing layer k)
            # And we have back_fields at z_{k+1}
            
            # We need to select d for each (i,j) in layer k
            # To do this, we need u_ijk+1^n(d) which is u_ijk^n propagated through d
            
            # Get forward fields at z_k (before crossing layer k)
            # These are stored in stored_fields[k-1] if 0-indexed properly?
            # Let's assume we have access to fields at z_k and z_{k+1}
            
            # [G] Retrieve forward fields at z_k
            forward_fields_at_zk = stored_fields[k] # Assuming this is at z_k
            
            # Iterate over all (i,j) in this layer
            for i in range(I):
                for j in range(J):
                    best_d = 0
                    best_score = -np.inf
                    
                    # Get forward fields at z_k for this sub-volume
                    # [G] Extract sub-region from forward_fields_at_zk for (i,j)
                    # This requires spatial mapping
                    u_forward_sub = get_sub_region(forward_fields_at_zk, i, j)
                    
                    # Get backpropagated target fields at z_{k+1}
                    # [G] Extract sub-region from back_fields for (i,j)
                    v_target_sub = get_sub_region(back_fields, i, j)
                    
                    for d in range(D):
                        # Propagate u_forward_sub through voxel d to z_{k+1}
                        # Apply phase
                        phase = phase_factors[d, :, 0, 0] # Simplified: assume uniform phase over sub-region?
                        # No, phase is local.
                        # [G] Apply phase factor d to u_forward_sub
                        u_propagated = u_forward_sub * phase_factors[d, :, 0, 0] # Placeholder for spatial application
                        # [G] Propagate through homogeneous medium dz_sim
                        u_propagated = bpm_propagate(u_propagated, wavelengths, bulk_indices, dz_sim)
                        
                        # Calculate local criterion
                        score = 0.0
                        for n in range(N):
                            score += np.sum(np.real(np.conj(v_target_sub[n]) * u_propagated[n]))
                            
                        if score > best_score:
                            best_score = score
                            best_d = d
                    
                    voxel_map[i, j, k] = best_d
            
            # Backpropagate back_fields from z_{k+1} to z_k through the SELECTED voxels
            # Create phase mask for this layer using the newly selected voxels
            phase_mask_back = np.zeros((Nx, Ny), dtype=complex)
            for i in range(I):
                for j in range(J):
                    d_idx = voxel_map[i, j, k]
                    # [G] Apply phase factor d_idx to corresponding region
                    pass
            
            # Backpropagation is inverse of forward propagation
            for n in range(N):
                # [G] Apply inverse phase mask
                back_fields[n] = back_fields[n] * np.conj(phase_mask)
                # [G] Backpropagate through homogeneous medium
                back_fields[n] = bpm_backpropagate(back_fields[n], wavelengths[n], bulk_indices[n], dz_sim)

    return voxel_map
```

**Edge Cases and Pitfalls**
*   **Precision:** The simulation resolution ($dx, dy, dz$) must be much finer than the elementary voxel length ($D_z$) to ensure numerical accuracy [S1, unofficial].
*   **Boundary Conditions:** The algorithm assumes a finite extent sample. Edge effects in the transverse plane may require padding or specific boundary conditions in the BPM [S1, unofficial].
*   **Lossless Assumption:** The maximization criterion (Eq. 6) relies on the conservation of mode energy. If the medium is lossy, the criterion must be adjusted [S1, unofficial].

## Subtopics

### Split-Step Fourier BPM
The propagation engine used for forward and backward steps. It handles diffraction in homogeneous media and phase modulation in voxels. [Link to BPM article if available, otherwise link to Parent](6884-holographic-computer-generated-holography-cgh-calculation.md).

### Voxel Library Design
The set of available transverse refractive index distributions $\{\Delta RI_d\}$ is determined by fabrication capabilities (e.g., femtosecond laser writing parameters). [Link to Parent](6884-holographic-computer-generated-holography-cgh-calculation.md).

## Operating characteristics
*   **Time Complexity:** The algorithm is iterative. Each iteration involves $K$ forward propagation steps and $K$ backward steps. The voxel selection step requires evaluating $D$ candidates for each of the $I \times J$ sub-volumes in each layer. Thus, the complexity per iteration is roughly $O(K \cdot I \cdot J \cdot D)$.
*   **Memory Complexity:** Requires storing fields at each plane $z_k$ for all $N$ modes, plus the voxel map. $O(K \cdot N \cdot N_x \cdot N_y)$.
*   **Strengths:** Allows for multi-wavelength, multi-mode conversion with high efficiency. The iterative nature allows for optimization of complex 3D structures.
*   **Limitations:** Computationally expensive due to the large number of candidates $D$ and the iterative nature. Convergence is not guaranteed and depends on the initial state and the library of voxels [S1, unofficial].
*   **When to Use:** When designing volume holograms for mode sorting or multiplexing where precise control over multiple modes and wavelengths is required.

## Background
The algorithm was proposed in the context of designing efficient computer-generated volume holograms in glass for mode conversion [S1, unofficial]. It builds on the Iterative Propagation Algorithm (IPA) concept, where local overlap integrals are maximized iteratively [S1, unofficial]. The method is applicable to femtosecond laser-written structures in materials like lithium niobate or glass [S1, unofficial].

## Sources
[S1] Design and manufacturing of efficient computer-generated volume holograms in glass, https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
