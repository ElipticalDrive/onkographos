# Antagonistic Dual-Wavelength TVAM

Also called: Antagonistic TVAM

## Overview
Antagonistic Dual-Wavelength TVAM is a volumetric additive manufacturing integrator that suppresses tomographic background dose by combining a patterned activation field (405 nm) with a separately optimized inhibition field (365 nm) [S1, unofficial]. It extends the standard TVAM Integrator by adding a second wavelength channel, allowing the definition of complete 3D geometries through inhibition and mitigating diffusion- and attenuation-driven loss of spatial fidelity [S1, unofficial].

## How it works
The method operates in two distinct phases: generating fixed activation patterns and optimizing inhibition patterns. It builds on the Dr.TVAM framework [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- Target geometry (mesh file).
- Vial geometry configuration.
- Projector and sensor configurations.
- Activation configuration file (defining 405 nm parameters).
- Inhibition configuration file (defining 365 nm parameters).
- Pre-computed 405 nm activation patterns (`.npz` file).
- Scaling factor $\alpha$ for the activation field.

**Outputs:**
- Optimized 365 nm inhibition patterns.
- Effective dose field $D_{\mathrm{eff}}(\mathbf{r})$.

### Data Structures
The implementation uses sparse matrices for diffusion operators and numpy arrays for fields and patterns.

**Laplacian Matrix (`no_flux_laplacian`):**
- **Type:** Sparse CSR matrix.
- **Shape:** $(N, N)$, where $N$ is the number of voxels where the `vial` mask is true.
- **Construction:**
    - Indices are mapped from 2D coordinates to 1D indices where `vial` is true.
    - Entries are `1.0` for neighbors and `-degree` for the center.
    - Values are divided by `pixel_size_mm**2`.

**Fields:**
- $D_{405}(\mathbf{r})$: Reconstructed activation field at 405 nm.
- $D_{365}(\mathbf{r})$: Reconstructed inhibition field at 365 nm.
- $D_{\mathrm{eff}}(\mathbf{r})$: Effective dose field.

### Steps

1.  **Load Scene Configuration:**
    - Parse configuration files for `target`, `vial`, `projector`, and `sensor`.
    - Validate required fields.
    - Load target mesh and compute bounding box.
    - Apply scaling and centering transforms based on configuration.
    - Construct sensor transform.

2.  **Generate Activation Patterns (Fixed):**
    - Use the Dr.TVAM framework to generate fixed 405 nm activation patterns.
    - Command: `drtvam examples/background_suppression/activation_config.json --backend cuda -D output=output/background_suppression/activation`
    - Save patterns to `output/background_suppression/activation/patterns.npz`.

3.  **Optimize Inhibition Patterns:**
    - Load the fixed activation patterns.
    - Hold the reconstructed activation field $D_{405}$ constant.
    - Run the antagonistic optimizer to find non-negative 365 nm projection values that minimize the error between the effective dose and the target, while suppressing background.
    - Command: `python antagonistic_tvam/optimize_antagonistic.py --activation-config ... --inhibition-config ... --activation-patterns ... --activation-scale 1.00 --backend cuda --output ...`

### Formula
The effective dose field is calculated as:

$$
D_{\mathrm{eff}}(\mathbf{r}) = \max \left[\alpha D_{405}(\mathbf{r}) - D_{365}(\mathbf{r}),\,0\right]
$$

Where:
- $D_{\mathrm{eff}}(\mathbf{r})$ is the effective dose field at position $\mathbf{r}$.
- $D_{405}(\mathbf{r})$ is the reconstructed activation field at 405 nm.
- $D_{365}(\mathbf{r})$ is the reconstructed inhibition field at 365 nm.
- $\alpha$ is a scaling factor for the activation field in the numerical objective.

### Parameters
- $\alpha$: Scaling factor for the activation field. Typical value: 1.00 [S1, unofficial].

### Simulation Loop (Diffusion and Reaction)
The core simulation updates active inhibitor and response fields over temporal substeps. The loop applies:
1.  Half-step diffusion.
2.  Inhibitor generation.
3.  Another half-step diffusion.
4.  Activation increments: consume inhibitor and accumulate response.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.sparse import csr_matrix

def no_flux_laplacian(vial_mask, pixel_size_mm):
    """
    Construct a sparse CSR matrix representing the discrete Laplacian operator
    on a boolean vial mask.
    
    Args:
        vial_mask: 2D boolean array indicating valid voxels.
        pixel_size_mm: Float, pixel size in mm.
        
    Returns:
        Sparse CSR matrix of shape (N, N) where N is the number of True values in vial_mask.
    """
    # Get indices where vial is true
    coords = np.argwhere(vial_mask)
    n_voxels = len(coords)
    
    # Map 2D coordinates to 1D indices
    # Assuming a simple linear mapping for demonstration; actual implementation
    # may use a more complex indexing scheme depending on the grid structure.
    # Here we assume a 2D grid for simplicity as per the note's description.
    # In a 3D volume, this would be extended.
    
    # Create a mapping from (x, y) to index
    index_map = np.full(vial_mask.shape, -1, dtype=int)
    for i, (x, y) in enumerate(coords):
        index_map[x, y] = i
        
    rows = []
    cols = []
    data = []
    
    for i, (x, y) in enumerate(coords):
        neighbors = []
        # Check 4-connected neighbors (up, down, left, right)
        for dx, dy in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
            nx, ny = x + dx, y + dy
            if 0 <= nx < vial_mask.shape[0] and 0 <= ny < vial_mask.shape[1] and vial_mask[nx, ny]:
                neighbors.append(index_map[nx, ny])
        
        degree = len(neighbors)
        rows.append(i)
        cols.append(i)
        data.append(-degree / (pixel_size_mm ** 2))
        
        for neighbor in neighbors:
            rows.append(i)
            cols.append(neighbor)
            data.append(1.0 / (pixel_size_mm ** 2))
            
    laplacian = csr_matrix((data, (rows, cols)), shape=(n_voxels, n_voxels))
    return laplacian

def simulate_diffusion_reaction(active_inhibitor, response, laplacian, dt, diffusion_coeff, inhibitor_gen_rate, activation_increment, inhibitor_consumption_rate):
    """
    Simulate diffusion and reaction over one time step.
    
    Args:
        active_inhibitor: 1D numpy array, current active inhibitor concentration.
        response: 1D numpy array, current response (polymerization) level.
        laplacian: Sparse CSR matrix, discrete Laplacian operator.
        dt: Float, time step size.
        diffusion_coeff: Float, diffusion coefficient.
        inhibitor_gen_rate: Float, rate of inhibitor generation.
        activation_increment: Float, amount of activation to process.
        inhibitor_consumption_rate: Float, rate of inhibitor consumption by activation.
        
    Returns:
        updated_active_inhibitor: 1D numpy array, updated active inhibitor concentration.
        updated_response: 1D numpy array, updated response level.
    """
    # Half-step diffusion
    # D_inhibitor(t + dt/2) = D_inhibitor(t) + dt/2 * D * laplacian * D_inhibitor(t)
    diffusion_term = diffusion_coeff * laplacian.dot(active_inhibitor)
    active_inhibitor_half = active_inhibitor + (dt / 2.0) * diffusion_term
    
    # Add inhibitor generation
    # D_inhibitor(t + dt/2) += dt/2 * inhibitor_gen_rate
    active_inhibitor_half += (dt / 2.0) * inhibitor_gen_rate
    
    # Another half-step diffusion
    # D_inhibitor(t + dt) = D_inhibitor(t + dt/2) + dt/2 * D * laplacian * D_inhibitor(t + dt/2)
    diffusion_term_half = diffusion_coeff * laplacian.dot(active_inhibitor_half)
    active_inhibitor = active_inhibitor_half + (dt / 2.0) * diffusion_term_half
    
    # Process activation increments
    # Consume inhibitor and accumulate response
    # Check if there is enough inhibitor to consume
    consumed = np.minimum(activation_increment, active_inhibitor * inhibitor_consumption_rate)
    active_inhibitor -= consumed
    response += consumed
    
    # Clamp negative values to zero
    active_inhibitor = np.maximum(active_inhibitor, 0.0)
    response = np.maximum(response, 0.0)
    
    # Check for negative values due to numerical errors
    if active_inhibitor.min() < -1e-12:
        raise RuntimeError("Negative values detected in active inhibitor field.")
    
    # Clamp very small negative values to zero
    active_inhibitor[abs(active_inhibitor) < 1e-14] = 0.0
    
    return active_inhibitor, response

def optimize_antagonistic_tvam(target, activation_patterns, activation_config, inhibition_config, alpha=1.00):
    """
    Optimize inhibition patterns for antagonistic TVAM.
    
    Args:
        target: Target geometry or dose field.
        activation_patterns: Pre-computed 405 nm activation patterns.
        activation_config: Configuration for activation.
        inhibition_config: Configuration for inhibition.
        alpha: Scaling factor for activation field.
        
    Returns:
        optimized_inhibition_patterns: Optimized 365 nm inhibition patterns.
    """
    # Load scene and setup
    # ... (Implementation details from Dr.TVAM framework)
    
    # Reconstruct activation field D_405 from activation_patterns
    D_405 = reconstruct_field(activation_patterns, activation_config)
    
    # Initialize inhibition patterns (non-negative)
    inhibition_patterns = initialize_inhibition_patterns(inhibition_config)
    
    # Optimize inhibition patterns
    # The optimizer minimizes the error between D_eff and target
    # D_eff = max(alpha * D_405 - D_365, 0)
    # where D_365 is reconstructed from inhibition_patterns
    
    # ... (Optimization loop using differentiable framework)
    
    return optimized_inhibition_patterns
```

### Edge Cases and Pitfalls
- **Negative Values in Diffusion:** During diffusion simulation, negative values can occur if the time step or diffusion coefficient is too large. The code raises a `RuntimeError` if `updated.min() < -1e-12` and clamps values with absolute magnitude less than `1e-14` to zero [S1, unofficial].
- **Boundary Conditions:** The `no_flux_laplacian` function assumes no-flux boundary conditions by only considering neighbors within the `vial` mask.

## Operating characteristics
- **Time Complexity:** Depends on the number of projections, voxels, and optimization iterations. The diffusion simulation is $O(N)$ per time step, where $N$ is the number of voxels. The optimization involves solving a large-scale inverse problem, which can be computationally expensive.
- **Memory Complexity:** Proportional to the number of voxels and projections. The Laplacian matrix is sparse, so memory usage is manageable for large grids.
- **Strengths:**
    - Suppresses tomographic background dose.
    - Confinement of individual features.
    - Definition of complete 3D geometries through inhibition.
    - Mitigation of diffusion- and attenuation-driven loss of spatial fidelity.
- **Limitations:**
    - Requires two wavelength channels (405 nm and 365 nm).
    - More complex optimization process compared to standard TVAM.
    - Sensitive to the choice of scaling factor $\alpha$.

## Background
Antagonistic Dual-Wavelength TVAM was developed to address the limitations of standard TVAM, particularly the accumulation of unwanted background dose outside the target area due to overlapping projections [S1, unofficial]. It extends the standard TVAM Integrator by adding a second wavelength channel for inhibition, allowing for better control over the polymerization process [S1, unofficial]. The method is implemented using the open-source Dr.TVAM framework [S1, unofficial].

## Sources
[S1] EPFL-LAPD/Antagonistic-Dual-Wavelength-Tomographic-Volumetric-Additive-Manufacturing, https://github.com/EPFL-LAPD/Antagonistic-Dual-Wavelength-Tomographic-Volumetric-Additive-Manufacturing, fetched 2026-10-26 (unofficial)
