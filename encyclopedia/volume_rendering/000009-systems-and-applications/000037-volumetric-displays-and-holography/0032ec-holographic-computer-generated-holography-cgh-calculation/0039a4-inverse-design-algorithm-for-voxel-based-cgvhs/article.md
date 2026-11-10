# Inverse Design Algorithm for Voxel-based CGVHs

Also called: Iterative Propagation Algorithm (IPA) for mode conversion

## Overview
This method designs voxel-based Computer-Generated Volume Holograms (CGVHs) that convert $N$ mutually incoherent transverse input modes $\{u_n\}$ into $N$ target modes $\{v_n\}$ with a one-to-one mapping [S1, unofficial]. It solves the inverse problem of determining the optimal 3D refractive index distribution within a finite sample to achieve specific mode conversions. Unlike direct forward calculation methods, it uses an iterative optimization loop involving forward and backward propagation to maximize the overlap between propagated fields and target modes [S1, unofficial]. It is distinct from sibling methods like [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) by focusing on discrete voxel selection for mode sorting rather than continuous field synthesis for image reconstruction.

## How it works
**Inputs:**
*   $N$ input mode fields $\{u_n\}$ defined at plane $z_{in} = z_1$.
*   $N$ target mode fields $\{v_n\}$ defined at plane $z_{out} = z_{K+1}$.
*   Wavelengths $\lambda_n$ associated with each mode pair $(u_n, v_n)$ [S1, unofficial].
*   Bulk refractive index $n_n$ of the sample at $\lambda_n$ [S1, unofficial].
*   A library of $D$ transverse voxel distributions $\{\Delta RI_d\}_{1 \le d \le D}$ representing relative refractive index changes, plus a null distribution $\Delta RI_0$ [S1, unofficial].
*   Sample geometry: Rectangular coordinate system $(O,x,y,z)$ with $z$ as propagation direction. The sample is virtually split into sub-volumes $v_{ijk}$ ($1 \le i \le I, 1 \le j \le J, 1 \le k \le K$) with extensions $D_x, D_y, D_z$ [S1, unofficial].

**Outputs:**
*   A 3D map of selected voxel distributions $\{\Delta RI_{ijk}^*\}$ for each sub-volume $v_{ijk}$, defining the final CGVH structure [S1, unofficial].

**Data Structures:**
*   **Grid:** 3D array representing the sample, indexed by $(i,j,k)$.
*   **Voxel Library:** List of $D+1$ 2D complex fields or refractive index maps corresponding to $\Delta RI_d$.
*   **Field Storage:** 3D array or list of 2D fields to store intermediate mode distributions $u_{ijk}^n$ at each plane $z_k$ [S1, unofficial].

