# Light Cone Construction

Also called: Light cone projection, Light rays

## Overview
Light Cone Construction is a method for generating synthetic astronomical observations (imaging or spectroscopy) from cosmological simulation data by stacking discrete redshift slices into a continuous line-of-sight volume [S1]. It solves the problem of creating realistic light cones from periodic simulation boxes that do not natively span the required redshift range, while minimizing redundant sampling of the same physical structures [S1]. It is distinguished from standard volume rendering by its specific handling of cosmological distance metrics, periodic domain wrapping, and the statistical generation of unique, non-overlapping sightlines [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Observer redshift ($z_{obs}$).
*   Redshift interval $[z_{min}, z_{max}]$.
*   Simulation dataset stack (periodic boxes at various redshifts).
*   Cosmological parameters: $\Omega_M$ (matter density), $\Omega_k$ (curvature density), $\Omega_\Lambda$ (cosmological constant density) [S1].
*   For Light Cone Projections: Angular field of view ($\theta$), image resolution, and maximum allowed common volume between projections [S1].
*   For Light Rays: Number of sightlines and maximum allowed common volume [S1].

**Outputs:**
*   A stack of datasets ordered by redshift.
*   For Projections: A 2D image or data cube representing the integrated signal along the line of sight.
*   For Rays: A list of ray segments, each containing intersected pixels, path lengths ($dl$), and assigned redshifts [S1].

### Data Structures

**Dataset Stack:**
A list of simulation data objects, ordered from highest redshift ($z_{max}$) to lowest ($z_{min}$). Each object represents a simulation box at a specific redshift [S1].

**Ray Segment (for Light Rays):**
An array containing:
*   Indices of all pixels intersected by the ray within the dataset.
*   The path length $dl$ through each intersected pixel [S1].

**Projection Object (for Light Cone Projections):**
A pre-computed projection of the *entire* lateral domain of a dataset. This allows multiple light cone projections to sample different regions of the same box without re-projecting the full volume [S1].

### Formulas

**Expansion Factor $E(z)$:**
$$ E(z) \equiv \sqrt{\Omega_M (1 + z)^3 + \Omega_k (1 + z)^2 + \Omega_\Lambda} $$
where $\Omega_M, \Omega_k, \Omega_\Lambda$ are the energy density contributions of matter, curvature, and the cosmological constant, respectively [S1].

**Angular Size Distance $D_A$:**
The width of the sampled region in the image plane for a field of view $\theta$ is $(1+z) D_A \theta$. $D_A$ is defined as:
$$
D_A = \begin{cases} 
\frac{D_H}{\sqrt{|\Omega_k|}} \frac{\sinh(\sqrt{|\Omega_k|} D_C)}{(1+z)} & \text{for } \Omega_k > 0 \\
\frac{D_C}{1+z} & \text{for } \Omega_k = 0 \\
\frac{D_H}{\sqrt{|\Omega_k|}} \frac{\sin(\sqrt{|\Omega_k|} D_C)}{(1+z)} & \text{for } \Omega_k < 0
\end{cases}
$$
where $D_C$ is the comoving distance and $D_H$ is the Hubble distance [S1].

**Column Density:**
For a ray segment, the column density $N$ contributed by a physical density $\rho$ over a path length element $dl$ is:
$$ N = \rho \times dl $$
[S1].

### Parameters
*   **Fraction of Box Length:** The user may specify a fraction of the total simulation box length to calculate the maximum allowable redshift step $\delta z$ between datasets, rather than using the full box length [S1].
*   **Maximum Common Volume:** A user-specified limit on the volume overlap between different light cone projections or rays to ensure statistical independence [S1].

### Steps

1.  **Select Datasets:** Use time series machinery to select simulation datasets appropriate for the requested redshift interval $[z_{min}, z_{max}]$ [S1].
2.  **Build Stack:** Initialize an empty stack. Add the dataset with the redshift closest to $z_{max}$ as the first element [S1].
3.  **Iterate Downward:**
    a. Calculate $\delta z$, the redshift interval corresponding to the length of the simulation box. Note that $\delta z$ varies with redshift [S1].
    b. Determine the target redshift for the next dataset: $z_{target} = z_{current} - \delta z$ [S1].
    c. Select the dataset whose redshift is closest to, but no less than, $z_{target}$ [S1].
    d. Add this dataset to the stack.
    e. Repeat until the lower limit $z_{min}$ is reached [S1].
    *Note:* The user may constrain the maximum $\delta z$ to be a fraction of the full box length to ensure smoother transitions [S1].
4.  **Generate Sightlines (Randomization):**
    a. For each dataset in the stack, choose a random projection axis and center for the projected region. This exploits the periodicity of the computational domain to minimize sampling the same structures multiple times [S1].
    b. For Light Rays, assign a random orientation to each ray segment [S1].
5.  **Compute Projections/Rays:**
    a. **For Projections:** Project the full lateral domain of each dataset. The comoving radial distance determines the fraction of the box used in the line of sight. The image width is determined by $(1+z) D_A \theta$ [S1].
    b. **For Rays:** For each ray segment, identify all intersected pixels and calculate the path length $dl$ through each [S1].
6.  **Assign Redshifts:** For each length element in a ray, assign a unique redshift based on the dataset's redshift and the point's position along the ray, assuming a smooth Hubble flow [S1].
7.  **Ensure Uniqueness (Optional):** If generating multiple projections/rays, use a unique solution generator to find realizations with a user-specified maximum common volume. This generator first attempts to vary only the lateral randomization (reusing projection objects) before attempting fully unique randomization [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_light_cone(observer_redshift, z_min, z_max, datasets, cosmology):
    """
    Constructs a stack of datasets for light cone observation.
    
    Args:
        observer_redshift: Redshift of the observer (typically 0).
        z_min: Lower bound of redshift interval.
        z_max: Upper bound of redshift interval.
        datasets: List of available simulation datasets, each with a 'redshift' attribute.
        cosmology: Object containing Omega_M, Omega_k, Omega_Lambda, H0.
        
    Returns:
        stack: List of datasets ordered from z_max to z_min.
    """
    # Step 1: Preliminary selection of datasets
    # Filter datasets that fall within or near the requested interval
    relevant_datasets = [d for d in datasets if d.redshift >= z_min and d.redshift <= z_max]
    
    # Sort by redshift descending
    relevant_datasets.sort(key=lambda d: d.redshift, reverse=True)
    
    stack = []
    current_z = z_max
    
    # Step 2: Add first dataset
    # Find dataset closest to z_max
    first_dataset = min(relevant_datasets, key=lambda d: abs(d.redshift - z_max))
    stack.append(first_dataset)
    current_z = first_dataset.redshift
    
    # Step 3: Iterate down to z_min
    while current_z > z_min:
        # Calculate delta_z for the current redshift
        # delta_z corresponds to the length of the simulation box
        # Note: delta_z is not constant with redshift
        delta_z = calculate_delta_z(current_z, cosmology, box_length)
        
        # Determine target redshift
        target_z = current_z - delta_z
        
        # Select next dataset: closest to, but no less than, target_z
        # Filter candidates that are >= target_z
        candidates = [d for d in relevant_datasets if d.redshift >= target_z]
        
        if not candidates:
            break # Should not happen if datasets are dense enough
            
        # Choose the one closest to target_z
        next_dataset = min(candidates, key=lambda d: abs(d.redshift - target_z))
        
        # Avoid duplicates if the same dataset is selected multiple times
        if next_dataset not in stack:
            stack.append(next_dataset)
            current_z = next_dataset.redshift
        else:
            # If the closest valid dataset is already in the stack, 
            # we might need to pick the next closest or handle overlap.
            # The source implies a strict selection process.
            # For implementation robustness, we pick the next closest unique one.
            remaining_candidates = [d for d in candidates if d not in stack]
            if remaining_candidates:
                next_dataset = min(remaining_candidates, key=lambda d: abs(d.redshift - target_z))
                stack.append(next_dataset)
                current_z = next_dataset.redshift
            else:
                break

    return stack

def calculate_delta_z(z, cosmology, box_length):
    """
    Calculates the redshift interval corresponding to the comoving distance
    of the simulation box length.
    """
    # Comoving distance calculation depends on cosmology
    # This is a placeholder for the specific integral of 1/E(z)
    # D_C = integral from 0 to z of c/H(z) dz
    # delta_z is derived from the box length in comoving coordinates
    # The source notes that delta_z is not constant with redshift.
    # Implementation detail: Calculate comoving distance for box_length at z.
    # Then find dz such that D_C(z+dz) - D_C(z) = box_length_comoving
    # For simplicity in this pseudocode, we assume a helper exists.
    return compute_redshift_step_from_comoving_distance(box_length, z, cosmology)

def generate_light_ray(dataset, num_rays, max_common_volume):
    """
    Generates light rays for a single dataset in the stack.
    """
    rays = []
    for _ in range(num_rays):
        # Random orientation for the ray segment
        direction = random_unit_vector()
        origin = random_point_in_box()
        
        # Intersect ray with the dataset's grid
        # Returns array of intersected pixels and path lengths dl
        intersections = intersect_ray_with_grid(origin, direction, dataset.grid)
        
        ray_segment = {
            'intersections': intersections, # Array of (pixel_index, dl)
            'direction': direction,
            'origin': origin,
            'redshift': dataset.redshift # Base redshift for this segment
        }
        rays.append(ray_segment)
        
    return rays

def assign_redshifts(ray_segment, dataset_redshift, position_along_ray):
    """
    Assigns unique redshift to each length element assuming smooth Hubble flow.
    """
    # For each intersection in ray_segment['intersections']:
    #   Calculate physical distance from observer
    #   Map distance to redshift using cosmological model
    #   This allows creation of synthetic spectra with properly redshifted lines
    pass
```

### Edge Cases and Pitfalls
*   **Non-constant $\delta z$:** The redshift interval corresponding to the simulation box length changes with redshift. Failing to recalculate $\delta z$ at each step will lead to incorrect stacking [S1].
*   **Periodicity:** The method relies on the periodicity of the computational domain. If the simulation is not periodic, the randomization of projection axes and centers may result in discontinuities or artifacts at the boundaries of the stacked boxes [S1].
*   **Sampling Redundancy:** Without randomizing the projection axis and center for each dataset, the same large-scale structures may be sampled multiple times, biasing statistical analyses [S1].
*   **Common Volume:** If the maximum common volume is not specified or set too high, multiple light cone projections may sample the same volume, violating the assumption of independent realizations [S1].

## Operating characteristics

*   **Time Complexity:** The initial dataset stacking is linear with respect to the number of datasets in the stack. Projection of the full lateral domain is performed once per dataset, making subsequent light cone projections computationally cheap (only sampling different regions). Ray intersection calculations are proportional to the number of rays and the complexity of the grid [S1].
*   **Memory Complexity:** Requires storing the full lateral projection for each dataset in the stack if multiple light cone projections are to be generated efficiently. This can be memory-intensive for large boxes [S1].
*   **Strengths:** Efficiently generates multiple independent sightlines from a limited set of simulation boxes. The pre-computation of full lateral projections allows for rapid generation of statistical samples [S1].
*   **Limitations:** Relies on the assumption of a smooth Hubble flow for redshift assignment along rays, which may not capture local velocity distortions (redshift-space distortions) unless explicitly modeled [S1]. The quality depends on the density of available simulation datasets at different redshifts [S1].
*   **When to Use:** When generating synthetic observations (imaging or spectroscopy) from cosmological simulations that do not span the full redshift range of interest, or when statistical independence of multiple sightlines is required [S1].
*   **When Not to Use:** When high-fidelity modeling of local velocity fields or non-periodic boundary conditions is critical, or when computational resources are insufficient to store full lateral projections for all datasets [S1].

## Background

Light Cone Construction was implemented in the `yt` analysis toolkit, building on earlier work by Hallman et al. (2007, 2009) [S1]. The method leverages `yt`'s projection machinery to handle the complexities of projecting data from periodic simulation boxes onto a light cone [S1]. It is primarily used in astrophysics for creating synthetic surveys of the universe, such as quasar absorption line studies or galaxy distribution maps, allowing researchers to compare simulation predictions directly with observational data [S1].

## Sources

[S1] YT: A Multi-Code Analysis Toolkit for Astrophysical Simulation Data, https://digital.library.unt.edu/ark:/67531/metadc834104/?q=fluid simulation volume rendering, fetched 2026-10-28
