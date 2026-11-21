# Cloud Radiative Transfer Forward Model Setup

Also called: RTE Solver Configuration, Radiative Transfer Grid and Medium Setup

## Overview
This method configures the geometric grid, optical property tables, and solver parameters required to solve the Radiative Transfer Equation (RTE) for cloudy atmospheres. It establishes the spatial discretization and the microphysical scattering models (Mie theory) that feed into the numerical solver. It differs from sibling methods by focusing on the *setup* of the medium and boundary conditions rather than the iterative solution of the transport equation itself. It serves as the prerequisite configuration for [Discrete Ordinates Method](1046-discrete-ordinates-method.md) and [Principal-Ordinates Propagation](4121-principal-ordinates-propagation.md).

## How it works
The setup process constructs a 3D radiative transfer grid, generates wavelength-dependent optical properties based on cloud microphysics, and initializes the solver with specific boundary and source conditions.

**Inputs:**
*   Cloud scatterer data: 3D arrays for coordinates $x, y, z$ and scattering properties.
*   Mie lookup tables: Pre-computed scattering matrices for mono-disperse clouds.
*   Size distribution function: Gamma distribution parameters.
*   Microphysical ranges: Effective radius ($r_{eff}$) and effective variance ($v_{eff}$) grids.
*   Solver configuration: Numerical parameters, surface albedo, solar source definition.

**Outputs:**
*   `rte_grid`: A structured 3D grid object defining the spatial domain.
*   `medium`: An optical property object containing extinction, scattering, and phase functions.
*   `solver`: An initialized RTE solver instance ready for propagation.

**Data Structures:**
1.  **RTE Grid (`rte_grid`)**:
    *   Constructed from coordinate differentials and dimensions.
    *   Fields: `dx` (x-spacing), `nx` (x-dim), `dy` (y-spacing), `ny` (y-dim), `z_coords` (z-coordinates).
    *   Layout: The z-coordinates are appended with a zero offset: $z_{grid} = [0, z_0, z_1, \dots]$.
2.  **Optical Property Generator (`optical_property_generator`)**:
    *   Type: `'cloud'`.
    *   Inputs: Mie tables, size distribution function, $r_{eff}$ range, $v_{eff}$ range.
    *   Output: Interpolated optical coefficients ($\sigma_t, \sigma_s, \Phi$) for the grid cells.
3.  **Solver Configuration**:
    *   Surface: Lambertian reflector with albedo $\alpha = 0.0$.
    *   Source: Solar source defined by wavelength $\lambda$, solar zenith cosine $\mu_0$, and solar azimuth $\phi_0$.
    *   Stokes: Number of Stokes parameters $N_S = 1$ (unpolarized).

**Numbered Steps:**

1.  **Construct the Radiative Transfer Grid**:
    Calculate grid spacing from the cloud scatterer data.
    $$ dx = \frac{dx_{data}}{1} $$
    $$ dy = \frac{dy_{data}}{1} $$
    $$ z_{grid} = [0] \cup z_{data} $$
    Initialize `rte_grid` with $(dx, N_x, dy, N_y, z_{grid})$.

2.  **Generate Optical Properties**:
    Define the microphysical parameter ranges:
    $$ r_{eff} \in \text{linspace}(5.0, 30.0, 30) \, \mu m $$
    $$ v_{eff} \in \text{linspace}(0.03, 0.2, 9) $$
    Instantiate `optical_property_generator` using:
    *   Cloud type: `'cloud'`
    *   Mie tables: `mie_mono_tables`
    *   Size distribution: `size_distribution_function` (Gamma)
    *   Ranges: $r_{eff}$ and $v_{eff}$ as defined above.
    Apply this generator to `rte_grid` to produce the `medium` object.

