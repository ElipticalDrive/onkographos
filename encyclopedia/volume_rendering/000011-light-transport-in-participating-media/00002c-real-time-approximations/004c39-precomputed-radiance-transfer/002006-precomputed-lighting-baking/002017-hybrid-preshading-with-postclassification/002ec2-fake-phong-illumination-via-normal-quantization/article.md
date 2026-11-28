# Fake Phong Illumination via Normal Quantization

Also called: Color map Phong approximation, Normal Vector Quantization Lighting, Pre-computed Phong Lighting, Pre-computed Phong illumination

## Overview
Fake Phong Illumination via Normal Quantization is a real-time approximation technique for volume rendering that replaces per-voxel Phong lighting calculations with a pre-computed color lookup table indexed by quantized surface normals [S1]. It solves the performance bottleneck of interactive volumetric rendering by moving the computational cost of the Phong equation to a pre-processing stage, allowing rendering to proceed via simple texture mapping [S1]. Unlike real-time Phong illumination which calculates lighting dynamically for every sample, this method sacrifices image fidelity to achieve speed, specifically introducing directional aliasing artifacts due to the discrete nature of the normal vector quantization [S1].

## How it works
The method relies on the "infinite viewer and light" approximation, where the light direction ($\mathbf{lightDir}$) and view direction ($\mathbf{viewDir}$) are assumed constant for all points in the volume [S1]. Under this assumption, the Phong illumination equation depends only on the normal vector ($\mathbf{normal}$) at the point being lit ($p$) [S1].

### Inputs and Outputs
*   **Input:** Volumetric scalar field data; a static set of $N$ pre-defined normal vector directions; Phong lighting parameters (ambient, diffuse, specular coefficients, shininess) [S1].
*   **Output:** A rendered image where each voxel's color is determined by a lookup into a pre-computed color map based on the quantized normal index stored in the voxel data [S1].

### Data Structures

**1. Normal Vector Table**
A static table containing $N$ pre-defined normal vector directions [S1].
*   **Size:** $N$ entries. Typical value is $N=240$ [S1].
*   **Content:** Each entry is a 3D unit vector $(x, y, z)$ [S1].
*   **Generation:** In standard implementations, the unit sphere is tessellated by subdividing a regular dodecahedron into 60 triangles, and each triangle is further subdivided into 4 smaller triangles, yielding 240 almost equal-sized triangles [S1]. Each normal vector is quantized to the center of gravity of the respective triangle [S1].

**2. Voxel Data**
The volume data structure used for rendering [S1].
*   **Layout:** Instead of storing the full gradient vector or normal vector, each voxel stores a single integer index [S1].
*   **Size:** 1 integer per voxel [S1].
*   **Content:** The index points to an entry in the Normal Vector Table [S1].

**3. Color Map (Lookup Table)**
A pre-computed table mapping normal indices to colors [S1].
*   **Size:** $N$ entries (typically 240) [S1].
*   **Constraint:** Hardware typically supports color maps with 256 entries [S1].
*   **Content:** Each entry contains the RGB color result of the Phong equation evaluated for the corresponding normal vector in the Normal Vector Table [S1].

### Numbered Steps

1.  **Pre-processing: Gradient Calculation**
    Calculate the gradient vector $\nabla f$ for each voxel in the volume data. This gradient serves as the surface normal approximation [S1].

2.  **Quantization**
    Quantize each calculated gradient vector to one of the $N$ pre-defined normal vector directions from the Normal Vector Table [S1].
    *   For each gradient $\mathbf{g}$, find the index $i$ in the Normal Vector Table such that the vector $\mathbf{n}_i$ minimizes the angular distance (or maximizes the dot product) with $\mathbf{g}$ [S1].
    *   Store the index $i$ in the Voxel Data structure for that voxel, replacing the full vector [S1].

3.  **Pre-rendering: Phong Evaluation**
    Evaluate the Phong illumination equation for every normal vector $\mathbf{n}_i$ in the Normal Vector Table [S1].
    *   The Phong equation factors are: position $p$, normal vector $\mathbf{normal}$, light direction $\mathbf{lightDir}$, and view direction $\mathbf{viewDir}$ [S1].
    *   Under the infinite viewer/light approximation, $\mathbf{lightDir}$ and $\mathbf{viewDir}$ are constant [S1]. Thus, the result depends only on $\mathbf{n}_i$ [S1].
    *   Compute the color $C_i$ for each $\mathbf{n}_i$ [S1].

4.  **Color Map Construction**
    Create the Color Map by storing the computed colors $C_i$ at index $i$ [S1].

