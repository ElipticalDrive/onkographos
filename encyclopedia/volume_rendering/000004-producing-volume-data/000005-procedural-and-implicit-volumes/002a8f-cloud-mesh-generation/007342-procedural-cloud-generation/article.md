# Procedural Cloud Generation
Also called: Gabor and Gaussian Kernel Cloud Generation
## Overview
Procedural Cloud Generation is a method for synthesizing volumetric cloud assets using a hierarchical structure of "cloud chunks" composed of Gaussian and Gabor kernels [S1]. It solves the problem of generating realistic, multi-scale cloud formations (cores, splittings, wispy effects) without relying on regression-based training or manual sculpting [S1]. It differs from [Ellipsoid Fitting and Rotation for Cloud Meshes](13371-ellipsoid-fitting-and-rotation-for-cloud-meshes.md) by explicitly modeling microstructure and wind-induced wisps via orientation-selective Gabor kernels on chunk surfaces, rather than fitting static geometric primitives [S1].
## How it works
**Inputs:** Random seed, parameters for kernel distributions (uniform for frequency/orientation, Maxwell for scale/radius), grid resolution for chunk surfaces.
**Outputs:** A hierarchical tree of cloud chunks, where each chunk contains a set of 3D kernels (Gaussian core + Gabor surface kernels) with defined positions, orientations, frequencies, and scales.

**Data Structures:**
1.  **Cloud Chunk:**
    *   `center`: `vec3` (position in world space).
    *   `radius`: `float` (base size, drawn from Maxwell distribution for child chunks).
    *   `surface_grid`: `2D array` or `list` of cells covering the chunk's surface area.
    *   `kernels`: `list` of Kernel objects.
2.  **Kernel:**
    *   `type`: `enum` (GAUSSIAN, GABOR).
    *   `position`: `vec3` (3D coordinate).
    *   `orientation`: `vec3` or `float` (angle/direction).
    *   `frequency`: `float` (spatial frequency).
    *   `scale`: `float` (width/extent).
    *   `amplitude`: `float` (optional, default 1.0).
3.  **Chunk Tree:**
    *   `parent`: `Cloud Chunk` or `null`.
    *   `children`: `list` of `Cloud Chunk`.

**Steps:**

1.  **Define Base Chunk:** Create a root cloud chunk. Model its core using a single Gaussian kernel centered at `chunk.center` with a base radius [S1].
2.  **Surface Grid Generation:** Define a grid on the surface of the cloud chunk. The grid cells partition the surface area [S1].
3.  **Generate Gabor Kernels:** For each grid cell:
    a.  Select a random position `p` uniformly within the grid cell on the surface [S1].
    b.  Sample orientation `o` from a uniform distribution [S1].
    c.  Sample frequency `f` from a uniform distribution [S1].
    d.  Sample scale `s` from a Maxwell distribution [S1].
    e.  Create a Gabor kernel at `p` with parameters `o`, `f`, `s` [S1].
    f.  Add the kernel to the chunk's kernel list [S1].
4.  **Recursive Chunk Placement (Fractal Layers):**
    a.  To generate smaller clouds on the surface of a parent chunk (simulating inner pressure breakthrough [S1]):
    b.  Sample a radius `r_child` for the child chunk from a Maxwell distribution [S1].
    c.  Place the child chunk's center uniformly at random on the surface of the parent chunk [S1].
    d.  Recursively apply Steps 2-3 to the child chunk [S1].
5.  **Merge Formations (Optional):** To create complex merged clouds, generate multiple independent multi-layer cloud trees and translate them together in world space [S1].

**Formulas:**
*   **Maxwell Distribution for Scale/Radius:** The scale $s$ and child radius $r$ are drawn from a Maxwell distribution [S1]. The probability density function is:
    $$ f(x; \sigma) = \sqrt{\frac{2}{\pi}} \frac{x^2}{\sigma^2} e^{-\frac{x^2}{2\sigma^2}} $$
    where $\sigma$ is the scale parameter controlling the spread [S1]. The specific value of $\sigma$ is not given in the sources [S1].
