# Inverse Design Algorithm for Voxel-based CGVHs

Also called: Iterative Propagation Algorithm (IPA) for mode conversion

## Overview
The Inverse Design Algorithm for Voxel-based Computer-Generated Volume Holograms (CGVHs) is a method for designing volumetric optical elements that convert $N$ mutually incoherent transverse input modes $\{u_n\}$ into $N$ target modes $\{v_n\}$ with a one-to-one mapping [S1, unofficial]. It solves the problem of determining the optimal 3D refractive index distribution within a finite glass sample to achieve high-efficiency mode conversion. Unlike direct forward calculation methods, this algorithm uses an iterative optimization loop that alternates between forward propagation of input modes and backpropagation of target modes to locally maximize an overlap criterion at each voxel layer [S1, unofficial]. It is distinct from sibling methods like [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) by focusing on discrete voxel selection from a predefined library of transverse index profiles rather than continuous phase modulation or direct diffraction summation.

## How it works
The algorithm designs a hologram by virtually splitting the sample volume into a grid of sub-volumes $v_{ijk}$, where each sub-volume can contain one of $D$ predefined transverse refractive index distributions $\{\Delta RI_d\}_{1 \le d \le D}$ [S1, unofficial]. The design process minimizes the squared $L_2$ distance between the propagated input modes and the target modes, which is equivalent to maximizing the real part of their overlap integral under lossless conditions [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   $N$ input mode fields $\{u_n\}$ defined at plane $z_{in} = z_1$.
    *   $N$ target mode fields $\{v_n\}$ defined at plane $z_{out} = z_{K+1}$.
    *   Wavelengths $\lambda_n$ associated with each mode pair $(u_n, v_n)$.
    *   Bulk refractive index $n_n$ of the sample material at $\lambda_n$.
    *   Library of $D$ transverse voxel distributions $\{\Delta RI_d\}$, representing relative refractive index changes compared to the bulk index. $\Delta RI_0$ denotes no modification [S1, unofficial].
    *   Grid dimensions: $I \times J$ transverse sub-volumes, $K$ longitudinal layers.
    *   Sub-volume dimensions: $D_x, D_y, D_z$.
    *   Simulation resolutions: $dx, dy, dz$.
*   **Outputs:**
    *   A 3D map of selected voxel distributions $\{\Delta RI_{ijk}^*\}$ for each sub-volume $v_{ijk}$, defining the final hologram structure [S1, unofficial].

### Data Structures
The sample is discretized into a 3D grid of sub-volumes indexed by $(i, j, k)$ where $1 \le i \le I$, $1 \le j \le J$, and $1 \le k \le K$.
*   **Sub-volume $v_{ijk}$:** A placeholder for a voxel distribution. Its physical extent is $D_x \times D_y \times D_z$.
*   **Field Arrays:** Complex-valued arrays storing mode distributions $u_{ijk}^n$ and $v_{ijk}^n$ at transverse planes $z_k$.
*   **Selection Map:** A 3D integer array $S[i][j][k]$ storing the index $d \in \{0, \dots, D\}$ of the selected distribution for each sub-volume.

### Parameters
*   **Elementary length:** $D_z = 10 \, \mu\text{m}$ [S1, unofficial].
*   **Simulation resolution:** $dx = dy = 0.25 \, \mu\text{m}$, $dz = 0.5 \, \mu\text{m}$ [S1, unofficial].
*   **Propagation Model:** Split-step Fourier Beam Propagation Method (BPM) [S1, unofficial].

### Formulas
The propagation through a layer of thickness $dz$ with a local refractive index change $\Delta RI_{ijk}^d$ is modeled by a phase multiplication [S1, unofficial]:
$$ \phi_{ijk}(\lambda_n) = \exp\left(i \frac{2\pi}{\lambda_n} \Delta RI_{ijk}^d(\lambda_n) dz\right), \quad 0 \le d \le D $$

The global error metric to minimize is the squared $L_2$ distance:
$$ O_{min} = \min \sum_{n=1}^{N} \|u_n - v_n\|_2^2 = \min \left\{ \sum_{n=1}^{N} \int_{z_{out}} |u_n - v_n|^2 \right\} $$

For lossless designs where $\|u_n\|_2^2$ and $\|v_n\|_2^2$ are conserved, this is equivalent to maximizing the overlap criterion [S1, unofficial]:
$$ O_{max} = \max_{\Delta RI_{ijk}^d} \left\{ \sum_{n=1}^{N} \int_{z_{out}} \Re(v_n^* u_n) \right\} $$

The local selection criterion for sub-volume $v_{ijk}$ at layer $k$ is [S1, unofficial]:
$$ O_{ijk}^{max} = \max_{d} \left\{ \sum_{n=1}^{N} \int_{z_{k+1}} \Re\left( (v_{ijk+1}^n)^* u_{ijk+1}^n(d) \right) \right\} $$
where $u_{ijk+1}^n(d)$ is the input mode propagated from plane $z_k$ to $z_{k+1}$ through the distribution $\Delta RI_{ijk}^d$.

### Algorithm Steps
1.  **Initialization:** Set all sub-volumes $v_{ijk}$ to the null distribution $\Delta RI_0$. Initialize the selection map $S[i][j][k] = 0$.
2.  **Forward Propagation:** Propagate all input modes $\{u_n\}$ from $z_1$ to $z_{K+1}$ using the split-step Fourier BPM.
    *   For each layer $k$ from 1 to $K$:
        *   Store the transverse field distributions $\{u_{ijk}^n\}$ at plane $z_k$ before crossing $v_{ijk}$.
        *   Apply the phase mask $\phi_{ijk}$ corresponding to the current selection $S[i][j][k]$ for each mode $n$.
        *   Propagate through the layer of thickness $dz$.
3.  **Convergence Check:** Compute the final overlap criterion $O_{max}$ at $z_{K+1}$. If the value is satisfactory or has stagnated, terminate and output the current selection map [S1, unofficial].
4.  **Backward Propagation and Voxel Selection:**
    *   Initialize target fields at $z_{K+1}$ as $\{v_{ijk+1}^n\} = \{v_n\}$.
    *   Iterate $k$ from $K$ down to 1:
        *   For each sub-volume $v_{ijk}$ (for all $i, j$):
            *   For each candidate distribution $d \in \{0, \dots, D\}$:
                *   Propagate the current stored input fields $\{u_{ijk}^n\}$ from $z_k$ to $z_{k+1}$ using $\Delta RI_{ijk}^d$ to obtain $\{u_{ijk+1}^n(d)\}$.
                *   Compute the local criterion $O_{ijk}^d = \sum_{n=1}^{N} \int_{z_{k+1}} \Re\left( (v_{ijk+1}^n)^* u_{ijk+1}^n(d) \right)$.
            *   Select $d^*$ that maximizes $O_{ijk}^d$.
            *   Update $S[i][j][k] = d^*$.
        *   Backpropagate the target fields $\{v_{ijk+1}^n\}$ from $z_{k+1}$ to $z_k$ using the newly selected distributions $\{\Delta RI_{ijk}^{S[i][j][k]}\}$ for all $i, j$.
        *   Update $\{v_{ijk}^n\}$ for the next iteration step.
5.  **Repeat:** Return to Step 2 with the updated selection map.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_design_cgvh(N, K, I, J, D, u_in, v_target, lambda_n, n_bulk, 
                        delta_RI_library, dx, dy, dz, Dz):
    """
    N: Number of modes
    K: Number of longitudinal layers
    I, J: Transverse grid dimensions
    D: Number of candidate voxel distributions
    u_in: List of N input mode fields at z_1
    v_target: List of N target mode fields at z_{K+1}
    lambda_n: List of wavelengths for each mode
    n_bulk: Bulk refractive index (may depend on lambda)
    delta_RI_library: List of D transverse index distributions (delta_RI_0 is null)
    dx, dy, dz: Simulation resolutions
    Dz: Elementary layer thickness
    """
    
    # Initialize selection map with null distribution (index 0)
    # S[i][j][k] stores the index d in delta_RI_library
    S = [[[0 for _ in range(K)] for _ in range(J)] for _ in range(I)]
    
    # Current fields for forward pass
    u_current = [copy_field(u) for u in u_in]
    
    # Convergence threshold
    threshold = 1e-6
    prev_overlap = -1
    
    while True:
        # Step 1: Forward Propagation
        # Store fields at each plane z_k
        u_stored = [[[[None for _ in range(K)] for _ in range(J)] for _ in range(I)] for _ in range(N)]
        
        for k in range(K):
            # Save fields before crossing layer k
            for n in range(N):
                u_stored[n][i][j][k] = copy_field(u_current[n])
            
            # Apply phase masks and propagate through layer k
            for i in range(I):
                for j in range(J):
                    d_idx = S[i][j][k]
                    delta_RI = delta_RI_library[d_idx]
                    
                    # Calculate phase mask for this sub-volume
                    # phi = exp(i * 2*pi/lambda * delta_RI * dz)
                    phase_mask = compute_phase_mask(delta_RI, lambda_n, dz)
                    
                    for n in range(N):
                        # Apply phase multiplication
                        u_current[n] = u_current[n] * phase_mask[n]
            
            # Propagate through homogeneous medium of thickness dz
            # Using Split-Step Fourier BPM
            for n in range(N):
                u_current[n] = split_step_bpm(u_current[n], n_bulk, lambda_n[n], dz, dx, dy)
        
        # Step 2: Compute final overlap criterion
        # O_max = sum_n Re(int(v_n* * u_n))
        current_overlap = 0.0
        for n in range(N):
            # Integrate over the output plane z_{K+1}
            # Assuming u_current[n] is now at z_{K+1}
            overlap_integral = integrate_real_part_conjugate(u_current[n], v_target[n])
            current_overlap += overlap_integral
            
        # Check convergence
        if prev_overlap != -1:
            if abs(current_overlap - prev_overlap) < threshold:
                break
        prev_overlap = current_overlap
        
        # Step 3: Backward Propagation and Voxel Selection
        # Initialize target fields at z_{K+1}
        v_current = [copy_field(v) for v in v_target]
        
        for k in range(K - 1, -1, -1):
            # We need to select the best voxel for layer k
            # The target fields are currently at z_{k+1} (stored in v_current)
            # The input fields are stored at z_k (in u_stored)
            
            for i in range(I):
                for j in range(J):
                    best_d = 0
                    best_score = -infinity
                    
                    # Get stored input fields at z_k for this sub-volume
                    u_input_at_zk = [u_stored[n][i][j][k] for n in range(N)]
                    
                    # Get target fields at z_{k+1}
                    v_target_at_zk1 = [v_current[n] for n in range(N)]
                    
                    # Try all candidate distributions
                    for d in range(D):
                        delta_RI = delta_RI_library[d]
                        phase_mask = compute_phase_mask(delta_RI, lambda_n, dz)
                        
                        # Propagate u_input from z_k to z_{k+1} through this voxel
                        u_propagated = [copy_field(u) for u in u_input_at_zk]
                        
                        # Apply phase mask
                        for n in range(N):
                            u_propagated[n] = u_propagated[n] * phase_mask[n]
                        
                        # Propagate through homogeneous medium dz
                        for n in range(N):
                            u_propagated[n] = split_step_bpm(u_propagated[n], n_bulk, lambda_n[n], dz, dx, dy)
                        
                        # Compute local criterion
                        score = 0.0
                        for n in range(N):
                            score += integrate_real_part_conjugate(u_propagated[n], v_target_at_zk1[n])
                        
                        if score > best_score:
                            best_score = score
                            best_d = d
                    
                    # Update selection map
                    S[i][j][k] = best_d
            
            # Backpropagate target fields from z_{k+1} to z_k
            # Use the newly selected distributions for all sub-volumes in layer k
            for i in range(I):
                for j in range(J):
                    d_idx = S[i][j][k]
                    delta_RI = delta_RI_library[d_idx]
                    phase_mask = compute_phase_mask(delta_RI, lambda_n, dz)
                    
                    for n in range(N):
                        # Apply phase mask (conjugate for backprop? 
                        # Note: The source describes backpropagation through the selected structure.
                        # In lossless reciprocal systems, backpropagation through index profile n(r) 
                        # is equivalent to forward propagation through n(r) with reversed direction.
                        # The source implies using the same phase multiplication model.
                        v_current[n] = v_current[n] * phase_mask[n]
            
            # Backpropagate through homogeneous medium dz
            for n in range(N):
                v_current[n] = split_step_bpm_back(v_current[n], n_bulk, lambda_n[n], dz, dx, dy)
                
        # Step 4: Repeat
        # u_current is already updated from forward pass, but we need to reset it 
        # for the next forward pass iteration? 
        # The algorithm summary says "Repeat procedure from Step 1".
        # This implies we start forward propagation again with the new S map.
        # However, u_current holds the fields at z_{K+1} from the previous forward pass.
        # We must reset u_current to u_in for the new forward pass.
        u_current = [copy_field(u) for u in u_in]
```

### Edge Cases and Pitfalls
*   **Precision:** The simulation resolution $dz$ ($0.5 \, \mu\text{m}$) is half the elementary length $D_z$ ($10 \, \mu\text{m}$). Insufficient resolution may lead to inaccurate phase accumulation [S1, unofficial].
*   **Boundary Conditions:** The algorithm assumes a finite extent sample. Edge effects in the transverse grid $(i, j)$ may require padding or specific boundary conditions in the BPM propagation [S1, unofficial].
*   **Wavelength Dependence:** The voxel distributions $\Delta RI_d$ and bulk index $n_n$ are wavelength-dependent. The algorithm must handle multi-wavelength inputs correctly by applying the appropriate $\lambda_n$ for each mode [S1, unofficial].
*   **Convergence Stagnation:** The algorithm may stall in local optima. The convergence check compares the overlap criterion between iterations; if the change is below a threshold, it exits [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The algorithm is iterative. Each iteration involves $K$ forward propagation steps and $K$ backward propagation steps. In each backward step, for each of the $I \times J$ sub-volumes, it evaluates $D$ candidate distributions. Thus, one iteration requires $O(K \cdot I \cdot J \cdot D)$ propagation operations. The total time depends on the number of iterations until convergence.
*   **Memory Complexity:** Requires storing the field distributions for $N$ modes at each of the $K$ planes during the forward pass ($O(N \cdot K \cdot I \cdot J)$) and the selection map ($O(I \cdot J \cdot K)$).
*   **Strengths:** Allows for the design of complex 3D voxel structures for efficient mode conversion. The use of a predefined library of voxels simplifies the fabrication process [S1, unofficial].
*   **Limitations:** The search space is discrete ($D$ choices per voxel), which may not find the globally optimal continuous index profile. The iterative nature can be computationally expensive for large $I, J, K, D$.
*   **When to use:** When designing volume holograms for mode conversion where the fabrication process supports writing discrete voxel structures with a limited set of index profiles [S1, unofficial].

## Background
The Inverse Design Algorithm for Voxel-based CGVHs was introduced in the context of designing efficient computer-generated volume holograms in glass for mode sorting [S1, unofficial]. It builds upon the Iterative Propagation Algorithm (IPA) concept, where the overlap integral is maximized iteratively in reverse [S1, unofficial]. The method is particularly relevant for applications in optical communications and quantum photonics where efficient mode conversion is required [S1, unofficial].

## Sources
[S1] Design and manufacturing of efficient computer-generated volume holograms in glass, https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
