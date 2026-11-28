# Neural Plasma Field Ray Casting with LUT Emissivity

Also called: Neural Plasma Field Ray Casting, Ray Sampling and Rendering Geometry

## Overview
A forward rendering pipeline that casts rays through a normalized axis-aligned bounding box (AABB) to sample a neural plasma field and compute emissivity via a precomputed Look-Up Table (LUT) [S1, unofficial]. It solves the problem of synthesizing coronal images from neural network predictions of density and temperature, bridging the gap between implicit field representations and physical radiative transfer [S1, unofficial]. It differs from its parent [Emission-Absorption Ray Marching](3171-emission-absorption-ray-marching.md) by explicitly handling the conversion of neural field outputs (log-density, log-temperature) into physical emissivity values using a specialized LUT, rather than relying on a fixed analytic emission model [S1, unofficial]. It differs from [Volumetric Liquid Rendering](4017-volumetric-liquid-rendering.md) and [Energy-based Volume Rendering (Energy Engine)](8218-energy-based-volume-rendering-energy-engine.md) by targeting solar coronal physics with specific coordinate transformations and atomic-line emissivity tables [S1, unofficial].

## How it works
**Inputs:**
*   Neural plasma field function $f_\theta$, which takes 3D Cartesian coordinates and outputs predictions for $\log_{10} n_e$ and $\log_{10} T_e$ [S1, unofficial].
*   Emissivity Look-Up Table (LUT), a 3D tensor mapping $(\log_{10} n_e, \log_{10} T_e, \log_{10} r)$ to scalar Stokes-I emissivity [S1, unofficial].
*   Camera parameters: heliocentric distance $d \approx 215 R_\odot$, latitude $0^\circ$, field of view (FOV) $[-3, 3] R_\odot$ [S1, unofficial].
*   Step size $\Delta s = 1/256$ in AABB coordinates [S1, unofficial].

**Outputs:**
*   Rendered image with intensity in units of $\text{photons s}^{-1} \text{cm}^{-2} \text{arcsec}^{-2}$ [S1, unofficial].

**Data Structures:**
1.  **Neural Plasma Field $f_\theta$**: A neural network (e.g., MLP) taking input $\mathbf{x} \in \mathbb{R}^3$ (AABB coordinates) and outputting a vector $[\log_{10} n_e, \log_{10} T_e]$ [S1, unofficial].
2.  **Emissivity LUT**: A PyTorch tensor of shape $(128, 64, 32)$ representing the grid in $(\log_{10} n_e, \log_{10} T_e, \log_{10} r)$ space. The radial LUT spans approximately $1.001$ to $10 R_\odot$ [S1, unofficial]. The tensor is implemented using `torch.nn.functional.grid_sample` for trilinear interpolation [S1, unofficial].
3.  **Coordinate Systems**:
    *   **Physical**: $[-30, 30]^3 R_\odot$ [S1, unofficial].
    *   **AABB**: $[-1, 1]^3$, obtained by dividing physical coordinates by $30 R_\odot$ [S1, unofficial].
    *   **Spherical**: $(r, \theta, \phi)$, where $r = \|\mathbf{x}_{\text{phys}}\|$ [S1, unofficial].

**Steps:**

