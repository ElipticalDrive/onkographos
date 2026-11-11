# Volume Rendering based CGH Calculation

Also called: Direct volume rendering method for holograms, Approximate Volume Rendering Based CGH Calculation, Approximate Volume Rendering Based CGH Calculation with Elemental Holograms, Approximate volume rendering based CGH calculation method with elemental holograms

## Overview
Volume Rendering based CGH Calculation is a method for generating Computer-Generated Holograms (CGHs) for volumetric data by treating voxels as transparent objects with diffuse reflection properties [S3, unofficial]. It solves the limitation of Maximum Intensity Projection (MIP) methods, which only display objects with maximum voxel values, by enabling the generation of clear holograms for transparent and multi-layer objects [S3, unofficial]. Unlike standard volume rendering which produces 2D images, this method calculates the complex wavefield (amplitude and phase) for each voxel to record 3D wave information on the hologram plane [S3, unofficial]. It is distinguished from sibling methods by its explicit simulation of light propagation from a source to the voxel, diffuse reflection at the voxel, and propagation from the voxel to the viewpoint, often utilizing an elemental hologram approximation to reduce computational cost [S3, unofficial].

## How it works
**Inputs:**
*   Volume data: A 3D grid of voxels, each with a scalar value $I_x$.
*   Color Map and Alpha Map: Lookup tables defining the mapping from voxel value $I_x$ to RGB color components and opacity $\alpha$.
*   Scene parameters: Light source position, viewpoint position, wavelength $\lambda$, and hologram resolution.
*   (Optional for Approximate Method): Number of elemental holograms $N_e$.

**Outputs:**
*   Complex amplitude distribution $\mu(\xi, \eta)$ on the hologram plane.

**Data Structures:**
*   **Voxel Grid:** 3D array storing voxel values $I_x$.
*   **Attribute Maps:** 3D arrays storing derived $R, G, B, \alpha$ for each voxel.
*   **Hologram Plane:** 2D array of complex numbers representing the superposition of wavefields.

**Step 1: Initialization of Color and Opacity**
For each voxel $x$ with value $I_x$, calculate its color components ($R, G, B$) and opacity $\alpha$ using linear interpolation on the provided maps [S3, unofficial]. Let $C$ represent any of $R, G, B, \alpha$. Let $C_2, C_3$ be the known map values at endpoints $I_2, I_3$. The calculated component $C_x$ is:

$$ C_x = C_3 - \frac{(C_3 - C_2)(I_3 - I_x)}{I_3 - I_2} $$

where $I_x$ is the value of the current voxel [S3, unofficial].

**Step 2: Rendering Each Voxel (Diffuse Reflection Model)**
Each voxel is treated as a transparent object. For a single viewpoint, the process for voxel $eli$ involves three parts [S3, unofficial]:

1.  **Light Propagation from Source to Voxel:**
    Calculate the incident light intensity $I_{di}$ reaching voxel $eli$ after attenuation by preceding voxels along the path from the light source [S3, unofficial]:
    $$ I_{di} = I_{d0} \prod_{j=0}^{k} (1 - \alpha_j) $$
    where $I_{d0}$ is the intensity of the light source, $\alpha_j$ is the opacity value of voxel $j$ along the path, and the product is taken over all voxels $j$ between the source and voxel $eli$ [S3, unofficial].

2.  **Calculation of Reflected Light:**
    Calculate the intensity of the reflected light $I_{ri}$ using the Lambertian diffuse reflection model [S3, unofficial]:
    $$ I_{ri} = k_{di} I_{di} \max(0, \mathbf{L} \cdot \mathbf{N}) $$
    where $k_{di}$ is the ratio of Lambert light, $\mathbf{L}$ is the unit vector pointing to the light source, and $\mathbf{N}$ is the normal unit vector derived from the gradient of the $\alpha$ map in the $x, y, z$ directions [S3, unofficial]. Note that $I_{di}$ and $I_{ri}$ are independent of the viewpoint direction [S3, unofficial].

