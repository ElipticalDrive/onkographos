# Indirect Illumination Modeling via Baking

Also called: Baking-based Occlusion for Indirect Lighting, Irradiance Volume Baking, SH Occlusion Caching

## Overview
This method models indirect illumination in 3D Gaussian Splatting (3DGS) by precomputing occlusion information and storing it in Spherical Harmonics (SH) coefficients within 3D volumes [S1]. It solves the difficulty of calculating occlusion in 3DGS, which uses differentiable forward mapping splatting rather than ray marching [S1]. It sets itself apart from sibling methods like the Light Cache Rendering Pipeline by explicitly baking geometric occlusion into SH coefficients to approximate indirect lighting, inspired by real-time rendering techniques such as Irradiance Volumes and Light Probes [S1].

## How it works
**Inputs:**
*   Optimized 3D Gaussians $\mathcal{G}$ (mean $\mu$, covariance $\Sigma$, opacity, color) [S1].
*   A 3D grid space where occlusion volumes are placed [S1].
*   A distance threshold $\tau$ for binary occlusion determination [S1].

**Outputs:**
*   Occlusion Volumes $\mathcal{V}_{occl}$, where each volume $v_i^{occl}$ contains SH coefficients $f_i^o$ representing the occlusion function $O(\theta, \phi)$ [S1].
*   (Optional) Illumination Volumes $\mathcal{V}_{illu}$ containing SH coefficients for environment illumination [S1].

**Data Structures:**
*   **Occlusion Volume $v_i^{occl}$**: A 3D grid cell.
    *   Field `coeffs`: Array of SH coefficients $f_i^o$ (dimension depends on SH order, typically $L=2$ or $3$ for real-time, though exact order is "Not given in the sources" beyond the general SH architecture mention).
    *   These coefficients represent the occlusion function $O(\theta, \phi)$ [S1].

**Steps:**
1.  **Placement**: Regularly place occlusion volumes $\mathcal{V}_{occl}$ in the 3D space surrounding the scene geometry [S1].
2.  **Depth Rendering**: For each volume $v_i^{occl}$, perform six forward mapping splatting passes to render a depth cubemap $\{\hat{D}_p^i\}_{p=1}^6$, where $p$ indexes the six faces of the cubemap [S1].
3.  **Binary Conversion**: Convert the depth cubemap $\{\hat{D}_p^i\}$ into a binary occlusion cubemap $\{\hat{O}_p^i\}$ using the distance threshold $\tau$ [S1].
    *   Let $d$ be the depth value at a pixel in the cubemap.
    *   $\hat{O}_p^i(x,y) = 1$ if $d < \tau$, else $0$ (or vice versa depending on definition; the source implies "occlusion" is cached, so likely 1 for occluded/hidden, but the formula $1-m$ suggests $m$ is occlusion factor. The source states "convert... into binary occlusion cubemap", implying a threshold test).
4.  **SH Convolution**: Numerically calculate the convolution of the binary occlusion cubemap with SH basis functions $Y_{lm}$ to obtain the SH coefficients $f_i^o$ for the volume [S1].
    *   The occlusion function $O(\omega)$ is approximated as:
        $$ O(\omega) \approx \sum_{l=0}^{L} \sum_{m=-l}^{l} f_i^o(l,m) Y_{lm}(\omega) $$
    *   The coefficients $f_i^o(l,m)$ are derived from the integral of the occlusion cubemap against the basis functions [S1].
5.  **Illumination Caching (Optional)**: Similarly, cache environment illumination into illumination volumes $\mathcal{V}_{illu}$ using SH coefficients derived from captured environment cubemaps $\{\hat{I}_p^i\}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bake_occlusion_volumes(gaussians, grid_spacing, threshold):
    """
    Precompute occlusion SH coefficients for a 3D grid of volumes.
    """
    occlusion_volumes = []
    
    # 1. Place volumes in 3D space
    # Assuming a regular grid covering the scene bounding box
    grid_points = generate_3d_grid(grid_spacing)
    
    for point in grid_points:
        v_occl = {}
        
        # 2. Render depth cubemap for this volume center
        # Perform 6 forward mapping splatting passes
        depth_cubemap = []
        for face_index in range(6):
            # Define camera orientation for the cubemap face
            cam_pos = point
            cam_forward = get_cubemap_face_direction(face_index)
            cam_up = get_up_vector(cam_forward)
            
            # Render depth using 3DGS forward mapping
            # This projects 3D Gaussians onto the 2D plane of the cubemap face
            depth_map = render_depth_gaussian_splatting(
                gaussians=gaussians, 
                camera_position=cam_pos, 
                camera_forward=cam_forward, 
                camera_up=cam_up
            )
            depth_cubemap.append(depth_map)
        
        # 3. Convert depth cubemap to binary occlusion cubemap
        occlusion_cubemap = []
        for depth_map in depth_cubemap:
            # Binary thresholding
            # 1 if occluded (depth < threshold), 0 otherwise
            # Note: Exact binary logic (1 for occluded vs 1 for visible) 
            # depends on how O(theta, phi) is defined in the final shading.
            # Source says "binary occlusion cubemap".
            binary_map = (depth_map < threshold).astype(float)
            occlusion_cubemap.append(binary_map)
        
        # 4. Calculate SH coefficients from occlusion cubemap
        # Convolve the cubemap with SH basis functions
        sh_coeffs = compute_sh_coefficients_from_cubemap(occlusion_cubemap)
        
        v_occl['coeffs'] = sh_coeffs
        occlusion_volumes.append(v_occl)
        
    return occlusion_volumes