1.  **Camera Setup**: For each pixel, construct a ray originating from the observer at distance $d \approx 215 R_\odot$ pointing toward the origin. The ray spans the FOV $[-3, 3] R_\odot$ [S1, unofficial].
2.  **Coronagraph Masking**: Calculate the impact parameter of the ray. If the impact parameter is $< 1 R_\odot$, discard the ray (mask it) to mimic a coronagraph and exclude the solar disk [S1, unofficial].
3.  **Ray-AABB Intersection**: Compute the near ($t_{\text{near}}$) and far ($t_{\text{far}}$) intersection points of the ray with the normalized AABB $[-1, 1]^3$ [S1, unofficial].
4.  **Sampling**: Generate uniform sample points along the ray segment $[t_{\text{near}}, t_{\text{far}}]$ with step size $\Delta s = 1/256$ in AABB coordinates. The number of samples $N$ depends on the ray's intersection length [S1, unofficial].
5.  **Neural Field Query**: For each sample point $\mathbf{x}_{\text{AABB}}$:
    *   Convert to physical coordinates: $\mathbf{x}_{\text{phys}} = \mathbf{x}_{\text{AABB}} \times 30 R_\odot$ [S1, unofficial].
    *   Query the neural field: $[\log_{10} n_e, \log_{10} T_e] = f_\theta(\mathbf{x}_{\text{AABB}})$ [S1, unofficial].
    *   Compute heliocentric radius: $r = \|\mathbf{x}_{\text{phys}}\|$ [S1, unofficial].
6.  **LUT Emissivity Lookup**:
    *   Compute $\log_{10} r$ [S1, unofficial].
    *   Query the LUT at $(\log_{10} n_e, \log_{10} T_e, \log_{10} r)$ [S1, unofficial].
    *   **Clamping**: If any coordinate is outside the LUT bounds, clamp to the nearest boundary value before interpolation [S1, unofficial].
    *   Let $\epsilon_{\text{raw}}$ be the interpolated value in units of $\text{erg s}^{-1} \text{cm}^{-3} \text{sr}^{-1}$ [S1, unofficial].
7.  **Unit Conversion**: Convert $\epsilon_{\text{raw}}$ to photon counts per arcsec$^2$:
    *   Convert from erg to photon counts [S1, unofficial].
    *   Convert from $\text{sr}^{-1}$ to $\text{arcsec}^{-2}$ [S1, unofficial].
    *   Let $\epsilon_c$ be the result in $\text{photons s}^{-1} \text{cm}^{-3} \text{arcsec}^{-2}$ [S1, unofficial].
8.  **Differential Length**: Compute the differential path length $\Delta l$ for the sample segment. Scale the AABB step size $\Delta s$ to physical units ($R_\odot$) and then to cm [S1, unofficial].
9.  **Integration**: Perform a midpoint Riemann sum integration along the ray:
    $$ I = \sum_{i=1}^{N} \epsilon_c(\mathbf{x}_i) \cdot \Delta l_i $$
    The final intensity $I$ is in units of $\text{photons s}^{-1} \text{cm}^{-2} \text{arcsec}^{-2}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def intersect_aabb(origins, directions, aabb_min=-1.0, aabb_max=1.0):
    """Standard slab method for ray-AABB intersection."""
    inv_dirs = 1.0 / directions
    t1 = (aabb_min - origins) * inv_dirs
    t2 = (aabb_max - origins) * inv_dirs
    t_near = torch.max(torch.min(t1, t2), dim=1).values
    t_far = torch.min(torch.max(t1, t2), dim=1).values
    # Ensure valid intersection
    t_near = torch.where(t_near > t_far, t_far, t_near)
    t_far = torch.where(t_near > t_far, t_near, t_far)
    return t_near, t_far

def calculate_impact_parameter(origins, directions):
    """Distance from ray to origin (solar center)."""
    # Impact parameter = ||O - (O·D)D||
    proj = torch.sum(origins * directions, dim=1, keepdim=True)
    closest_point = origins - proj * directions
    return torch.norm(closest_point, dim=1)

def convert_emissivity_units(emissivity_raw):
    """Convert erg/s/cm^3/sr to photons/s/cm^2/arcsec^2 [G]."""
    # Constants depend on specific spectral lines (e.g., Fe XIII, Si IX) [S1, unofficial]
    PHOTON_ENERGY_FACTOR = 1.0  # Placeholder for h*c/lambda conversion [G]
    SR_TO_ARCSEC2 = (180 * 3600 / torch.pi) ** 2
    return emissivity_raw * PHOTON_ENERGY_FACTOR * SR_TO_ARCSEC2

