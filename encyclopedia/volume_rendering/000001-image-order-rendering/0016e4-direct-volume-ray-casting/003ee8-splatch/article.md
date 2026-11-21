# Splatch

Also called: Splotch

## Overview
Splatch is a lightweight, CPU-based ray-tracing software tool designed for the fast visualization of point-like data from cosmological simulations, such as Smoothed Particle Hydrodynamics (SPH) outputs [S1]. It solves the problem of rendering large-scale particle datasets into realistic 3D images by optimizing ray-tracing calculations through depth-based particle ordering and analytical integration of emission and absorption along rays [S1]. Unlike general-purpose visualization packages like TIPSY or VisIVO, which are not primarily designed for ray-tracing, Splatch specifically targets the generation of ray-traced-like images that provide more realistic impressions of 3D structures [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ particles, each defined by position $\mathbf{p}_i$, peak density $\rho_{0,i}$, spread parameter $\sigma_i$, and emission/absorption coefficients $\mathbf{E}_i, \mathbf{A}_i$ (vectors of size 3 for RGB) [S1]. Camera parameters: position $\mathbf{c}$, look-at point $\mathbf{l}$, sky vector $\mathbf{s}$, and field-of-view angle $\phi$ [S1].
*   **Output:** A 2D image (frame buffer) stored in a standard format (e.g., TGA, JPEG) [S1].

**Data Structures:**
*   **Particle Data:** The algorithm requires 30 bytes per particle [S1]. The exact memory layout is not specified in the sources [S1].
*   **Frame Buffer:** An array storing an RGB color triplet for every pixel of the resulting image [S1]. Initialized to black (0, 0, 0) [S1].

**Algorithm Steps:**

1.  **Coordinate Transformation:**
    Define the viewing direction $\mathbf{v} = \mathbf{l} - \mathbf{c}$ [S1]. Determine a linear transformation matrix $M$ that maps the camera position $\mathbf{c}$ to the origin and aligns $\mathbf{v}$ with the positive z-axis [S1]. Apply $M$ to all particle positions $\mathbf{p}_i$ to obtain transformed positions $\mathbf{p}'_i$ [S1]. This simplifies subsequent operations by aligning the ray-tracing direction with the z-axis [S1].

2.  **Sorting:**
    Sort the particles in descending order of their z-coordinate in the transformed space ($z'_i$) [S1]. This ensures that particles closer to the camera (higher z) are processed first, allowing for correct front-to-back compositing [S1].

3.  **Projection Setup:**
    Apply a perspective projection using the user-defined field-of-view angle $\phi$ [S1]. This defines the mapping from 3D world coordinates to 2D screen coordinates for each pixel [S1].

4.  **Initialization:**
    Allocate the frame buffer for the desired image resolution [S1]. Initialize all pixel values to black (0, 0, 0) [S1].

5.  **Rendering Loop:**
    For each particle $i$ in the sorted list:
    a.  Determine the bounding box of the particle's influence on the image plane. The particle's density contribution is modeled as a Gaussian distribution:
        $$ \rho_i(\mathbf{r}) = \rho_{0,i} \exp(-r^2/\sigma_i^2) $$
        where $r$ is the distance from the particle center [S1].
    b.  The distribution is set to zero at a distance $f \cdot \sigma_i$, where $f$ is a compact support factor chosen such that $f \cdot \sigma_i \approx h$ (the SPH smoothing length) [S1]. Rays passing further than this distance are unaffected [S1].
    c.  For each pixel $(x,y)$ within the particle's footprint:
        i.  Calculate the minimum distance $d_0$ between the ray passing through $(x,y)$ and the particle center in the transformed space [S1].
        ii. Calculate the integral of the density along the ray:
            $$ \int_{-\infty}^{\infty} \rho_i(x) dx = \rho_{0,i} \sigma_i \exp(-d_0^2/\sigma_i^2) \sqrt{\pi} $$
            [S1].
        iii. Update the pixel's intensity $\mathbf{I}$ using the radiative transfer equation solution:
            $$ \mathbf{I}_{after} = (\mathbf{I}_{before} - \mathbf{E}_i/\mathbf{A}_i) \exp(-\mathbf{A}_i \int_{-\infty}^{\infty} \rho_i(x) dx) + \mathbf{E}_i/\mathbf{A}_i $$
            where $\mathbf{E}_i$ and $\mathbf{A}_i$ are the emission and absorption coefficient vectors for the particle [S1]. The division and multiplication are component-wise for RGB [S1].
        iv. Store the updated $\mathbf{I}_{after}$ in the frame buffer [S1].

6.  **Output:**
    Write the final frame buffer to disk in a standard format (e.g., TGA or JPEG) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def splatch_render(particles, camera_pos, look_at, sky_vec, fov_deg, image_width, image_height):
    # Step 1: Coordinate Transformation
    v = np.array(look_at) - np.array(camera_pos)
    v_norm = v / np.linalg.norm(v)
    
    # Construct transformation matrix M: maps camera_pos to origin, aligns v with +z
    z_axis = v_norm
    x_axis = np.cross(sky_vec, z_axis)
    x_axis = x_axis / np.linalg.norm(x_axis)
    y_axis = np.cross(z_axis, x_axis)
    y_axis = y_axis / np.linalg.norm(y_axis)
    
    R = np.array([x_axis, y_axis, z_axis])
    T = np.eye(4)
    T[:3, 3] = -np.array(camera_pos)
    R_4x4 = np.eye(4)
    R_4x4[:3, :3] = R
    M = R_4x4 @ T
    
    transformed_particles = []
    for p in particles:
        pos = np.array(p['pos'])
        pos_hom = np.append(pos, 1.0)
        pos_new = M @ pos_hom
        transformed_particles.append({
            'pos': pos_new[:3],
            'rho0': p['rho0'],
            'sigma': p['sigma'],
            'E': np.array(p['E']),
            'A': np.array(p['A'])
        })
        
    # Step 2: Sorting
    transformed_particles.sort(key=lambda p: p['pos'][2], reverse=True)
    
    # Step 3: Projection Setup
    fov_rad = math.radians(fov_deg)
    tan_fov_half = math.tan(fov_rad / 2.0)
    
    # Step 4: Initialization
    frame_buffer = np.zeros((image_height, image_width, 3))
    
    # Step 5: Rendering Loop
    for p in transformed_particles:
        pos_p = np.array(p['pos'])
        rho0 = p['rho0']
        sigma = p['sigma']
        E = p['E']
        A = p['A']
        
        # Guard against division by zero if A is zero [G]
        A_safe = np.where(A == 0, 1.0, A)
        E_safe = np.where(A == 0, 0.0, E)
        ratio = E_safe / A_safe
        
        for y in range(image_height):
            for x in range(image_width):
                # Calculate ray direction in transformed space
                nx = (x - image_width / 2.0) / (image_width / 2.0)
                ny = (y - image_height / 2.0) / (image_height / 2.0)
                rx = nx * tan_fov_half
                ry = ny * tan_fov_half
                rz = 1.0
                r = np.array([rx, ry, rz])
                r_unit = r / np.linalg.norm(r)
                
                # d0 is distance from ray to particle center
                cross_prod = np.cross(pos_p, r_unit)
                d0 = np.linalg.norm(cross_prod)
                
                # Compact support culling: skip if d0 is too large
                # f is typically ~3 for SPH kernels, but exact value not given [G]
                if d0 > sigma * 3.0:
                    continue
                    
                # Calculate integral of density along ray
                integral_val = rho0 * sigma * math.exp(-(d0**2) / (sigma**2)) * math.sqrt(math.pi)
                
                # Update intensity using radiative transfer solution
                exponent = -A_safe * integral_val
                frame_buffer[y, x] = (frame_buffer[y, x] - ratio) * np.exp(exponent) + ratio
        
    return frame_buffer
```

**Parameters:**
*   **Compact support factor $f$:** Chosen such that $f \cdot \sigma_p \approx h$ (SPH smoothing length) [S1].
*   **Emission/Absorption ratio:** Recommended to set $E_p = A_p$ for visually appealing images [S1].
*   **Field-of-view angle $\phi$:** Typically set to 30 degrees [S1].

**Edge Cases and Pitfalls:**
*   **Particle Overlap:** The algorithm assumes particles do not overlap for the simple intensity update formula, but in reality, they do. The source notes that since relative intensity changes due to a single particle are small, the approximation is acceptable [S1].
*   **Precision:** The exponential decay in the Gaussian distribution can lead to underflow for large $d_0$. The compact support factor $f$ helps mitigate this by truncating the distribution [S1].
*   **Division by Zero:** If absorption coefficient $A_p$ is zero, the term $E_p/A_p$ is undefined. The sources do not specify handling for this case, so a guard should clamp or skip zero absorption [G].

## Operating characteristics
*   **Time Complexity:** CPU time $T_{CPU}$ is proportional to the number of pixels $n_{pixel}$ ($T_{CPU} \propto n_{pixel}$) and inversely proportional to the square root of the field-of-view angle $\phi$ ($T_{CPU} \propto \phi^{-0.5}$) [S1].
*   **Memory Complexity:** The algorithm requires 30 bytes per particle [S1]. This allows rendering more than 70 million particles on a standard PC with 2 GByte of main memory [S1].
*   **Performance:** On an AMD Opteron 850 CPU (2400 MHz), rendering a frame with 1400x1050 resolution from a simulation with ~11 million gas and ~5 million star particles took ~67 seconds [S1]. Rendering a zoomed frame with 800x800 resolution from a simulation with ~5 million gas particles took ~72 seconds on the same system [S1].
*   **Parallelization:** Uses OpenMP to distribute image regions over several cores on shared memory platforms [S1].
*   **Strengths:** Fast, lightweight, handles large particle sets (millions of particles), produces realistic 3D impressions via ray-tracing [S1].
*   **Limitations:** CPU-based, not GPU-accelerated (unlike many modern splatting methods), assumes small intensity changes per particle for compositing accuracy [S1].

## Background
Splatch (Splotch) was developed by K. Dolag, C. Gheller, and S. Imboden from the Max-Planck-Institut für Astrophysik and CINECA [S1]. It was published in 2008 in the New Journal of Physics [S1]. The tool was designed to address the limitations of existing visualization packages like TIPSY and VisIVO, which were not optimized for ray-tracing-like images of particle-based cosmological simulations [S1]. It is specifically tailored for SPH data, leveraging the Gaussian-like shape of the SPH kernel [S1].

## Sources
[S1] Splotch: visualizing cosmological simulations, https://doi.org/10.1088/1367-2630/10/12/125006, fetched 2026-10-28