**Parameters:**
*   $D_z$: Elementary length (waveguide length step). Typical value: $10 \mu m$ [S1, unofficial].
*   $dx, dy$: Transverse simulation resolution. Typical value: $0.25 \mu m$ [S1, unofficial].
*   $dz$: Longitudinal simulation resolution. Typical value: $0.5 \mu m$ [S1, unofficial].
*   $K$: Number of longitudinal layers [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Set all sub-volumes $v_{ijk}$ to the null distribution $\Delta RI_0$.
2.  **Forward Propagation:** Propagate all input modes $\{u_n\}$ from $z_1$ to $z_{K+1}$ using the Split-Step Fourier Beam Propagation Method (BPM) [S1, unofficial].
    *   The sample is traversed layer by layer for $k$ from 1 to $K$.
    *   For each layer $k$, the field is propagated through the current voxel distribution $\Delta RI_{ijk}$ in each sub-volume $v_{ijk}$.
    *   The phase multiplication model for crossing a layer of thickness $dz$ through a homogeneous medium $n_n$ with local phase $\phi_{ijk}$ is:
        $$ \phi_{ijk}(\lambda_n) = \exp\left(i \frac{2\pi}{\lambda_n} \Delta RI_{ijk}^d(\lambda_n) dz\right), \quad 0 \le d \le D $$
    *   Save the transverse field distributions $\{u_{ijk}^n\}$ at the planes $z_k$ (before crossing $v_{ijk}$) for later reference [S1, unofficial].
3.  **Convergence Check:** Compute the global overlap criterion $O_{max}$ (see Formulas). If the value is satisfactory or stagnant, terminate and output the current voxel selection [S1, unofficial].
4.  **Backward Propagation and Voxel Selection:**
    *   Start from the output plane $z_{K+1}$ with the target modes $\{v_n\}$.
    *   Iterate $k$ from $K$ down to 1:
        *   For each sub-volume $v_{ijk}$, select the optimal distribution $d^*$ that maximizes the local criterion $O_{ijk}^{max}$ (see Formulas).
        *   To find $d^*$, try all distributions $\{\Delta RI_{ijk}^d\}$ for $d \in \{0, \dots, D\}$.
        *   Backpropagate the target modes from $z_{k+1}$ to $z_k$ using the selected distributions for layers $k+1 \dots K$ and the newly selected distributions for layer $k$.
        *   Store the backpropagated fields at $z_k$ as the new "target" for the next iteration step $k-1$ [S1, unofficial].
5.  **Repeat:** Return to Step 2 with the updated voxel selection.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_design_cgvh(input_modes, target_modes, wavelengths, bulk_indices, 
                        voxel_library, sample_dims, resolution, max_iterations):
    """
    input_modes: List of N complex 2D arrays (fields at z_in)
    target_modes: List of N complex 2D arrays (fields at z_out)
    wavelengths: List of N floats
    bulk_indices: List of N floats
    voxel_library: List of D+1 complex 2D arrays (refractive index changes)
    sample_dims: Tuple (I, J, K, Dx, Dy, Dz)
    resolution: Tuple (dx, dy, dz)
    """
    I, J, K, Dx, Dy, Dz = sample_dims
    dx, dy, dz = resolution
    
    # Initialize voxel selection map: shape (I, J, K), stores index d
    voxel_selection = np.zeros((I, J, K), dtype=int)
    
    for iteration in range(max_iterations):
        # Step 1: Forward Propagation
        current_fields = [np.copy(u) for u in input_modes]
        stored_fields = {} # Dictionary to store fields at each plane z_k
        
        # Propagate from z_1 to z_{K+1}
        for k in range(1, K + 1):
            # Store fields before crossing layer k
            stored_fields[k] = [np.copy(u) for u in current_fields]
            
            # Propagate through layer k
            new_fields = []
            for n in range(len(input_modes)):
                u_n = current_fields[n]
                # Split-step BPM: Propagate through homogeneous medium then apply phase
                # Note: Actual BPM involves FFT/IFFT steps for homogeneous propagation.
                # Here we abstract the homogeneous propagation as a function.
                u_propagated = propagate_homogeneous(u_n, wavelengths[n], bulk_indices[n], dz, resolution)
                
                # Apply local phase from selected voxels in this layer
                # The field is a superposition, but modes are incoherent, so we process each mode independently
                # through the specific voxel path. However, the algorithm implies a single voxel map 
                # affects all modes. The phase is wavelength dependent.
                
                # Create phase mask for this layer based on current selection
                phase_mask = np.zeros((I, J), dtype=complex)
                for i in range(I):
                    for j in range(J):
                        d_idx = voxel_selection[i, j, k]
                        # Interpolate or apply the specific voxel distribution
                        # Assuming voxel_library[d] is a 2D map relative to sub-volume
                        # In practice, this requires mapping the 2D voxel to the grid
                        phase_mask[i, j] = voxel_library[d_idx] 
                
                # Apply phase multiplication
                # phi_ijk = exp(i * 2pi/lambda * dRI * dz)
                # The voxel_library likely already contains the phase or dRI. 
                # Based on formula: phi = exp(i * 2pi/lambda * dRI * dz)
                # If voxel_library stores dRI, we compute phase here.
                # If it stores phase directly, use it. Let's assume it stores dRI for generality.
                dRI = voxel_library[voxel_selection[:, :, k]] # Shape (I, J)
                phase = np.exp(1j * 2 * np.pi / wavelengths[n] * dRI * dz)
                
                # Apply phase to the propagated field
                # Note: The field u_propagated is on a fine grid. The voxel selection is on a coarse grid.
                # Upsampling/interpolation is required.
                u_final = apply_phase_mask(u_propagated, phase, resolution, sample_dims)
                new_fields.append(u_final)
            
            current_fields = new_fields
            
        # Store final fields for convergence check
        final_fields = current_fields
        
        # Step 2: Convergence Check
        overlap = compute_overlap(final_fields, target_modes)
        if is_converged(overlap, iteration):
            break
            
        # Step 3: Backward Propagation and Voxel Selection
        # Start with target modes at z_{K+1}
        back_fields = [np.copy(v) for v in target_modes]
        
        for k in range(K, 0, -1):
            # For each sub-volume in layer k, find best d
            for i in range(I):
                for j in range(J):
                    best_d = 0
                    best_score = -np.inf
                    
                    # Get stored forward field at z_k for this sub-volume location
                    # The stored fields are global, need to extract region corresponding to v_ij
                    # This requires spatial indexing.
                    u_forward = extract_region(stored_fields[k], i, j, Dx, Dy, resolution)
                    
                    # Get backpropagated target field at z_{k+1}
                    # This is the field arriving at the output of sub-volume v_ij
                    v_target_next = extract_region(back_fields, i, j, Dx, Dy, resolution)
                    
                    # Try all distributions
                    for d in range(len(voxel_library)):
                        # Propagate u_forward from z_k to z_{k+1} through voxel d
                        dRI = voxel_library[d]
                        phase = np.exp(1j * 2 * np.pi / wavelengths[n] * dRI * dz) # For each n
                        
                        # Propagate locally
                        u_local = propagate_homogeneous(u_forward, wavelengths[n], bulk_indices[n], dz, resolution)
                        u_local = apply_phase_mask(u_local, phase, resolution, sample_dims)
                        
                        # Compute local criterion O_ijk_max
                        # O_ijk_max = max_d sum_n Re( (v_{ijk+1}^n)* u_{ijk+1}^n(d) )
                        score = 0
                        for n in range(len(input_modes)):
                            # Note: v_target_next must correspond to mode n
                            v_n = extract_region(back_fields[n], i, j, Dx, Dy, resolution)
                            u_n_local = extract_region(u_local, i, j, Dx, Dy, resolution) # u_local is list of fields
                            
                            # Overlap integral
                            overlap_val = np.real(np.sum(np.conj(v_n) * u_n_local))
                            score += overlap_val
                            
                        if score > best_score:
                            best_score = score
                            best_d = d
                    
                    voxel_selection[i, j, k] = best_d
            
            # Backpropagate fields from z_{k+1} to z_k through the selected voxels in layer k
            # This is the inverse of the forward step
            new_back_fields = []
            for n in range(len(input_modes)):
                v_n = back_fields[n]
                # Backpropagate through layer k
                # Reverse the phase application and homogeneous propagation
                v_propagated = backpropagate_homogeneous(v_n, wavelengths[n], bulk_indices[n], dz, resolution)
                
                # Apply inverse phase
                phase_inv = np.conj(np.exp(1j * 2 * np.pi / wavelengths[n] * 
                                          voxel_library[voxel_selection[:, :, k]] * dz))
                v_final = apply_phase_mask(v_propagated, phase_inv, resolution, sample_dims)
                new_back_fields.append(v_final)
            
            back_fields = new_back_fields

    return voxel_selection

def compute_overlap(fields, targets):
    # O_max = sum_n Re( integral (v_n* u_n) )
    total_overlap = 0
    for n in range(len(fields)):
        total_overlap += np.real(np.sum(np.conj(targets[n]) * fields[n]))
    return total_overlap
```

**Edge Cases and Pitfalls:**
*   **Precision:** The simulation resolution $dx, dy, dz$ must be much finer than the voxel dimensions to avoid numerical artifacts [S1, unofficial].
*   **Boundary Conditions:** The algorithm assumes a finite extent sample. Boundary effects at $x, y$ edges may require padding or specific boundary conditions in the BPM [S1, unofficial].
*   **Incoherence:** The modes are mutually incoherent, allowing the superposition principle to be applied independently for each mode in the overlap calculation [S1, unofficial].
*   **Losslessness:** The maximization criterion $O_{max}$ is derived from the minimization of $L_2$ distance under the assumption of lossless design (conservation of $||u_n||^2$ and $||v_n||^2$). If losses are present, the criterion may need adjustment [S1, unofficial].

## Operating characteristics
*   **Complexity:** The algorithm is iterative. Each iteration involves $K$ forward and $K$ backward propagation steps. In each backward step, for each of the $I \times J$ sub-volumes, $D$ candidate distributions are tested. Each test involves a local propagation. The total complexity per iteration is roughly $O(K \cdot I \cdot J \cdot D \cdot \text{BPM\_cost})$. BPM cost is $O(M \log M)$ for a grid of size $M$.
*   **Memory:** Requires storing fields at each plane $z_k$ for all $N$ modes, plus the voxel selection map. Memory scales with $N \cdot K \cdot \text{GridSize}$.
*   **Strengths:** Provides an exact inverse design for mode conversion tasks. Handles multiple wavelengths and modes simultaneously.
*   **Limitations:** Computationally expensive due to the iterative nature and the need to test $D$ candidates for every voxel. Convergence is not guaranteed to be global optimum.
*   **When to use:** When precise mode conversion is required for volumetric displays or optical communication systems, and computational resources allow for iterative optimization.

## Background
The method was introduced in the context of designing efficient computer-generated volume holograms in glass for mode sorting applications [S1, unofficial]. It builds on the Iterative Propagation Algorithm (IPA) concept, where overlap integrals are maximized iteratively [S1, unofficial]. The specific application to voxel-based CGVHs allows for the fabrication of complex 3D optical elements using femtosecond laser writing techniques, where discrete voxel distributions can be written into the glass [S1, unofficial].

## Sources
[S1] Design and manufacturing of efficient computer-generated volume holograms in glass, https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