def compute_sh_coefficients_from_cubemap(cubemap_faces):
    """
    Compute SH coefficients by integrating the cubemap values against SH basis functions.
    """
    # Initialize coefficients array for desired SH order L
    # Number of coefficients = (L+1)^2
    num_coeffs = (3 + 1)**2 # Assuming L=2 or 3, exact L not specified
    coeffs = np.zeros(num_coeffs)
    
    # Integrate over the sphere using the 6 cubemap faces
    for face in cubemap_faces:
        # Iterate over pixels in the face
        for y in range(face.height):
            for x in range(face.width):
                # Get direction vector for this pixel
                omega = get_direction_from_cubemap_pixel(face, x, y)
                
                # Get value from occlusion cubemap
                val = face[y, x]
                
                # Add to SH coefficients
                # f_lm = integral( f(omega) * Y_lm(omega) domega )
                # Discrete approximation
                for l in range(3): # Assuming L=2 or 3
                    for m in range(-l, l+1):
                        idx = get_sh_index(l, m)
                        y_lm = spherical_harmonic(l, m, omega)
                        coeffs[idx] += val * y_lm
                        
    # Normalize by solid angle of each pixel (approximate)
    # Not given in sources, but required for correct SH projection [G]
    solid_angle = 4 * np.pi / (6 * face.width * face.height)
    coeffs *= solid_angle
    
    return coeffs
```

**Parameters:**
*   **Distance Threshold $\tau$**: Manually set threshold to determine occlusion from depth [S1]. Value: "Not given in the sources."
*   **Grid Spacing**: Determines the density of occlusion volumes [S1]. Value: "Not given in the sources."

**Edge Cases and Pitfalls:**
*   **Precision**: The conversion from depth to binary occlusion is sensitive to the threshold $\tau$ [S1].
*   **Boundaries**: The method relies on forward mapping splatting, which may have artifacts at volume boundaries if not handled carefully [S1].
*   **Degenerate Input**: If the scene geometry is sparse, the depth cubemap may have large gaps, leading to inaccurate occlusion estimates [S1].

## Subtopics

### Light Cache Rendering Pipeline
A sibling method that uses a light cache rendering pipeline. It differs by focusing on the rendering pipeline structure rather than explicit SH baking of occlusion. Choose when a full pipeline integration is preferred over explicit volume caching. [Light Cache Rendering Pipeline](9159-light-cache-rendering-pipeline.md)

## Operating characteristics
*   **Time Complexity**: Precomputation involves $N_v \times 6$ rendering passes, where $N_v$ is the number of occlusion volumes. Each pass is $O(N_g)$ where $N_g$ is the number of Gaussians. SH convolution is $O(N_p)$ where $N_p$ is the number of pixels in the cubemap.
*   **Memory Complexity**: $O(N_v \times C)$, where $C$ is the number of SH coefficients per volume.
*   **Measured Performance**: "Not given in the sources."
*   **Strengths**: Efficient online calculation of indirect illumination by leveraging precomputed SH coefficients [S1].
*   **Limitations**: Requires precomputation step; accuracy depends on grid density and threshold [S1].
*   **When to Use**: When indirect illumination is needed in 3DGS and real-time performance is critical, and precomputation is acceptable [S1].
*   **When Not to Use**: For dynamic scenes where geometry changes frequently, as precomputation would need to be repeated [S1].

## Background
The method is inspired by the "Indirect Lighting Cache" used in real-time rendering [S1]. It draws inspiration from precomputation techniques in the video game industry, such as Irradiance Volume in Blender, Lightmass Volume in Unreal, and Light Probes in Unity [S1]. It was introduced in the GS-IR (3D Gaussian Splatting for Inverse Rendering) framework [S1].

## Sources
[S1] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