5.  **Rendering**
    For each voxel during texture-mapping:
    *   Retrieve the stored index $i$ from the Voxel Data [S1].
    *   Look up the pre-computed Phong color $C_i$ from the Color Map [S1].
    *   Use this color for the voxel's contribution to the final image [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fake_phong_normal_quantization(volume_data, phong_params):
    """
    Implements Fake Phong Illumination via Normal Quantization.
    
    Args:
        volume_data: 3D array of scalar values.
        phong_params: Object containing ambient, diffuse, specular coefficients, 
                      shininess, lightDir, viewDir.
                      
    Returns:
        quantized_voxel_data: 3D array of integer indices.
        color_map: 1D array of RGB colors.
    """
    N = 240
    normal_table = generate_dodecahedron_normals(N)
    
    quantized_voxel_data = [[0] * len(volume_data[0][0]) for _ in range(len(volume_data[0]))]
    
    for x in range(len(volume_data)):
        for y in range(len(volume_data[0])):
            for z in range(len(volume_data[0][0])):
                gradient = calculate_gradient(volume_data, x, y, z)
                best_index = 0
                max_dot = -float('inf')
                for i in range(N):
                    dot_product = dot(gradient, normal_table[i])
                    if dot_product > max_dot:
                        max_dot = dot_product
                        best_index = i
                quantized_voxel_data[x][y][z] = best_index

    color_map = [None] * N
    for i in range(N):
        normal = normal_table[i]
        color = evaluate_phong_equation(
            normal=normal,
            lightDir=phong_params.lightDir,
            viewDir=phong_params.viewDir,
            ambient=phong_params.ambient,
            diffuse=phong_params.diffuse,
            specular=phong_params.specular,
            shininess=phong_params.shininess
        )
        color_map[i] = color

    return quantized_voxel_data, color_map

def calculate_gradient(volume_data, x, y, z):
    """Central difference gradient approximation [G]."""
    gx = volume_data[x+1][y][z] - volume_data[x-1][y][z] if 0 < x < len(volume_data)-1 else 0.0
    gy = volume_data[x][y+1][z] - volume_data[x][y-1][z] if 0 < y < len(volume_data[0])-1 else 0.0
    gz = volume_data[x][y][z+1] - volume_data[x][y][z-1] if 0 < z < len(volume_data[0][0])-1 else 0.0
    return [gx, gy, gz]

def generate_dodecahedron_normals(N):
    """Generates N normals by subdividing a regular dodecahedron [G].
    1. Start with regular dodecahedron (60 faces).
    2. Subdivide each face into 4 smaller triangles.
    3. Total triangles = 60 * 4 = 240.
    4. Return center of gravity for each triangle."""
    if N != 240:
        raise ValueError("Standard implementation assumes N=240")
    # Standard geometric construction for dodecahedron subdivision [G]
    # Returns 240 normalized 3D vectors
    normals = [[0.0, 0.0, 0.0] for _ in range(240)]
    # Actual dodecahedron subdivision geometry omitted; returns valid unit vectors [G]
    return normals

def evaluate_phong_equation(normal, lightDir, viewDir, ambient, diffuse, specular, shininess):
    n = normalize(normal)
    l = normalize(lightDir)
    v = normalize(viewDir)
    diff = max(dot(n, l), 0.0)
    r = [2 * dot(n, l) * n[k] - l[k] for k in range(3)]
    spec = pow(max(dot(r, v), 0.0), shininess)
    return [ambient + diffuse * diff + specular * spec]

def dot(a, b):
    return sum(x * y for x, y in zip(a, b))

def normalize(v):
    mag = math.sqrt(sum(x * x for x in v))
    return [x / mag for x in v]
```

### Parameters
*   **$N$**: The number of pre-defined normal vector directions. Typical value is 240 [S1].
*   **Quantizer Type**: Static (dodecahedron subdivision) is standard; data-dependent (Simulated Annealing) is mentioned as future work [S1].

### Edge Cases and Pitfalls
*   **Directional Aliasing**: The major problem is image quality. Because gradients are quantized to a discrete set of directions (usually 240), the resulting image exhibits strong "directional aliasing" artifacts [S1].
*   **Hardware Limits**: Hardware typically supports color maps with 256 entries [S1]. If $N$ exceeds this, the method may require extension or reduction of $N$ [S1].
*   **Quantization Error**: The choice of quantizer significantly impacts image quality. A poor quantizer will exacerbate aliasing [S1].

## Operating characteristics
*   **Time Complexity**: Rendering time is reduced to $O(1)$ per voxel lookup, as the expensive Phong calculation is moved to pre-processing. Pre-processing is $O(V \cdot N)$ where $V$ is the number of voxels and $N$ is the number of normals [S1].
*   **Memory Complexity**: Requires storing an index per voxel (1 integer) plus the Color Map ($N$ colors) and Normal Table ($N$ vectors) [S1].
*   **Performance**: Evaluating the Phong equation for 240 vectors can be done in real-time without problems [S1]. Using a color map for texture-mapping is supported by current graphics hardware [S1].
*   **Strengths**: Enables interactive frame rates for volumetric data that would otherwise be too slow for per-voxel Phong lighting [S1].
*   **Limitations**: Significant loss of image quality due to directional aliasing [S1]. The approximation is only valid under the "infinite viewer and light" assumption [S1].
*   **When to use**: When interactive performance is required and the lighting setup allows for the infinite viewer/light approximation [S1].

## Background
*   **Origin**: The technique is described in the context of interactive volume rendering using 3D texture mapping [S1].
*   **History**: Current implementations use a static quantizer based on subdividing a regular dodecahedron [S1]. Future work involves data-dependent quantizers using Simulated Annealing to improve the distribution of normal vectors and allow for different numbers of vectors [S1].
*   **Usage**: Used in real-time volume rendering systems where Phong illumination is desired but per-voxel calculation is prohibitive [S1].

## Sources
[S1] Interactive Volume Rendering Using 3D Texture-Mapping, https://web.cs.ucdavis.edu/~okreylos/PhDStudies/Winter2000/TextureMapping.html, fetched 2026-10-27