*   **Uniform Distribution for Orientation/Frequency:** Orientation $\theta$ and frequency $\omega$ are drawn from $U(a, b)$ [S1]. The bounds $a, b$ are not given in the sources [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math
import numpy as np

def maxwell_sample(sigma):
    """Sample from Maxwell distribution with scale parameter sigma."""
    # Using Box-Muller or similar to generate normal variables
    # Maxwell distribution is the distribution of the magnitude of a 3D vector with normal components
    x = random.gauss(0, sigma)
    y = random.gauss(0, sigma)
    z = random.gauss(0, sigma)
    return math.sqrt(x*x + y*y + z*z)

def uniform_sample(a, b):
    return random.uniform(a, b)

class Kernel:
    def __init__(self, pos, orient, freq, scale, k_type):
        self.position = pos
        self.orientation = orient
        self.frequency = freq
        self.scale = scale
        self.type = k_type

class CloudChunk:
    def __init__(self, center, radius):
        self.center = center
        self.radius = radius
        self.kernels = []
        self.children = []

    def generate_surface_kernels(self, grid_resolution, sigma_scale, sigma_freq, sigma_orient):
        """
        Generate Gabor kernels on the surface grid.
        Note: Exact grid definition and surface parameterization are not specified in sources [S1].
        We assume a simplified surface sampling for implementation completeness [G].
        """
        # Simplified: Sample points on a sphere surface
        num_points = grid_resolution * grid_resolution
        for _ in range(num_points):
            # Random point on sphere surface
            theta = random.uniform(0, 2 * math.pi)
            phi = random.acos(2 * random.random() - 1)
            x = self.radius * math.sin(phi) * math.cos(theta)
            y = self.radius * math.sin(phi) * math.sin(theta)
            z = self.radius * math.cos(phi)
            pos = (self.center[0] + x, self.center[1] + y, self.center[2] + z)
            
            # Sample parameters
            orient = uniform_sample(0, 2 * math.pi) # Orientation angle [S1]
            freq = uniform_sample(sigma_freq[0], sigma_freq[1]) # Frequency [S1]
            scale = maxwell_sample(sigma_scale) # Scale [S1]
            
            kernel = Kernel(pos, orient, freq, scale, "GABOR")
            self.kernels.append(kernel)

    def add_child_chunk(self, sigma_radius):
        """Add a smaller child chunk on the surface."""
        radius_child = maxwell_sample(sigma_radius)
        # Place uniformly on surface
        theta = random.uniform(0, 2 * math.pi)
        phi = random.acos(2 * random.random() - 1)
        x = self.radius * math.sin(phi) * math.cos(theta)
        y = self.radius * math.sin(phi) * math.sin(theta)
        z = self.radius * math.cos(phi)
        child_center = (self.center[0] + x, self.center[1] + y, self.center[2] + z)
        
        child = CloudChunk(child_center, radius_child)
        self.children.append(child)
        return child

def generate_cloud(depth, sigma_radius, sigma_scale, sigma_freq, sigma_orient, grid_res):
    """Generate a hierarchical cloud structure."""
    root = CloudChunk((0,0,0), 1.0) # Base radius 1.0 [G]
    root.generate_surface_kernels(grid_res, sigma_scale, sigma_freq, sigma_orient)
    
    if depth > 0:
        # Add children to root
        num_children = 5 # Not given in sources [G]
        for _ in range(num_children):
            child = root.add_child_chunk(sigma_radius)
            child.generate_surface_kernels(grid_res, sigma_scale, sigma_freq, sigma_orient)
            # Recurse
            generate_cloud(depth - 1, sigma_radius, sigma_scale, sigma_freq, sigma_orient, grid_res)
    return root
```

**Edge Cases and Pitfalls:**
*   **Precision:** Gabor kernels are not orthonormal, which may cause artifacts if not handled carefully during rendering or merging [S1].
*   **Boundaries:** The exact bounds for the uniform distributions of frequency and orientation are not specified [S1].
*   **Degenerate Input:** If the grid resolution is too low, the wispy effect may be lost; if too high, computational cost increases significantly [G].
*   **Merging:** Merging clouds by translation may cause visual artifacts at the boundary if the underlying density fields are not blended smoothly [S1].

## Subtopics

### Procedural Cloud Authoring via Tree of Chunks
An interactive pipeline for generating volumetric clouds using a hierarchical structure of procedural chunks, enabling direct rendering without regression [S1]. It differs from the fully procedural method by allowing user interaction to place kernels and guide shapes [S1]. Choose this when artistic control is required over the final cloud shape [S1]. It uses a base cloud chunk modeled as a textured ellipsoid to capture microstructure [S1]. Chunks are organized into a tree to model multi-level formations (e.g., smaller clouds forming on surfaces due to inner pressure) [S1]. Users place kernels in 3D space interactively using mouse input to achieve desired shapes [S1]. A real-time tomography image of the generated cloud supports this process [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Regression-based methods take 0.1-2h on a single RTX 4090 [S1]. Procedural generation is implied to be faster as it is a forward method [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Captures microstructure and wind-induced wisps; allows for complex multi-layer formations; interactive authoring possible [S1].
*   **Limitations:** Gabor kernels are not orthonormal, which may complicate certain mathematical operations [S1]. The method may require fine-tuning between assets of substantially different classes if regression is used in conjunction [S1].
*   **When to use:** When realistic, multi-scale cloud formations are needed without the cost of regression training [S1].
*   **When not to use:** When strict orthonormality is required for mathematical stability [S1].

## Background
Originated from the paper "Gabor Fields: Orientation-Selective Level-of-Detail for Volume Rendering" [S1]. Inspired by gradient noise density erosion common in industry [Fajardo and Pharr 2023; Wrenninge et al. 2011] and earlier work on cloud modeling [Bouthors and Neyret [2004]; Dobashi et al. [2000]]. The method uses Gabor noise concepts [Galerne et al. 2012; Lagae et al. 2009] and Maxwell distribution for physical intuition [Wu and McFarquhar 2018].

## Sources
[S1] Gabor Fields: Orientation-Selective Level-of-Detail for Volume Rendering, https://arxiv.org/abs/2602.05081, fetched 2026-10-27
