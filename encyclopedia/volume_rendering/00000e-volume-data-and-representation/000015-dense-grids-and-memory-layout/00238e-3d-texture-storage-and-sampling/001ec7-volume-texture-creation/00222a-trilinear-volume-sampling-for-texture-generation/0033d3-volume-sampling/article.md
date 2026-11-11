# Volume Sampling

Also called: Scalar Density Sampling, 3D Texture Density Fetch

## Overview
Volume Sampling retrieves the scalar density value from a 3D texture at given normalized texture coordinates [S1, unofficial]. It serves as the fundamental data retrieval primitive for raycasting-based volume rendering, providing the raw intensity values required for ray marching, isosurface detection, and maximum intensity projection [S1, unofficial]. Unlike its parent method, which handles coordinate transformation and interpolation, this method focuses strictly on the final fetch operation from the sampled texture resource.

## How it works
**Inputs:**
- `volume_data`: A 3D sampler texture containing the volume intensity data [S1, unofficial].
- `texcoords`: A 3D vector of normalized texture coordinates $(u, v, w)$, typically in the range $[0, 1]$ [S1, unofficial].

**Outputs:**
- `density`: A scalar float representing the density value at the specified coordinates [S1, unofficial].

**Data Structures:**
- The `volume_data` is assumed to be a GPU texture object (e.g., `sampler3D` in GLSL) bound to a texture unit. The internal format is not specified in the sources, but the retrieval logic assumes a multi-channel structure where the density is stored in the red channel [S1, unofficial].

**Steps:**
1. Perform a texture lookup using the provided normalized coordinates `texcoords` on the `volume_data` texture sampler [S1, unofficial].
2. Extract the red channel component from the resulting 4D vector (RGBA) [S1, unofficial].
3. Return the red channel value as the scalar density [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_sample(volume_data_sampler, texcoords):
    """
    Samples the scalar density value from a 3D texture at given normalized texture coordinates.
    
    Args:
        volume_data_sampler: The 3D texture sampler object.
        texcoords: A 3D vector (u, v, w) in normalized texture space [0, 1].
        
    Returns:
        float: The density value from the red channel of the texture sample.
    """
    # Perform the texture fetch. The specific API call depends on the shading language.
    # In GLSL: texture(volume_data_sampler, texcoords)
    # In HLSL: tex3D(volume_data_sampler, texcoords)
    sample_result = texture(volume_data_sampler, texcoords)
    
    # Return the red channel component
    return sample_result.r
```

**Parameters:**
- `volume_data`: The 3D texture resource [S1, unofficial].
- `texcoords`: The sampling coordinates [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** The sources assume coordinates are within $[0, 1]$. Behavior outside this range depends on the texture wrapping mode (clamp, repeat, border), which is not specified [S1, unofficial].
- **Channel Assumption:** The method explicitly assumes density is stored in the red channel. If the texture format uses a different channel or interleaved data, this method will return incorrect values [S1, unofficial].

## Subtopics

### Chunk Density Field Sampling
A method to populate a chunk's density grid by sampling a global density field at discrete 3D coordinates within the chunk's bounds [S2, unofficial]. It differs from standard texture sampling by iterating through a local grid resolution to build a chunk-specific data structure rather than fetching a single point [S2, unofficial]. Choose this when pre-computing or streaming chunk-based voxel data from a continuous field.
- **Definition:** Populate a chunk's density grid by sampling a global density field at discrete 3D coordinates within the chunk's bounds [S2, unofficial].
- **Steps:** Iterate through all sample coordinates $(sampleX, sampleY, sampleZ)$ from $0$ to $CHUNK\_SAMPLE\_SIZE-1$, compute the world position for each sample, sample the density field at that position, and store the result in the chunk [S2, unofficial].
- **Parameters:** $CHUNK\_SAMPLE\_SIZE$: The resolution of the density grid within each chunk, used as the upper bound for all three sample loops [S2, unofficial].

### [Custom Volume Sampling via GLSL Injection](7156-custom-volume-sampling-via-glsl-injection.md)
Custom Volume Sampling via GLSL Injection is a mechanism that overrides the default volume sampling logic by injecting a user-defined GLSL function into the fragment shader. It distinguishes itself from standard trilinear sampling by decoupling the data retrieval logic from the texture fetch, enabling dynamic, procedural, or external-data-driven volume evaluation directly within the rendering pipeline. Choose this when standard texture fetches are insufficient for procedural or time-varying density calculations.

### [3D Texture Volume Render](10782-3d-texture-volume-render.md)
The 3D Texture Volume Render node renders volumetric textures from 3D data to create cloud and fog effects by integrating light through a volume defined by a 3D texture and its corresponding signed distance field (SDF). It differs from standard ray marching on raw voxel grids by operating on a specific compressed texture layout (16x16 grid of 256 slices) and providing built-in lighting models (directional + hemispherical skylight) and post-processing features like dithering and ground planes. It builds on Volume Sampling for density retrieval.

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it involves a single texture lookup and channel extraction [S1, unofficial].
- **Memory Complexity:** Depends on the size of the `volume_data` texture, which is not specified [S1, unofficial].
- **Strengths:** Extremely fast, hardware-accelerated via GPU texture units [S1, unofficial].
- **Limitations:** Limited to the data format and coordinate space of the bound texture; does not handle coordinate transformation or interpolation itself [S1, unofficial].
- **When to use:** When raycasting or marching through a pre-loaded 3D texture and only the density value is needed at a specific point [S1, unofficial].
- **When not to use:** When procedural generation, custom interpolation, or non-texture-based data sources are required [S1, unofficial].

## Background
This method originates from direct volume rendering approaches using raycasting, where the fragment shader calculates intersections with a unit cube and marches along the view ray, sampling from the 3D texture along the way [S1, unofficial]. The specific implementation cited uses a viridis transfer function for coloring and PBR lighting models for surface calculations, with the density sample serving as the primary input for these subsequent steps [S1, unofficial].

## Sources
[S1] Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, https://github.com/Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, fetched 2026-10-26 (unofficial)
[S2] aaron-eugene/PlaneVoxelLab, https://github.com/aaron-eugene/PlaneVoxelLab, fetched 2026-10-27 (unofficial)
