# Halftone Shading

Also called: Disk-based halftone

## Overview
Halftone shading is a non-photorealistic illumination technique that represents gradual lighting variations through shading patterns of varying sizes, specifically disks whose radii are inversely proportional to illumination intensity [S1, unofficial]. It distinguishes itself from standard photorealistic shading by replacing continuous tone gradients with discrete, scalable geometric primitives, creating an illustrative aesthetic suitable for anatomical or technical visualization. It builds on [Lit Sphere Shading](785-lit-sphere-shading.md) by providing a specific pattern-based shading option within that framework.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D model (point cloud or polygonal mesh), light position $LP$, surface normals $n_i$, and view direction.
*   **Output:** A rendered image containing scaled disks representing the shading intensity.

**Data Structures:**
*   **Lighting Model:** Phong illumination model is used to compute light intensity $L_i$ at a point $x_i$ with normal $n_i$ [S1, unofficial].
    *   Formula: $L_i = k_a + k_d \max(0, n_i \cdot (LP - x_i)) + k_s \max(0, n_i \cdot v)^{shininess}$ [S1, unofficial].
    *   Note: The exact coefficients ($k_a, k_d, k_s$) and shininess exponent are not specified in the source, but $L_i$ is defined as the result of this standard Phong calculation [S1, unofficial].
*   **Scaling Factor:** A scalar value $S_{shd}$ determines the size of the disk.

**Parameters:**
*   $S_{all}$: Global normalization factor for the scaling factor [S1, unofficial].
*   $I_{shd}$: Parameter controlling the overall intensity (darkness) of shading [S1, unofficial].
*   $A_{shd}$: Parameter controlling the stretching of the shading [S1, unofficial].
*   $n$: Parameter controlling the transition within the shade [S1, unofficial].

**Steps:**
1.  **Compute Illumination:** For each point or triangle, calculate the illumination intensity $L_i$ using the Phong model [S1, unofficial].
2.  **Calculate Scaling Factor:** Compute the scaling factor $S_{shd}$ using the following formula [S1, unofficial]:
    $$S_{shd} = S_{all} \cdot I_{shd} \cdot A_{shd} \cdot L_i^{-n}$$
    *   Note: The source presents the formula as $S_{shd} = S_{all} \cdot I_{shd} \cdot A_{shd} \cdot L_i^{-n}$ [S1, unofficial]. The order of multiplication for $I_{shd}$ and $A_{shd}$ is commutative.
3.  **Render Disks:**
    *   **For Point Models:** Scale each shading disk by $S_{shd}$ [S1, unofficial].
    *   **For Polygonal Models:** Render scaled shading disks at the midpoints of triangles, oriented orthogonal to the viewing direction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def halftone_shading(points, normals, light_position, view_direction, params):
    """
    points: List of 3D coordinates (x, y, z)
    normals: List of 3D unit vectors (nx, ny, nz)
    light_position: 3D coordinate (lx, ly, lz)
    view_direction: 3D unit vector (vx, vy, vz)
    params: Dictionary containing S_all, I_shd, A_shd, n, k_a, k_d, k_s, shininess
    """
    S_all = params['S_all']
    I_shd = params['I_shd']
    A_shd = params['A_shd']
    n = params['n']
    k_a = params['k_a']
    k_d = params['k_d']
    k_s = params['k_s']
    shininess = params['shininess']
    
    rendered_disks = []
    
    for i in range(len(points)):
        x_i = points[i]
        n_i = normals[i]
        
        # Compute Phong illumination
        # Vector from point to light
        light_vec = (light_position[0] - x_i[0], 
                     light_position[1] - x_i[1], 
                     light_position[2] - x_i[2])
        
        # Diffuse term
        dot_nd_l = max(0, n_i[0]*light_vec[0] + n_i[1]*light_vec[1] + n_i[2]*light_vec[2])
        diffuse = k_d * dot_nd_l
        
        # Specular term
        # Reflection vector r = 2 * (n . l) * n - l
        # However, standard Phong uses view vector v. 
        # Assuming v is fixed view direction for simplicity as per source context
        # Specular: k_s * (n . v)^shininess
        dot_nv = max(0, n_i[0]*view_direction[0] + n_i[1]*view_direction[1] + n_i[2]*view_direction[2])
        specular = k_s * (dot_nv ** shininess)
        
        # Ambient term
        ambient = k_a
        
        L_i = ambient + diffuse + specular
        
        # Avoid division by zero or negative base for negative exponent
        if L_i <= 0:
            L_i = 1e-6
            
        # Calculate scaling factor
        # S_shd = S_all * I_shd * A_shd * L_i^(-n)
        S_shd = S_all * I_shd * A_shd * (L_i ** (-n))
        
        # Determine disk properties
        # Radius is inversely proportional to illumination intensity.
        # The source states radii are inversely proportional to illumination intensity.
        # S_shd is the scaling factor. The base radius is not explicitly defined, 
        # but S_shd scales the disk.
        
        disk = {
            'position': x_i,
            'scale': S_shd,
            'normal': n_i, # For point models, orientation might be based on surface normal or view
            'type': 'point'
        }
        rendered_disks.append(disk)
        
    return rendered_disks

