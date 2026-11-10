# Inverse Design Algorithm for Voxel-based CGVHs

Also called: Iterative Propagation Algorithm (IPA) for mode conversion

## Overview
This method designs voxel-based Computer-Generated Volume Holograms (CGVHs) that convert $N$ mutually incoherent transverse input modes $\{u_n\}$ into $N$ target modes $\{v_n\}$ with a one-to-one mapping [S1, unofficial]. It solves the problem of determining the optimal 3D refractive index distribution within a finite volume to achieve high-fidelity mode conversion for multiple wavelengths simultaneously. Unlike direct phase extraction methods which suffer from noise and twin-image artifacts, this algorithm uses an iterative forward-backward propagation scheme to maximize the overlap between propagated and target modes [S1, unofficial]. It is distinct from sibling methods like [Volume Rendering based CGH Calculation](808-volume-rendering-based-cgh-calculation.md) by focusing on discrete voxel selection within a split-step framework rather than continuous field optimization or stereographic projection.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $N$ input mode fields $\{u_n\}_{n=1}^N$ defined at plane $z_{in} = z_1$.
    *   $N$ target mode fields $\{v_n\}_{n=1}^N$ defined at plane $z_{out} = z_{K+1}$.
    *   Wavelengths $\lambda_n$ associated with each mode $n$.
    *   Bulk refractive index $n_n$ at wavelength $\lambda_n$.
    *   A library of $D$ transverse voxel distributions $\{\Delta RI_d\}_{d=1}^D$, representing relative refractive index changes compared to the bulk index. $\Delta RI_0$ denotes no modification.
    *   Sub-volume dimensions $D_x, D_y, D_z$.
*   **Outputs:**
    *   A 3D map of selected voxel indices $\{d_{ijk}\}$ for each sub-volume $v_{ijk}$, defining the final CGVH structure.

**Data Structures:**
The sample volume is virtually split into a grid of sub-volumes $v_{ijk}$ where $1 \le i \le I$, $1 \le j \le J$, and $1 \le k \le K$.
*   **Sub-volume:** Identified by indices $(i,j,k)$.
*   **Field Storage:** During propagation, transverse field distributions are stored at common planes $z_k$. Let $u_{ijk}^n$ denote the field of mode $n$ at plane $z_k$ within sub-volume $(i,j)$.
*   **Resolution:** Simulation uses spatial resolutions $dx, dy, dz$. Typical values: $dx=dy=0.25\,\mu\text{m}$, $dz=0.5\,\mu\text{m}$ [S1, unofficial].
*   **Elementary Length:** The longitudinal step $D_z$ is typically $10\,\mu\text{m}$ [S1, unofficial].

**Mathematical Model:**
Propagation through a layer of thickness $dz$ is modeled using the split-step Fourier Beam Propagation Method (BPM). Crossing a layer through a homogeneous medium $n_n$ followed by a local phase shift $\phi_{ijk}$ is given by [S1, unofficial]:
$$ \phi_{ijk}(\lambda_n) = \exp\left(i \frac{2\pi}{\lambda_n} \Delta RI_{ijk}^d(\lambda_n) dz\right), \quad 0 \le d \le D $$
The global objective is to minimize the squared $L_2$ distance between input and target modes:
$$ O_{min} = \min \sum_{n=1}^N \|u_n - v_n\|_2^2 = \min \left\{ \sum_{n=1}^N \int_{z_{out}} |u_n - v_n|^2 \right\} $$
For lossless designs where energy is conserved ($\|u_n\|^2$ and $\|v_n\|^2$ are constant), this minimization is equivalent to maximizing the overlap integral [S1, unofficial]:
$$ O_{max} = \max_{\Delta RI_{ijk}^d} \left\{ \sum_{n=1}^N \int_{z_{out}} \Re(v_n^* u_n) \right\} $$

**Algorithm Steps:**
1.  **Forward Propagation:** Propagate input modes $\{u_n\}$ from $z_1$ to $z_{K+1}$ using BPM. Initialize all sub-volumes $v_{ijk}$ with the null distribution $\Delta RI_0$. Store the transverse field distributions $\{u_{ijk}^n\}$ at each intermediate plane $z_k$ before crossing sub-volume $v_{ijk}$.
2.  **Convergence Check:** Compute the final overlap criterion $O_{max}$ (Eq. 6). If the value is satisfactory or has stagnated, terminate and output the current voxel selection.
3.  **Backward Propagation and Selection:** Iteratively backpropagate target modes $\{v_n\}$ from $z_{K+1}$ to $z_1$. For each plane $k$ from $K$ down to $1$:
    *   For each sub-volume $(i,j)$ at plane $k$, select the voxel distribution $d$ that maximizes the local multi-objective criterion:
        $$ O_{ijk}^{max} = \max_d \left\{ \sum_{n=1}^N \int_{z_{k+1}} \Re\left( (v_{ijk+1}^n)^* u_{ijk+1}^n(d) \right) \right\} $$
        where $u_{ijk+1}^n(d)$ is the field propagated from $z_k$ to $z_{k+1}$ through the candidate distribution $\Delta RI_{ijk}^d$.
    *   This requires testing all $D$ candidate distributions for each sub-volume.
    *   Once the best $d$ is selected for all $(i,j)$ at plane $k$, backpropagate the target modes from $z_{k+1}$ to $z_k$ using the newly selected voxels.
    *   Repeat for $k-1$ until $k=1$.