def query_lut_3d(lut_tensor, coords_grid):
    """Query 3D LUT using PyTorch grid_sample with clamping backstop [S1, unofficial]."""
    # lut_tensor shape: (D, H, W) = (128, 64, 32)
    # coords_grid shape: (N, 3) in [-1, 1]
    N = coords_grid.shape[0]
    # grid_sample expects grid shape (N, D_out, H_out, W_out, 3)
    grid = coords_grid.unsqueeze(0).unsqueeze(1).unsqueeze(2)  # (1, N, 1, 1, 3)
    input_lut = lut_tensor.unsqueeze(0).unsqueeze(0)  # (1, 1, 128, 64, 32)
    output = F.grid_sample(input_lut, grid, align_corners=False, mode='bilinear', padding_mode='zeros')
    # Output shape: (1, 1, N, 1, 1) -> squeeze to (N,)
    return output.squeeze()

def render_neural_plasma_field(f_theta, lut_tensor, lut_bounds, camera_params, step_size=1/256):
    """
    Render a single view using the Neural Plasma Field and LUT Emissivity.
    
    Args:
        f_theta: Neural network function returning [log10_ne, log10_Te] for AABB coords.
        lut_tensor: PyTorch tensor of shape (128, 64, 32) for LUT.
        lut_bounds: Tuple of (min, max) for each LUT axis (log10_ne, log10_Te, log10_r).
        camera_params: Dict with 'distance', 'fov', 'resolution'.
        step_size: Float, step size in AABB coordinates.
        
    Returns:
        image: Tensor of rendered intensities.
    """
    # 1. Camera Setup
    # Observer at distance d, pointing to origin. FOV [-3, 3] R_sun.
    # Generate ray origins and directions for each pixel.
    ray_origins, ray_directions = generate_perspective_rays(camera_params)
    
    # 2. Ray-AABB Intersection
    # AABB is [-1, 1]^3.
    t_near, t_far = intersect_aabb(ray_origins, ray_directions, aabb_min=-1.0, aabb_max=1.0)
    
    # 3. Coronagraph Masking
    # Calculate impact parameter for each ray.
    # Impact parameter < 1 R_sun -> mask out.
    impact_params = calculate_impact_parameter(ray_origins, ray_directions)
    valid_mask = impact_params >= 1.0
    
    # 4. Sampling
    # Generate sample points along the ray segment [t_near, t_far]
    # Number of samples N depends on intersection length.
    # t_samples = torch.linspace(t_near, t_far, steps=N)
    # However, step_size is fixed in AABB coords.
    # We calculate N based on length / step_size.
    # For simplicity, assume we generate samples at uniform intervals of step_size.
    
    # 5. Integration Loop
    # Initialize intensity accumulator
    intensities = torch.zeros_like(ray_origins[..., 0])
    
    # Convert step_size from AABB to physical (R_sun)
    # AABB range is 2 units (-1 to 1), Physical range is 60 R_sun (-30 to 30).
    # Scale factor: 30 R_sun per AABB unit.
    scale_factor = 30.0 
    delta_s_physical = step_size * scale_factor # in R_sun
    
    # Convert R_sun to cm (approx 6.957e10 cm)
    R_sun_to_cm = 6.957e10
    delta_l_cm = delta_s_physical * R_sun_to_cm
    
    # Process each valid ray
    for i in range(len(ray_origins)):
        if not valid_mask[i]:
            continue
            
        t_start = t_near[i]
        t_end = t_far[i]
        
        # Calculate number of samples
        length_aabb = t_end - t_start
        num_samples = int(length_aabb / step_size)
        
        # Generate sample positions in AABB
        # Midpoint rule: samples are at t_start + (k + 0.5) * step_size
        sample_indices = torch.arange(num_samples)
        t_samples = t_start + (sample_indices + 0.5) * step_size
        
        # Get 3D coordinates in AABB
        # x_aabb = ray_origins[i] + t_samples[:, None] * ray_directions[i]
        # Note: ray_directions must be normalized for t to represent distance in AABB units?
        # The source says "step size 1/256 (AABB coordinates)". This implies t is in AABB units.
        # So ray_directions should be normalized such that |direction| = 1 in AABB space?
        # Or t is just the parameter. Let's assume t is the parameter along the ray vector.
        # If ray_directions are normalized to unit length in physical space, we need to adjust.
        # However, the source says "compute near and far bounds as intersections with the AABB".
        # Standard ray-AABB intersection returns t values.
        # Let's assume ray_directions are normalized to unit length in the coordinate system of t.
        # If not, we must normalize.
        
        # Re-normalize direction to unit length in AABB space if necessary.
        # Assuming ray_directions are already unit vectors in the coordinate system of t.
        x_aabb = ray_origins[i] + t_samples[:, None] * ray_directions[i]
        
        # 6. Neural Field Query
        # Query f_theta for log10_ne and log10_Te
        # f_theta takes x_aabb.
        log10_ne, log10_Te = f_theta(x_aabb) # Shape: (num_samples, 2)
        
        # 7. LUT Emissivity Lookup
        # Compute radius r in physical units
        # x_phys = x_aabb * 30 R_sun
        x_phys = x_aabb * scale_factor
        r = torch.norm(x_phys, dim=1) # Shape: (num_samples,)
        log10_r = torch.log10(r)
        
        # Prepare LUT query coordinates
        # LUT axes: (log10_ne, log10_Te, log10_r)
        # Grid coordinates for grid_sample are typically normalized to [-1, 1] or [0, 1].
        # We need to map log10_ne, log10_Te, log10_r to grid indices.
        
        # Clamp to LUT bounds
        log10_ne_clamped = torch.clamp(log10_ne, min=lut_bounds[0][0], max=lut_bounds[0][1])
        log10_Te_clamped = torch.clamp(log10_Te, min=lut_bounds[1][0], max=lut_bounds[1][1])
        log10_r_clamped = torch.clamp(log10_r, min=lut_bounds[2][0], max=lut_bounds[2][1])
        
        # Normalize to [0, 1] for grid_sample
        # Assuming lut_bounds are the min/max of the log values.
        def normalize_to_01(val, vmin, vmax):
            return (val - vmin) / (vmax - vmin)
            
        coords_ne = normalize_to_01(log10_ne_clamped, lut_bounds[0][0], lut_bounds[0][1])
        coords_Te = normalize_to_01(log10_Te_clamped, lut_bounds[1][0], lut_bounds[1][1])
        coords_r = normalize_to_01(log10_r_clamped, lut_bounds[2][0], lut_bounds[2][1])
        
        # grid_sample expects input of shape (N, C, H, W) or (N, C, D, H, W) for 3D.
        # And grid of shape (N, ..., 3) for 3D.
        # Here we have 1D samples. We need to reshape for grid_sample.
        # Let's assume we process in batches or reshape.
        # For simplicity, assume we can call a helper that handles the tensor reshape.
        
        # Create grid coordinates for grid_sample
        # grid_sample expects coordinates in [-1, 1].
        coords_grid = torch.stack([coords_ne, coords_Te, coords_r], dim=-1) # (num_samples, 3)
        coords_grid = 2.0 * coords_grid - 1.0 # Map [0,1] to [-1,1]
        
        # Reshape for grid_sample: (1, 1, num_samples, 1, 1) for 3D?
        # Actually, grid_sample for 3D takes input (N, C, D, H, W) and grid (N, ..., 3).
        # We have 1D data. We can treat it as (1, 1, num_samples, 1, 1).
        # But the LUT is 3D.
        # Let's assume a helper function `query_lut` handles the tensor operations.
        
        emissivity_raw = query_lut_3d(lut_tensor, coords_grid) # Shape: (num_samples,)
        
        # 8. Unit Conversion
        # Convert from erg/s/cm^3/sr to photons/s/cm^2/arcsec^2
        # This involves physical constants and angular conversion.
        # Source says: "convert the raw LUT emissivity from erg to photon counts, and from steradian^-1 to arcsec^-2"
        # Let epsilon_c be the converted value.
        epsilon_c = convert_emissivity_units(emissivity_raw)
        
        # 9. Integration
        # Midpoint Riemann sum
        # I = sum(epsilon_c * delta_l)
        # delta_l is in cm.
        intensity = torch.sum(epsilon_c * delta_l_cm)
        
        intensities[i] = intensity
        
    return intensities