def halftone_shading_polygonal(triangles, light_position, view_direction, params):
    """
    triangles: List of triangles, each defined by 3 vertices (v1, v2, v3)
    """
    S_all = params['S_all']
    I_shd = params['I_shd']
    A_shd = params['A_shd']
    n = params['n']
    k_a = params['k_a']
    k_d = params['k_d']
    k_s = params['k_s']
    shininess = params['shininess']
    
    rendered_disks = []
    
    for tri in triangles:
        v1, v2, v3 = tri
        
        # Compute triangle midpoint
        midpoint = (
            (v1[0] + v2[0] + v3[0]) / 3.0,
            (v1[1] + v2[1] + v3[1]) / 3.0,
            (v1[2] + v2[2] + v3[2]) / 3.0
        )
        
        # Compute triangle normal
        edge1 = (v2[0]-v1[0], v2[1]-v1[1], v2[2]-v1[2])
        edge2 = (v3[0]-v1[0], v3[1]-v1[1], v3[2]-v1[2])
        normal = cross_product(edge1, edge2)
        normal = normalize(normal)
        
        # Compute illumination at midpoint
        light_vec = (light_position[0] - midpoint[0], 
                     light_position[1] - midpoint[1], 
                     light_position[2] - midpoint[2])
        
        dot_nd_l = max(0, normal[0]*light_vec[0] + normal[1]*light_vec[1] + normal[2]*light_vec[2])
        diffuse = k_d * dot_nd_l
        
        dot_nv = max(0, normal[0]*view_direction[0] + normal[1]*view_direction[1] + normal[2]*view_direction[2])
        specular = k_s * (dot_nv ** shininess)
        
        ambient = k_a
        L_i = ambient + diffuse + specular
        
        if L_i <= 0:
            L_i = 1e-6
            
        S_shd = S_all * I_shd * A_shd * (L_i ** (-n))
        
        disk = {
            'position': midpoint,
            'scale': S_shd,
            'normal': view_direction, # Orthogonal to viewing direction
            'type': 'polygonal'
        }
        rendered_disks.append(disk)
        
    return rendered_disks

def cross_product(v1, v2):
    return (
        v1[1]*v2[2] - v1[2]*v2[1],
        v1[2]*v2[0] - v1[0]*v2[2],
        v1[0]*v2[1] - v1[1]*v2[0]
    )

def normalize(v):
    mag = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    if mag == 0:
        return (0, 0, 0)
    return (v[0]/mag, v[1]/mag, v[2]/mag)
```

**Edge Cases and Pitfalls:**
*   **Zero Illumination:** If $L_i$ is zero or negative, the term $L_i^{-n}$ becomes undefined or infinite. A guard value (e.g., $1e-6$) should be applied to $L_i$ before exponentiation [G].
*   **Negative Exponent:** The parameter $n$ controls the transition. If $n$ is negative, the relationship between illumination and disk size inverts. The source implies $n$ is positive to achieve "inversely proportional" behavior [S1, unofficial].
*   **Polygonal Orientation:** For polygonal models, disks are rendered orthogonal to the viewing direction, not the surface normal. This ensures the disks appear as circles in screen space, maintaining the halftone aesthetic regardless of surface orientation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of points or triangles, as each primitive requires one illumination calculation and one scaling factor computation.
*   **Memory Complexity:** Linear with respect to the number of primitives to store the disk data structures.
*   **Strengths:** Provides a clear, illustrative representation of lighting gradients. The use of discrete disks can help emphasize form and structure in anatomical or technical visualizations.
*   **Limitations:** The discrete nature of disks may introduce aliasing or a "stippled" appearance if the resolution is insufficient. The reliance on a specific lighting model (Phong) may not capture complex global illumination effects.
*   **When to Use:** Suitable for illustrative rendering where artistic style is prioritized over photorealism, particularly in medical or educational contexts.
*   **When Not to Use:** Not suitable for photorealistic rendering or scenes requiring accurate light transport simulation.

## Background
Halftone shading was introduced as one of the illustrative shading options in the GMS (Current Topics in Computer and Robot Assisted Surgery) journal article by [S1, unofficial]. It is designed to create expressive anatomical illustrations based on scanned patient data. The technique is part of a broader set of non-photorealistic rendering methods aimed at enhancing the clarity and aesthetic appeal of medical visualizations.

## Sources
[S1] GMS | GMS Current Topics in Computer and Robot Assisted Surgery | Expressive anatomical illustrations based on scanned patien..., https://www.egms.de/static/de/journals/curac/2006-1/curac000009.shtml, fetched 2026-10-30 (unofficial)