4.  **Iteration:** Repeat steps 1–3 until convergence.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_design_cgvh(N, input_modes, target_modes, wavelengths, bulk_indices, 
                        voxel_library, D_z, dx, dy, dz, max_iterations, tol):
    """
    N: int, number of modes
    input_modes: list of complex arrays, shape (H, W), defined at z_1
    target_modes: list of complex arrays, shape (H, W), defined at z_{K+1}
    wavelengths: list of floats, length N
    bulk_indices: list of floats, length N
    voxel_library: list of complex phase masks, shape (H, W), length D+1
                   voxel_library[0] is all ones (null modification)
    D_z: float, elementary longitudinal length
    dx, dy, dz: float, simulation resolutions
    max_iterations: int
    tol: float, convergence threshold
    """
    # Determine grid dimensions based on sample size and D_z
    # I, J, K are derived from physical sample dimensions / D_x, D_y, D_z
    # For this pseudocode, we assume I, J, K are known or derived from input field size
    # and the BPM handles the internal propagation steps.
    
    # Initialize voxel selection map: selection[i][j][k] = d
    # D is the index in voxel_library
    K = number_of_longitudinal_steps(D_z, sample_length)
    I, J = number_of_transverse_blocks(H, W, D_x, D_y)
    
    # Initialize with null distribution
    selection = [[[0 for _ in range(K)] for _ in range(J)] for _ in range(I)]
    
    for iteration in range(max_iterations):
        # Step 1: Forward Propagation
        # Start with input modes at z_1
        current_fields = list(input_modes)
        stored_fields = [] # List of lists of lists: stored_fields[k][i][j][n]
        
        for k in range(K):
            # Store fields before crossing sub-volume v_ijk
            # stored_fields[k] is a list of I*J fields for each mode n
            layer_fields = []
            for i in range(I):
                for j in range(J):
                    layer_fields.append([current_fields[n] for n in range(N)])
            stored_fields.append(layer_fields)
            
            # Propagate through sub-volumes at plane k
            # Using split-step BPM
            next_fields = [None] * N
            for n in range(N):
                # Propagate through homogeneous medium dz
                # This involves FFT, transfer function, IFFT
                # H(u,v) = exp(i * 2*pi * z * sqrt(1/lambda^2 - u^2 - v^2))
                field_prop = propagate_step(current_fields[n], dz, wavelengths[n], bulk_indices[n])
                
                # Apply local phase from selected voxel
                # The field is divided into blocks (i,j)
                for i in range(I):
                    for j in range(J):
                        d = selection[i][j][k]
                        phase_mask = voxel_library[d]
                        # Apply phase mask to the specific block
                        # Assuming field_prop is already tiled or we apply block-wise
                        field_prop[i*block_h:(i+1)*block_h, j*block_w:(j+1)*block_w] *= phase_mask
                
                next_fields[n] = field_prop
            
            current_fields = next_fields

        # Store final fields for convergence check
        final_fields = current_fields
        
        # Step 2: Convergence Check
        # Compute O_max = sum_n Re(int(v_n* * u_n))
        overlap = 0.0
        for n in range(N):
            # Integral approximation via sum
            overlap += np.sum(np.real(np.conj(target_modes[n]) * final_fields[n]))
        
        if iteration > 0 and abs(overlap - prev_overlap) < tol:
            break
        prev_overlap = overlap
        
        # Step 3: Backward Propagation and Selection
        # Start with target modes at z_{K+1}
        back_fields = list(target_modes)
        
        for k in range(K-1, -1, -1): # From K-1 down to 0
            # We need to select voxels for plane k
            # The fields stored_fields[k] are at z_k (before crossing v_ijk)
            # The fields back_fields are at z_{k+1} (after crossing v_ijk)
            
            for i in range(I):
                for j in range(J):
                    best_d = 0
                    best_score = -np.inf
                    
                    # Get stored input fields at z_k for this sub-volume
                    u_at_zk = [stored_fields[k][i][j][n] for n in range(N)]
                    
                    # Get target fields at z_{k+1}
                    v_at_zk1 = [back_fields[n] for n in range(N)]
                    
                    # Try all voxel distributions
                    for d in range(len(voxel_library)):
                        score = 0.0
                        for n in range(N):
                            # Propagate u_at_zk[n] through candidate voxel d to z_{k+1}
                            # This is a local propagation over dz
                            u_candidate = propagate_step(u_at_zk[n], dz, wavelengths[n], bulk_indices[n])
                            
                            # Apply phase of candidate voxel
                            phase_mask = voxel_library[d]
                            u_candidate[i*block_h:(i+1)*block_h, j*block_w:(j+1)*block_w] *= phase_mask
                            
                            # Compute overlap contribution
                            # Note: v_at_zk1 is the field at z_{k+1}
                            # We compare u_candidate (propagated from z_k) with v_at_zk1
                            # However, v_at_zk1 might be the result of backpropagation from further steps
                            # The criterion is Re(v* u)
                            score += np.sum(np.real(np.conj(v_at_zk1[n]) * u_candidate))
                        
                        if score > best_score:
                            best_score = score
                            best_d = d
                    
                    selection[i][j][k] = best_d
            
            # Backpropagate the target modes from z_{k+1} to z_k using the NEWLY selected voxels
            # This prepares back_fields for the next iteration (k-1)
            new_back_fields = [None] * N
            for n in range(N):
                # Start with back_fields[n] at z_{k+1}
                # We need to backpropagate through the sub-volumes at plane k
                # This is the inverse of the forward step
                field_back = back_fields[n]
                
                for i in range(I):
                    for j in range(J):
                        d = selection[i][j][k]
                        # The forward step was: Propagate dz -> Apply Phase
                        # The backward step is: Apply Conjugate Phase -> Backpropagate dz
                        # Or equivalently, reverse the forward BPM steps
                        
                        # 1. Apply conjugate phase of the selected voxel
                        phase_mask = voxel_library[d]
                        # Inverse phase
                        inv_phase = np.conj(phase_mask)
                        field_back[i*block_h:(i+1)*block_h, j*block_w:(j+1)*block_w] *= inv_phase
                        
                        # 2. Backpropagate through homogeneous medium dz
                        # This is the inverse of propagate_step
                        field_back = backpropagate_step(field_back, dz, wavelengths[n], bulk_indices[n])
                
                new_back_fields[n] = field_back
            
            back_fields = new_back_fields

    return selection