def generate_perspective_rays(camera_params):
    """Generate ray origins and directions for a perspective camera [G]."""
    # Placeholder implementation for ray generation based on camera_params
    # Returns ray_origins (N, 3) and ray_directions (N, 3)
    pass
```

## Subtopics

### Emissivity Look-Up Table (LUT) Construction & Interpolation
A precomputed 3D table mapping $\log_{10} n_e$, $\log_{10} T_e$, and $\log_{10} r$ to scalar Stokes-I emissivity, used to convert neural field predictions into radiative values during rendering [S1, unofficial]. It is constructed from PSI cube statistics and atomic-response ranges, storing values in log-space [S1, unofficial]. The grid resolution is $128 \times 64 \times 32$ along axes $(\log_{10} n_e, \log_{10} T_e, \log_{10} r)$ [S1, unofficial]. The radial LUT bounds are approximately $1.001$ to $10 R_\odot$ [S1, unofficial]. Queries outside the tabulated domain are clamped to the nearest LUT boundary before interpolation, with a backstop that explicitly returns the nearest boundary value rather than zeros or linear extrapolation [S1, unofficial]. It is implemented in PyTorch as a tensor-compatible structure using `torch.nn.functional.grid_sample` for trilinear interpolation [S1, unofficial]. This LUT is used both for synthetic observation generation (interpolating GT density/temperature on a spherical grid with periodic longitude wrapping) and for CoroNeRF rendering [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of pixels and the number of samples per ray. The number of samples per ray depends on the ray's intersection length with the AABB [S1, unofficial].
*   **Memory Complexity**: Dominated by the neural network weights and the LUT tensor ($128 \times 64 \times 32$ floats) [S1, unofficial].
*   **Measured Performance**: At production step size ($\Delta s = 1/256$), renders differ from a 4x-finer reference ($\Delta s = 1/1024$) by 7.5% in relative L2 error and $1.7 \times 10^{-3}$ in asinh image metric [S1, unofficial]. Convergence is roughly first-to-second order [S1, unofficial].
*   **Strengths**: Efficiently converts neural field predictions to physical emissivity; handles complex atomic physics via precomputed LUT; masks solar disk effectively [S1, unofficial].
*   **Limitations**: Requires precomputation of LUT; discretization-dependent field recovery; assumes scalar Stokes-I emissivity, neglecting magnetic-field geometry terms [S1, unofficial].
*   **When to use**: Synthetic dataset generation for solar corona; rendering neural plasma fields for visualization [S1, unofficial].
*   **When not to use**: When high-fidelity magnetic field effects are required; when real-time rendering is critical and LUT precomputation is prohibitive [S1, unofficial].

## Background
Originated in the context of neural tomography for solar corona reconstruction, specifically in the work "Image Fidelity is Not Field Fidelity: Joint Thermodynamic Reconstruction and Error Localization in Neural Tomography" [S1, unofficial]. The method uses oracle PSI density and temperature fields for synthetic dataset generation, which are identical to the ray sampling and integral procedures but replace the neural plasma field with ground truth fields [S1, unofficial]. The oracle fields live on a spherical coordinate system, requiring transformation from AABB to spherical coordinates before sampling using spherical trilinear field interpolators [S1, unofficial].

## Sources
[S1] Image Fidelity is Not Field Fidelity: Joint Thermodynamic Reconstruction and Error Localization in Neural Tomography, https://doi.org/10.48550/arxiv.2609.28868, fetched 2026-10-26 (unofficial)
