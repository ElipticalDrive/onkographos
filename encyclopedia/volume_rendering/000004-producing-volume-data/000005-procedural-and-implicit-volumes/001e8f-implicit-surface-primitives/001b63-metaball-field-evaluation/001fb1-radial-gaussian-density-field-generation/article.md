# Radial Gaussian Density Field Generation
Also called: Caffeine Molecule Volume Generation, Sum of Radial Gaussians

## Overview
Generates a 3D scalar density field for volume rendering by summing normalized radial Gaussian functions centered at predefined atomic positions, then scaling and clamping the result to a [0,1] range [S1, unofficial]. It solves the problem of converting discrete atomic coordinates into a continuous volumetric representation suitable for surface extraction [S1, unofficial]. Unlike [Metaball Field Evaluation](1017-metaball-field-evaluation.md), which evaluates a dynamic scalar field composed of moving metaballs, this method uses static, smooth Gaussian falloffs centered on fixed points [S1, unofficial].

## How it works
**Inputs:**
- Volume dimensions: `dimX`, `dimY`, `dimZ` (integers).
- Atom definitions: A list of atoms, each containing center coordinates $(c_X, c_Y, c_Z)$ and a variance parameter $\sigma^2$.
- Parameters: `postDensityScale` (float, default 2.5), `isoValue` (float, default 0.5, clamped to [0,1]).

**Outputs:**
- A 3D volume array of type `uint16_t` (16-bit), storing density values scaled to the range $[0, 65535]$.

**Data Structures:**
The `RadialGaussian` class stores:
- `cX`, `cY`, `cZ`: Center coordinates (float).
- `normalization`: Precomputed constant $N = \frac{1}{\sqrt{2\pi \sigma^2}}$ (float).
- `falloff`: Precomputed constant $F = -\frac{0.5}{\sigma^2}$ (float).

**Steps:**
1. Initialize volume dimensions (e.g., $128 \times 128 \times 128$) and allocate memory for `dimX * dimY * dimZ` elements.
2. Compute inverse dimensions:
   $$ invDimX = \frac{1.0}{dimX - 1}, \quad invDimY = \frac{1.0}{dimY - 1}, \quad invDimZ = \frac{1.0}{dimZ - 1} $$
3. Define atom centers and variance-derived scales. For the caffeine molecule example, a scale factor $s = 0.1$ and offsets $o_X=0.5, o_Y=0.5, o_Z=0.5$ are applied to raw coordinates [S1, unofficial].
4. For each voxel $(z, y, x)$:
   a. Compute normalized coordinates:
      $$ n_X = x \cdot invDimX, \quad n_Y = y \cdot invDimY, \quad n_Z = z \cdot invDimZ $$
   b. Initialize density $\rho = 0.0$.
   c. For each atom $i$ with center $(c_{X,i}, c_{Y,i}, c_{Z,i})$ and precomputed constants $N_i, F_i$:
      i. Compute squared distance from normalized voxel position to atom center:
         $$ d^2 = (n_X - c_{X,i})^2 + (n_Y - c_{Y,i})^2 + (n_Z - c_{Z,i})^2 $$
      ii. Add contribution to $\rho$:
          $$ \rho \leftarrow \rho + N_i \cdot e^{F_i \cdot d^2} $$
   d. Apply global scaling:
      $$ \rho \leftarrow \rho \cdot postDensityScale $$
   e. Clamp $\rho$ to $[0.0, 1.0]$.
   f. Convert to 16-bit integer:
      $$ value = \rho \cdot 65535 $$
   g. Store `value` in the volume array at index corresponding to $(x, y, z)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class RadialGaussian:
    def __init__(self, cx, cy, cz, variance):
        self.cx = cx
        self.cy = cy
        self.cz = cz
        self.normalization = 1.0 / math.sqrt(2.0 * math.pi * variance)
        self.falloff = -0.5 / variance

    def eval(self, nx, ny, nz):
        dx = nx - self.cx
        dy = ny - self.cy
        dz = nz - self.cz
        dist_sq = dx*dx + dy*dy + dz*dz
        return self.normalization * math.exp(self.falloff * dist_sq)

def generate_volume(dim_x, dim_y, dim_z, atoms, post_density_scale=2.5):
    # Initialize volume
    volume_size = dim_x * dim_y * dim_z
    volume = [0] * volume_size
    
    inv_dim_x = 1.0 / (dim_x - 1)
    inv_dim_y = 1.0 / (dim_y - 1)
    inv_dim_z = 1.0 / (dim_z - 1)
    
    max_uint16 = 65535
    
    for z in range(dim_z):
        for y in range(dim_y):
            for x in range(dim_x):
                # Normalized coordinates
                nx = x * inv_dim_x
                ny = y * inv_dim_y
                nz = z * inv_dim_z
                
                rho = 0.0
                
                # Sum contributions from all atoms
                for atom in atoms:
                    rho += atom.eval(nx, ny, nz)
                
                # Scale and clamp
                rho *= post_density_scale
                if rho < 0.0:
                    rho = 0.0
                elif rho > 1.0:
                    rho = 1.0
                
                # Convert to uint16
                volume[z * dim_y * dim_x + y * dim_x + x] = int(rho * max_uint16)
                
    return volume
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** If `dimX`, `dimY`, or `dimZ` is 1, `invDim` becomes undefined. The source assumes dimensions $>1$ [S1, unofficial].
- **Precision:** Floating-point precision in `exp()` and distance calculations may affect density accuracy, especially for small variances [S1, unofficial].
- **Clamping:** Values exceeding 1.0 are clamped, potentially losing density information for very high concentrations [S1, unofficial].

## Subtopics

### [Particle Density Field Computation](4443-particle-density-field-computation.md)
Particle Density Field Computation generates a 3D scalar density field by evaluating a radial kernel function for every voxel in a 3D grid, summing contributions from a set of particles. It solves the problem of converting discrete particle data into a continuous volumetric representation suitable for ray marching or marching cubes. Unlike [Radial Gaussian Density Field Generation](2081-radial-gaussian-density-field-generation.md), which uses smooth Gaussian falloffs, this method employs a specific polynomial kernel with a sharp cutoff at a normalized squared distance of 0.5, resulting in a compact support that strictly excludes distant particles.

## Operating characteristics
- **Time Complexity:** $O(V \cdot A)$, where $V$ is the number of voxels and $A$ is the number of atoms. Each voxel requires evaluating the Gaussian function for every atom.
- **Memory Complexity:** $O(V)$ for storing the volume data. For a $128^3$ grid, this is $128^3 \times 2$ bytes $\approx 32$ MB [S1, unofficial].
- **Performance:** The example implementation processes a $128^3$ volume in a few seconds on a standard CPU [S1, unofficial].
- **Strengths:** Simple to implement; produces smooth, continuous density fields; suitable for static molecular structures.
- **Limitations:** Computationally expensive for large numbers of atoms or high-resolution volumes; no spatial acceleration structure (e.g., BVH, grid) is mentioned, leading to $O(V \cdot A)$ complexity [S1, unofficial].
- **When to Use:** Static, small-to-medium scale molecular visualization where smooth density is desired.
- **When Not to Use:** Large-scale particle systems or dynamic scenes requiring real-time performance.

## Background
Originated from the `dualmc` project by Dominik Wodniok, which demonstrates dual marching cubes for volume rendering [S1, unofficial]. The specific example uses the 3D structure of the caffeine molecule from PubChem [S1, unofficial]. It is used for generating manifold surfaces from atomic data.

## Sources
[S1] dominikwodniok/dualmc, https://github.com/dominikwodniok/dualmc, fetched 2026-10-26 (unofficial)