```

**Edge Cases and Pitfalls:**
*   **Precision:** The simulation resolution $dz$ ($0.5\,\mu\text{m}$) must be significantly smaller than the elementary length $D_z$ ($10\,\mu\text{m}$) to ensure numerical accuracy in the split-step BPM [S1, unofficial].
*   **Boundary Conditions:** The algorithm assumes a finite extent sample. Edge effects in the transverse planes $x,y$ may require padding or appropriate boundary conditions in the Fourier transforms.
*   **Lossless Assumption:** The equivalence between minimization and maximization (Eq. 5 to Eq. 6) holds strictly for lossless designs where mode energy is conserved. If absorption is present, the criterion must be adjusted [S1, unofficial].
*   **Computational Cost:** The voxel selection step requires testing $D$ distributions for every sub-volume $(i,j)$ at every plane $k$. This is computationally intensive. Parallelization over sub-volumes or modes is recommended.

## Operating characteristics
*   **Time Complexity:** The algorithm is iterative. Each iteration involves one forward pass and one backward pass. The forward/backward pass complexity is $O(K \cdot I \cdot J \cdot N \cdot \text{FFT\_cost})$. The voxel selection step adds a factor of $D$ to the inner loop, making the selection step $O(K \cdot I \cdot J \cdot D \cdot N \cdot \text{FFT\_cost})$. Total complexity per iteration is dominated by the selection step.
*   **Memory Complexity:** Requires storing fields at each plane $z_k$ for all modes and sub-volumes during the forward pass: $O(K \cdot I \cdot J \cdot N \cdot H \cdot W)$. The voxel library is small.
*   **Strengths:** Handles multiple incoherent modes and wavelengths simultaneously. Provides a physically realizable voxel-based design. Converges to a local optimum that maximizes mode overlap.
*   **Limitations:** Computationally expensive due to the exhaustive search over voxel distributions. Convergence is not guaranteed to be global. Requires a pre-defined library of voxel distributions.
*   **When to use:** When designing volume holograms for mode sorting or multiplexing where discrete fabrication constraints (voxel-based) are present.
*   **When not to use:** For continuous-index designs or when computational resources are limited.

## Background
The method was introduced in the context of designing efficient computer-generated volume holograms in glass for mode conversion [S1, unofficial]. It builds upon the Iterative Propagation Algorithm (IPA) concept, where the global optimization problem is decomposed into local maximization problems at each propagation plane [S1, unofficial]. The approach leverages the conservation of overlap integrals in lossless optical systems to simplify the error metric.

## Sources
[S1] Design and manufacturing of efficient computer-generated volume holograms in glass, https://www.researchgate.net/publication/363920430_Design_and_manufacturing_of_efficient_computer-generated_volume_holograms_in_glass, fetched 2026-11-03 (unofficial)