3.  **Light Propagation from Voxel to Viewpoint:**
    Calculate the final reflected light intensity $I_{rLi}$ arriving at the viewpoint, accounting for attenuation by subsequent voxels [S3, unofficial]:
    $$ I_{rLi} = I_{ri} \prod_{j=i+1}^{m} (1 - \alpha_j) $$
    where the product is taken over all voxels $j$ between voxel $eli$ and the viewpoint [S3, unofficial]. This step must be recalculated for each viewpoint [S3, unofficial].

**Step 3: CGH Calculation (Point Light Source Method)**
Treat each voxel as a point light source. The complex amplitude distribution $\mu(\xi, \eta)$ contributed by voxel $i$ to the hologram plane is [S3, unofficial]:

$$ \mu_i(\xi, \eta) = \frac{\sqrt{I_{rLi}}}{r_i} \exp\left(-j(k r_i + \phi_i)\right) $$

where $r_i$ is the distance from voxel $i$ to the hologram pixel, $k = 2\pi/\lambda$ is the wavenumber, and $\phi_i$ is the initial phase of the source [S3, unofficial]. The total hologram is the superposition of all voxel contributions:

$$ U_h(x,y) = \sum_{i=1}^{M} \mu_i(x,y) $$

**Approximate Method with Elemental Holograms:**
To reduce computation, divide the hologram into $N_e$ elemental holograms [S3, unofficial].
1.  Use the center pixel of each elemental hologram to represent the entire elemental hologram for the calculation of final light propagation [S3, unofficial].
2.  Calculate the final light propagation $I_{rLi}$ only once per elemental hologram $k$ [S3, unofficial].
3.  For each pixel in elemental hologram $k$, the amplitude of point light source $i$ is assumed constant, denoted as $I_{ek\_rLi}$ [S3, unofficial].
4.  The complex amplitude for elemental hologram $k$ is:
    $$ \mu_i(\xi, \eta) = \frac{\sqrt{I_{ek\_rLi}}}{r_i} \exp\left(-j(k r_i + \phi_i)\right) $$
    where $I_{ek\_rLi}$ is the amplitude of point light source $i$ for elemental hologram $k$ [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_rendering_cgh(volume_data, color_map, alpha_map, light_source_pos, viewpoint_pos, wavelength, hologram_res):
    """
    Standard Volume Rendering based CGH Calculation.
    """
    M = volume_data.num_voxels
    N = hologram_res.width * hologram_res.height
    k = 2 * pi / wavelength
    
    # Step 1: Initialize attributes
    voxel_attrs = []
    for voxel in volume_data:
        Ix = voxel.value
        # Linear interpolation for R, G, B, Alpha
        R = interpolate(Ix, color_map['R'])
        G = interpolate(Ix, color_map['G'])
        B = interpolate(Ix, color_map['B'])
        Alpha = interpolate(Ix, alpha_map['Alpha'])
        voxel_attrs.append({'R': R, 'G': G, 'B': B, 'Alpha': Alpha, 'pos': voxel.pos})
        
    # Step 2 & 3: Calculate wavefield for each voxel and viewpoint
    # Note: For single viewpoint, we sum contributions. 
    # For multi-viewpoint, this loop runs per viewpoint.
    hologram_plane = zeros(hologram_res, dtype=complex)
    
    for i, attr in enumerate(voxel_attrs):
        pos_i = attr['pos']
        alpha_i = attr['Alpha']
        
        # Part 1: Incident light from source
        # Calculate attenuation from source to voxel i
        path_source_to_i = get_voxels_between(light_source_pos, pos_i)
        attenuation_source = 1.0
        for j in path_source_to_i:
            if j != i: # Exclude self if necessary, or include based on specific model definition
                 attenuation_source *= (1 - voxel_attrs[j]['Alpha'])
        I_di = I_d0 * attenuation_source
        
        # Part 2: Reflected light (Lambertian)
        # Calculate normal N from alpha gradient (simplified here)
        N = compute_normal_gradient(alpha_i, pos_i)
        L = normalize(light_source_pos - pos_i)
        I_ri = k_di * I_di * max(0, dot(L, N))
        
        # Part 3: Propagation to viewpoint
        # Calculate attenuation from voxel i to viewpoint
        path_i_to_view = get_voxels_between(pos_i, viewpoint_pos)
        attenuation_view = 1.0
        for j in path_i_to_view:
            if j != i:
                 attenuation_view *= (1 - voxel_attrs[j]['Alpha'])
        I_rLi = I_ri * attenuation_view
        
        # Calculate complex amplitude contribution to hologram
        for pixel in hologram_plane:
            r_i = distance(pos_i, pixel.pos)
            phase = -(k * r_i) # Assuming phi_i = 0 for simplicity or derived from source
            amplitude = sqrt(I_rLi) / r_i
            hologram_plane[pixel] += amplitude * exp(1j * phase)
            
    return hologram_plane

def approximate_volume_rendering_cgh(volume_data, color_map, alpha_map, light_source_pos, viewpoint_pos, wavelength, hologram_res, N_e):
    """
    Approximate Volume Rendering based CGH Calculation with Elemental Holograms.
    """
    M = volume_data.num_voxels
    N = hologram_res.width * hologram_res.height
    k = 2 * pi / wavelength
    
    # Step 1: Initialize attributes (Same as standard)
    voxel_attrs = []
    for voxel in volume_data:
        Ix = voxel.value
        R = interpolate(Ix, color_map['R'])
        G = interpolate(Ix, color_map['G'])
        B = interpolate(Ix, color_map['B'])
        Alpha = interpolate(Ix, alpha_map['Alpha'])
        voxel_attrs.append({'R': R, 'G': G, 'B': B, 'Alpha': Alpha, 'pos': voxel.pos})
        
    # Divide hologram into N_e elemental holograms
    elemental_holograms = divide_hologram(hologram_res, N_e)
    
    # Pre-calculate incident and reflected light for each voxel (independent of viewpoint/elemental split)
    voxel_reflected_intensity = []
    for i, attr in enumerate(voxel_attrs):
        pos_i = attr['pos']
        alpha_i = attr['Alpha']
        
        # Part 1: Incident light
        path_source_to_i = get_voxels_between(light_source_pos, pos_i)
        attenuation_source = 1.0
        for j in path_source_to_i:
            if j != i:
                 attenuation_source *= (1 - voxel_attrs[j]['Alpha'])
        I_di = I_d0 * attenuation_source
        
        # Part 2: Reflected light
        N = compute_normal_gradient(alpha_i, pos_i)
        L = normalize(light_source_pos - pos_i)
        I_ri = k_di * I_di * max(0, dot(L, N))
        
        voxel_reflected_intensity.append(I_ri)
        
    # Step 3: CGH Calculation with Elemental Holograms
    hologram_plane = zeros(hologram_res, dtype=complex)
    
    for k_elem, elem_holo in enumerate(elemental_holograms):
        # Use center pixel of elemental hologram to represent the entire elemental hologram
        center_pixel = get_center_pixel(elem_holo)
        
        for i, attr in enumerate(voxel_attrs):
            pos_i = attr['pos']
            I_ri = voxel_reflected_intensity[i]
            
            # Part 3: Propagation to viewpoint (calculated once per elemental hologram)
            path_i_to_view = get_voxels_between(pos_i, viewpoint_pos)
            attenuation_view = 1.0
            for j in path_i_to_view:
                if j != i:
                     attenuation_view *= (1 - voxel_attrs[j]['Alpha'])
            
            # I_ek_rLi is the amplitude for elemental hologram k
            I_ek_rLi = I_ri * attenuation_view
            
            # Apply to all pixels in this elemental hologram
            for pixel in elem_holo:
                r_i = distance(pos_i, pixel.pos)
                phase = -(k * r_i)
                amplitude = sqrt(I_ek_rLi) / r_i
                hologram_plane[pixel] += amplitude * exp(1j * phase)
                
    return hologram_plane
```

**Parameters:**
*   $N_e$: Number of elemental holograms. Smaller $N_e$ relative to $N$ significantly reduces calculation time [S3, unofficial].
*   Iterations: 50 Fourier iterative calculations based on the Gerchberg-Saxton method may be used to optimize the CGH if phase-only SLMs are targeted [S2].
*   Maximum spatial frequency: 4 lp/mm (used in demonstrations) [S2].
*   Focal point separation: 284 µm from the optical axis (used in demonstrations) [S2].

**Edge Cases and Pitfalls:**
*   **Computational Cost:** The calculation amount is very large, making the method computationally expensive compared to polygon-based or MIP-based methods [S3, unofficial]. Complexity of incident light is $O(M^2)$, reflected light $O(M)$, final light propagation $O(M^2 \cdot N_e)$, and total CGH calculation $O(M^2 \cdot N)$ [S3, unofficial].
*   **Resolution Limits:** Output devices may not transfer the pattern to film at the required resolution, necessitating multiplex holograms or elemental hologram approximations [S1, unofficial].
*   **Lighting Conditions:** Holograms may require special lighting conditions, specifically a point monochrome light source at a specific angle, for reconstruction [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The standard method has a computational complexity of $O(M^2 \cdot N)$ for CGH calculation, where $M$ is the number of voxels and $N$ is the number of hologram pixels [S3, unofficial]. The approximate method with elemental holograms reduces the final light propagation complexity to $O(M^2 \cdot N_e)$, where $N_e$ is the number of elemental holograms [S3, unofficial].
*   **Measured Performance:** In experiments, the proposed method took 24.3 hours for rendering and 3.8 minutes for CGH calculation, totaling 24.3 hours. The approximate method with elemental holograms reduced this to 3.6 hours total (3.6 hours rendering, 3.7 minutes CGH calculation) [S3, unofficial]. This is 5 to 6 times faster than the standard proposed method [S3, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Generates clear holograms for transparent and multi-layer objects, which MIP-based methods cannot do well [S3, unofficial]. Provides better detail compared to polygon-based and MIP-based methods [S3, unofficial].
*   **Limitations:** Extremely computationally expensive compared to other methods [S3, unofficial]. Requires special lighting conditions for reconstruction [S1, unofficial].
*   **When to use:** When high-fidelity rendering of transparent and multi-layer volumetric data is required, and computational resources allow for the approximate method.
*   **When not to use:** When real-time performance is critical and lower fidelity (MIP or polygon-based) is acceptable, or when output devices cannot handle the required resolution.

## Background
Holography is a form of lensless photography based on the recording of amplitude and phase of light illuminating an object [S1, unofficial]. Computed holograms are created by calculating the interference pattern and recording it on film [S1, unofficial]. However, fully computed holograms are not yet practical due to recording difficulties and expense [S1, unofficial]. Multiplex holograms were developed to overcome resolution limits by rendering a series of perspective computer-generated images and storing them on the same hologram using multiple exposures [S1, unofficial]. Benton's Alcove holograms use approximately 1000 images [S1, unofficial]. The volume rendering based CGH calculation method was proposed to generate holograms for volume data by applying direct volume rendering concepts, specifically treating voxels as transparent objects with diffuse reflection [S3, unofficial]. The approximate method with elemental holograms was introduced to address the high computational cost, building on the elemental hologram concept proposed by Dr. Ichikawa [S3, unofficial].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] Colour volumetric display based on holographic-laser-excited graphics using drawing space separation, https://www.youtube.com/watch?v=W-CbMq23NKk, fetched 2026-10-28
[S3] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