3.  **Configure the RTE Solver**:
    Define the solar source parameters:
    *   Wavelength: $\lambda$ (specific value from context, e.g., visible spectrum).
    *   Solar zenith cosine: $\mu_0 = \cos(\theta_{sun})$.
    *   Solar azimuth: $\phi_0 = 140^\circ$.
    Define the surface:
    *   Type: Lambertian.
    *   Albedo: $\alpha = 0.0$.
    Initialize the solver:
    $$ \text{Solver} = \text{RTE}(\text{numerical\_params}, \text{surface}, \text{source}, \text{medium}, N_S=1) $$
    Add the solver to the solver dictionary for the given wavelength.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def setup_cloud_rte(cloud_scatterer, mie_mono_tables, size_distribution_function, 
                    wavelength, solar_mu, solar_az, numerical_params):
    """
    Sets up the Cloud Radiative Transfer Forward Model.
    
    Args:
        cloud_scatterer: Object containing x, y, z data arrays.
        mie_mono_tables: Pre-computed Mie scattering tables.
        size_distribution_function: Gamma distribution function object.
        wavelength: Solar wavelength (float).
        solar_mu: Cosine of solar zenith angle (float).
        solar_az: Solar azimuth angle in degrees (float).
        numerical_params: Configuration dict for the RTE solver.
        
    Returns:
        solver: Initialized RTE solver instance.
    """
    
    # Step 1: Construct the radiative transfer grid
    # Calculate spacing from data differentials
    dx = cloud_scatterer.x.diff('x')[0]
    dy = cloud_scatterer.y.diff('y')[0]
    
    # Get dimensions
    nx = cloud_scatterer.x.data.size
    ny = cloud_scatterer.y.data.size
    
    # Construct z-coordinates, prepending 0
    z_coords = np.append(0, cloud_scatterer.z.data)
    
    # Create grid object
    rte_grid = make_grid(dx, nx, dy, ny, z_coords)
    
    # Step 2: Generate optical properties
    # Define microphysical ranges
    reff_range = np.linspace(5.0, 30.0, 30)
    veff_range = np.linspace(0.03, 0.2, 9)
    
    # Create optical property generator
    optical_property_generator = OpticalPropertyGenerator(
        cloud_type='cloud',
        mie_tables=mie_mono_tables,
        size_dist_func=size_distribution_function,
        reff=reff_range,
        veff=veff_range
    )
    
    # Apply generator to grid to create medium
    medium = optical_property_generator.generate(rte_grid)
    
    # Step 3: Configure and add solver
    # Define surface (Lambertian, albedo 0.0)
    surface = LambertianSurface(albedo=0.0)
    
    # Define solar source
    source = SolarSource(
        wavelength=wavelength,
        mu0=solar_mu,
        phi0=solar_az
    )
    
    # Initialize RTE solver
    # num_stokes=1 for unpolarized light
    solver = RTE(
        numerical_params=numerical_params,
        surface=surface,
        source=source,
        medium=medium,
        num_stokes=1
    )
    
    # Add to solver dictionary (implementation detail of the framework)
    solvers_dict = {}
    solvers_dict[wavelength] = solver
    
    return solver
```

**Edge Cases and Pitfalls:**
*   **Precision**: The effective radius and variance ranges are discretized. Coarse discretization may miss microphysical variations.
*   **Boundary Conditions**: The surface albedo is fixed at 0.0 (blackbody/absorbing). If a reflective surface is required, this must be changed, affecting the boundary term in the RTE.
*   **Polarization**: `num_stokes=1` assumes unpolarized light. For polarized radiative transfer, `num_stokes` must be set to 4, requiring full Mueller matrix Mie tables.

## Subtopics

### Discrete Ordinates Method
Solves the RTE by discretizing the angular domain into a finite set of directions. Choose when high angular accuracy is needed for moderate optical depths. [Link](1046-discrete-ordinates-method.md)

### Principal-Ordinates Propagation
A specialized propagation method for principal ordinates. Choose for efficient forward scattering approximation. [Link](4121-principal-ordinates-propagation.md)

### Multiple Scattering Model (Integral Equation)
Solves the integral form of the RTE. Choose when multiple scattering dominates and angular discretization is too expensive. [Link](1177-multiple-scattering-model-integral-equation.md)

## Operating characteristics
*   **Time Complexity**: Setup is $O(N_{grid} \cdot N_{wavelengths} \cdot N_{microphysical\_bins})$. The actual solve time depends on the chosen solver (e.g., discrete ordinates scales with $N_{angles} \cdot N_{steps}$).
*   **Memory Complexity**: $O(N_{grid} \cdot N_{wavelengths})$ for storing optical properties. Mie tables add $O(N_{angles} \cdot N_{wavelengths})$.
*   **Strengths**: Modular setup allows easy swapping of microphysical models (Mie vs. Rayleigh) and surface conditions.
*   **Limitations**: Dependent on pre-computed Mie tables. Accuracy is limited by the discretization of $r_{eff}$ and $v_{eff}$.
*   **When to use**: When setting up a forward model for cloud radiative transfer in atmospheric remote sensing or climate modeling.
*   **When not to use**: When real-time rendering is required without pre-computation, or when the medium is not cloud-like (e.g., fog, smoke) requiring different scattering models.

## Background
This method is derived from the `hybridCT` framework, which combines hybrid neural network approaches with traditional cloud radiative transfer solvers. It was developed to efficiently generate training data for machine learning models predicting cloud optical depths. The setup follows standard atmospheric radiative transfer principles, using Mie theory for spherical cloud droplets.

## Sources
[S1] lforster/hybridCT, https://github.com/lforster/hybridCT, fetched 2026-10-27 (unofficial)
