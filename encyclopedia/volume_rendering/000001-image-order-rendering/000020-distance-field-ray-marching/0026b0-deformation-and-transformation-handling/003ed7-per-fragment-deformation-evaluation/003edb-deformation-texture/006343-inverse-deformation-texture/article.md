# Inverse Deformation Texture
Also called: Inverse Free-Form Volumetric Deformation, Inverse Parameterization, Inverse Parameterization for Volumetric Deformation

## Overview
Inverse Parameterization (also known as Inverse Deformation Texture) is a technique used when forward free-form volumetric deformation is impossible to solve analytically or efficiently in real-time [S1, S2, unofficial]. It constructs a lattice grid around the deformed volume in world space and associates each lattice point with texture coordinates in the unit fire volume, allowing efficient evaluation of the deformation via trilinear interpolation in fragment shaders [S1, S2, unofficial]. It sets itself apart from forward deformation methods by avoiding the need to invert the deformation function at runtime, which is computationally prohibitive for infinite-detail volumes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A deformed volume (e.g., fire) defined by a set of forces or a simulation in world space.
- Output: A 3D texture (Inverse Deformation Texture) that maps world-space positions to unit-volume texture coordinates.

**Data Structures:**
- **Lattice Grid:** A 3D regular grid defined in world space coordinates surrounding the deformed volume [S1, S2, unofficial].
- **Inverse Deformation Texture:** A 3D texture where each texel stores the texture coordinates $(u, v, w)$ in the unit fire volume corresponding to the world-space position of the lattice point [S1, S2, unofficial].

**Steps:**
1. **Construct Lattice:** Build a lattice grid around the deformed volume in world space coordinates [S1, S2, unofficial].
2. **Associate Coordinates:** For each point in the lattice, determine its corresponding texture coordinates in the unit fire volume [S1, S2, unofficial]. This effectively pre-computes the inverse mapping from world space to the reference parameter space.
3. **Interpolate:** Use a 3D shader flame texture (the reference volume) to implicitly interpolate all points of the fire surfaces by sampling the Inverse Deformation Texture at the fragment's world-space position [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_inverse_deformation_texture(lattice_min, lattice_max, resolution):
    step = (lattice_max - lattice_min) / resolution
    texture = [[[] for _ in range(resolution)] for _ in range(resolution)]
    for i in range(resolution):
        for j in range(resolution):
            for k in range(resolution):
                world_pos = (lattice_min.x + i * step.x,
                             lattice_min.y + j * step.y,
                             lattice_min.z + k * step.z)
                # Pre-compute the inverse mapping from world space to the unit fire volume
                # The exact analytical inverse is unavailable; this step uses the pre-computed lattice association [G]
                tex_coords = world_pos_to_unit_volume(world_pos)
                texture[i][j][k] = tex_coords
    return texture

def render_fragment(world_pos, texture, flame_texture, resolution, lattice_min, step):
    # Locate the lattice cell containing the fragment's world position
    i = int((world_pos[0] - lattice_min.x) / step.x)
    j = int((world_pos[1] - lattice_min.y) / step.y)
    k = int((world_pos[2] - lattice_min.z) / step.z)
    # Clamp indices to valid texture bounds
    i = max(0, min(i, resolution - 1))
    j = max(0, min(j, resolution - 1))
    k = max(0, min(k, resolution - 1))
    # Retrieve pre-computed texture coordinates
    tex_coords = texture[i][j][k]
    # Sample the 3D shader flame texture to implicitly interpolate fire surfaces [S1, unofficial]
    return flame_texture.sample(tex_coords)
```

**Parameters:**
- Lattice resolution: Not given in the sources.
- Reference volume size: Unit volume (1x1x1) [S1, unofficial].

**Edge Cases and Pitfalls:**
- Forward deformation cannot be solved analytically [S1, unofficial].
- Numerical solutions for forward deformation require significant computation and exhibit robustness problems [S1, unofficial].
- Real-time deformation of the entire infinite-detail volume is impossible [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Avoids real-time inversion of complex deformation functions [S1, unofficial]. Efficient evaluation in fragment shaders [S1, unofficial].
- **Limitations:** Requires pre-computation or setup. Accuracy depends on lattice resolution [G].

## Background
Origin: Developed by Y. Vanzine for real-time volumetric rendering of fire in production systems [S1, S2, unofficial]. The method is based on the lattice method described in [13] (referenced in [S1, unofficial]) and [9] (referenced in [S2, unofficial]).

## Sources
[S1] Real-Time Volumetric Rendering of Fire in a Production System, https://clas.iusb.edu/math-compsci/_prior-thesis/YVanzine_thesis.pdf, fetched 2026-10-27 (unofficial)
[S2] Realistic Real-Time Rendering of Fire in a Production System, https://clas.iusb.edu/math-compsci/_prior-proposals/YVanzine_Proposal.pdf, fetched 2026-10-28 (unofficial)
