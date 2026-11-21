# Solid Texturing

Also called: 3D Map Texturing, Procedural Solid Texturing, Sampling solid noise

## Overview
Solid texturing is a method for applying procedural noise functions to surfaces or volumetric data by evaluating a 3D noise field at spatial coordinates, rather than using 2D texture coordinates [S1]. It solves the problem of creating objects that appear carved from a continuous block of material, avoiding the seams and distortions associated with 2D texture mapping [S1]. Unlike 2D texturing, it maintains low memory consumption and allows the noise features to flow naturally across complex topologies [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D procedural noise function $N(x, y, z)$ (e.g., Perlin noise) and a set of 3D coordinates $\mathbf{p}$ representing either surface points or ray sampling locations [S1][S2, book].
*   **Output:** A scalar value (or color) assigned to each coordinate, representing the material property or color at that location [S1][S2, book].

**Data Structures:**
*   **Noise Function:** Typically implemented as a hash-based interpolation of gradient vectors at lattice points [S2, book]. Explicit pre-computed tables are avoided to prevent large memory costs [S1].
*   **Coordinates:** 3D vectors $(x, y, z)$ [S1].

**Steps:**
1.  **Generalize Noise to 3D:** Ensure the base noise function accepts 3D input coordinates. Most standard noise functions (e.g., Perlin) are naturally generalized to 3D and higher dimensions [S1].
2.  **Evaluate Noise:**
    *   **For Surface Texturing:** Evaluate the 3D noise function at the 3D world coordinates of each surface point [S1].
    *   **For Voxelization:** During the voxelization phase, assign a value to each voxel belonging to the object using the texturing function or 3D map. This value is stored as part of the voxel information [S2, book].
    *   **For Volume Enrichment:** If used to enrich a volume dataset without increasing stored resolution, evaluate the texturing function during rendering time at the ray sampling locations [S2, book].

**Formulas:**
The power spectrum of the noise on the surface, $P_{surface}(u, v)$, is related to the power spectrum of the solid noise, $P_{solid}(u, v, w)$, by the slice-projection theorem [S1]:
$$ P_{surface}(u, v) = \int_{-\infty}^{\infty} P_{solid}(u, v, w) \, dw $$
This integration occurs because evaluating a 3D noise along a surface corresponds to slicing the 3D domain [S1].

**Pitfalls and Edge Cases:**
*   **Loss of Band-Pass Properties:** Slicing a 3D band-pass noise along a surface does not result in a band-pass noise on the surface. This is a consequence of the slice-projection theorem, which states that slicing in one domain corresponds to projection or integration in the other domain [S1]. The resulting power spectrum is not band-pass anymore, potentially leading to visual artifacts or incorrect frequency characteristics [S1].
*   **Memory Costs:** Explicit noises that rely on pre-computed 3D tables induce large memory costs and are generally avoided in favor of procedural generation [S1].
*   **Seams and Distortions:** While solid texturing avoids seams inherent to 2D UV mapping, it can still exhibit distortions if the noise function is not carefully chosen or if the surface curvature is extreme [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def solid_texture(surface_points, noise_function):
    """
    Applies solid texturing to a set of 3D surface points.
    
    Args:
        surface_points: List of 3D vectors (x, y, z) representing surface coordinates.
        noise_function: A callable that takes a 3D vector and returns a scalar value.
        
    Returns:
        List of scalar values representing the texture at each point.
    """
    texture_values = []
    for point in surface_points:
        # Evaluate the 3D noise function at the surface point's coordinates
        value = noise_function(point.x, point.y, point.z)
        texture_values.append(value)
    return texture_values

def solid_texture_volume_enrichment(ray_samples, noise_function):
    """
    Enriches a volume dataset by evaluating noise at ray sampling locations.
    
    Args:
        ray_samples: List of 3D vectors representing ray sampling locations in 3D space.
        noise_function: A callable that takes a 3D vector and returns a scalar value.
        
    Returns:
        List of scalar values representing the enriched texture at each sample location.
    """
    enriched_values = []
    for sample in ray_samples:
        # Evaluate the 3D noise function at the ray sampling location
        value = noise_function(sample.x, sample.y, sample.z)
        enriched_values.append(value)
    return enriched_values
```

## Subtopics

### Noise on Surface
Defining noise directly on the surface so features flow along curvatures. This is difficult in general but possible with sparse convolution noises [S1].

### Voxelization Texturing
Assigning noise values to voxels during the voxelization phase [S2, book].

### Ray Casting Texturing
Evaluating noise during rendering at ray sampling locations to enrich volume data [S2, book].

## Operating characteristics
*   **Memory:** Low memory consumption compared to explicit 3D texture maps, as the noise is generated procedurally [S1].
*   **Performance:** Computationally expensive if evaluated at every surface point or ray sample, as it requires evaluating the noise function for each point. However, it avoids the overhead of managing large 3D texture arrays [S1].
*   **Visual Quality:** Produces a natural, carved appearance that is difficult to achieve with 2D texturing [S1]. However, it may suffer from loss of band-pass characteristics on surfaces, leading to potential visual artifacts [S1].
*   **Scalability:** Scales well with resolution as it is resolution-independent, but performance depends on the complexity of the noise function and the number of evaluation points [S1].
*   **When to Use:** When a natural, continuous material appearance is desired, and memory is a constraint. Suitable for organic shapes and complex topologies [S1].
*   **When Not to Use:** When precise control over 2D texture coordinates is required, or when the loss of band-pass properties on surfaces is unacceptable without additional correction [S1].

## Background
The idea of sampling 3D noise on surfaces was introduced by Perlin [Per85] and Peachy [Pea85] and popularized as solid texturing [S1]. It has been widely used in computer graphics for creating procedural textures and modeling solid materials [S1][S2, book]. A comprehensive overview of solid texturing can be found in Dischler and Ghazanfarpour [DG01] [S1]. The slice-projection theorem, which explains the spectral properties of sliced noise, was established by Braun [Bra99] and Mallat [Mal93] [S1].

## Sources
[S1] A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S2] Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
