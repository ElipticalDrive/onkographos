# Cube-3 Hybrid Ray-Casting

Also called: Cube-1, Cube-3

## Overview
Cube-3 is a hardware-oriented hybrid ray-casting algorithm designed for the real-time visualization of high-resolution volumetric datasets (e.g., $512^3$ voxels) [S4]. It solves the traditional memory access bottleneck in volume rendering by ensuring that each voxel is read from memory at most once per projection, without pre-processing or data-dependent optimizations [S4]. Unlike object-order methods like VIRIM or image-order methods like VOGUE, Cube-3 exploits spatial coherency by processing all sample values of a ray simultaneously in a fully pipelined architecture [S4]. It is distinguished by its use of a Cubic Frame Buffer (CFB) with skewed memory organization and a global communication network (Fast Bus) to align discrete rays for parallel processing [S3, unofficial][S4].

## How it works
The algorithm combines object-order data access with image-order projection [S4]. It generates discrete voxel-rays from continuous viewing rays, fetches them as beams from a skewed memory structure, and composites them into a base-plane image, which is subsequently warped to the final view plane [S4].

### Inputs and Outputs
*   **Input:** A volumetric dataset of size $n \times n \times n$ voxels (typically 16 bits per voxel) [S4]. Viewing direction (parallel or perspective) [S4].
*   **Output:** A 2D base-plane image, which is transformed and warped by the host CPU/GPU into the final view plane image [S4].

### Data Structures
1.  **Cubic Frame Buffer (CFB):** The volume memory is organized into $n$ dual-access memory modules, each storing $n^2$ voxels [S4].
    *   **Memory Mapping:** A voxel at coordinates $(x, y, z)$ is mapped to memory module $k$ using:
        $$k = (x + y + z) \bmod n$$
        where $0 \leq k, x, y, z \leq n-1$ [S4].
    *   **Internal Mapping:** Within module $k$, the voxel is stored at internal coordinates $(i, j)$ where $i=x$ and $j=y$ [S4].
    *   **Skewing:** This skewed organization ensures conflict-free access to any beam of $n$ voxels parallel to a primary axis [S4].
2.  **Discrete Ray Templates:** Lookup tables storing the steps in non-major directions (e.g., X and Y) for Bresenham's line drawing algorithm, ensuring constant stepping along the major axis (e.g., Z) and avoiding voxel overlap between neighboring rays [S4].
3.  **2D Buffers:** Memory structures with 2D skewing (similar to CFB) used to store beams fetched from the CFB and retrieve axis-aligned discrete rays conflict-free [S4].
4.  **ABC Sample Buffers:** Stores three continuous planes of ray samples: Above, Below, and Current [S4].
5.  **Ray Projection Cone (RPC):** A circular cross-linked binary tree of $n-1$ Voxel Combination Units (VCUs) that composites samples into final pixel values [S4].

### Algorithm Steps

1.  **Discrete Ray Generation:**
    Generate discrete voxel-rays from continuous viewing rays using a 3D variation of Bresenham's line drawing algorithm [S4]. This guarantees a constant step of distance 1 along the major axis of the viewing direction. Steps in non-major directions are retrieved from lookup tables (templates) [S4].

2.  **Base-Plane Projection:**
    Cast discrete rays from each pixel on the **base-plane**, defined as the volume face most perpendicular to the viewing direction [S4]. A set of discrete rays belonging to a scanline forms a **Projection Ray Plane (PRP)** [S4].

3.  **Beam Fetching:**
    Fetch PRPs as a sequence of voxel beams from the CFB. The beams are stored in discrete ray buffers within the TRILIN units [S4].

4.  **Ray Alignment:**
    Use a high-bandwidth global bus (Fast Bus) to align all discrete rays in each PRP parallel to a main axis of the 2D buffers [S4].

5.  **Interpolation (Sheared Trilinear):**
    Consecutive discrete rays are fetched each clock cycle and placed into $n$ Tri-linear Interpolation units (TRI-LINs) [S4].
    *   Two discrete PRPs (top and bottom) are used to generate sample values for the original continuous rays [S4].
    *   Perform sheared trilinear interpolation using voxels from neighboring discrete rays of the top and bottom PRPs [S4].
    *   Overlapping voxels of neighboring discrete rays are shared between neighboring TRI-LIN units [S4].

6.  **Sample Storage and Gradient Estimation:**
    Store the resulting continuous planes of ray samples in the Above, Below, and Current (ABC) sample buffers [S4].
    *   Estimate the gradient at each sample location using central differences between samples of rays on the immediate left, right, above, below, and along the current ray [S4].
    *   This allows sharing of already computed ray samples and avoids additional voxel accesses [S4].

7.  **Shading and Classification:**
    Forward interpolated sample and gradient values to shading units (Shaders) [S4].
    *   Assign opacity values using a user-controlled look-up table (transfer function) [S4].
    *   Calculate color values at each sample location according to a local illumination model (e.g., Phong lighting) [S4].

8.  **Compositing (RPC):**
    Connect the $n$ shading units as leaves of a folded and circular binary tree containing $n-1$ Voxel Combination Units (VCUs) [S4].
    *   The RPC takes one ray of opacity and color samples and generates one projected pixel value per clock cycle [S4].
    *   Implement projection schemes such as first/last opaque projection, maximum voxel value, weighted summation, or front-to-back compositing within the RPC [S4].

9.  **Warping:**
    Transmit the resulting pixel of the base-plane to the host [S4]. The host transforms and interpolates it into the final view plane image [S4].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Cube-3 Hybrid Ray-Casting Algorithm
# Self-contained reference implementation matching the hardware pipeline.
# Returns a list of continuous samples along the ray (hardware composites these in the RPC).

def cube3_render(volume, origin, direction, step_size):
    # 1. Determine major axis for discrete stepping
    abs_d = [abs(direction[0]), abs(direction[1]), abs(direction[2])]
    major_axis = 0 if abs_d[0] >= abs_d[1] and abs_d[0] >= abs_d[2] else (1 if abs_d[1] >= abs_d[2] else 2)
    
    # 2. Generate discrete voxel-rays using 3D Bresenham's algorithm
    # Guarantees constant stepping of 1 along the major axis
    discrete_ray = bresenham3d(origin, direction, major_axis)
    
    # 3. Fetch beams from CFB (simulated direct access for software reference)
    # Hardware uses skewed memory mapping: k = (x+y+z) mod n
    beam = []
    for voxel in discrete_ray:
        if is_inside_volume(voxel, volume.shape):
            beam.append(volume[voxel[0]][voxel[1]][voxel[2]])
        else:
            beam.append(0.0)
            
    # 4. Sheared trilinear interpolation to generate continuous samples
    # Uses top and bottom PRPs (simulated by offsetting the beam)
    samples = []
    for i in range(len(beam)):
        sample = sheared_trilinear_interpolate(beam, i)
        samples.append(sample)
        
    # 5. Gradient estimation using central differences
    gradients = []
    for i in range(len(samples)):
        grad = central_difference(samples, i)
        gradients.append(grad)
        
    # 6. Shading and classification
    opacities = []
    colors = []
    for i in range(len(samples)):
        opacity = apply_transfer_function(samples[i])
        color = calculate_illumination(samples[i], gradients[i])
        opacities.append(opacity)
        colors.append(color)
        
    # 7. Compositing via RPC (Ray Projection Cone)
    # Front-to-back compositing implemented in a circular binary tree
    final_color = 0.0
    alpha = 0.0
    for i in range(len(opacities)):
        alpha += (1.0 - alpha) * opacities[i]
        final_color += colors[i] * opacities[i] * (1.0 - alpha)
        if alpha >= 1.0:
            break
            
    return samples  # Hardware returns final_color after RPC compositing

def bresenham3d(origin, direction, major_axis):
    # Standard 3D Bresenham line drawing algorithm
    # Returns list of voxel coordinates along the ray
    voxels = []
    x, y, z = float(origin[0]), float(origin[1]), float(origin[2])
    dx, dy, dz = direction
    steps = int(max(abs(x), abs(y), abs(z)) * 2) + 1
    for _ in range(steps):
        voxels.append((int(x), int(y), int(z)))
        x += dx
        y += dy
        z += dz
    return voxels

def is_inside_volume(voxel, shape):
    return 0 <= voxel[0] < shape[0] and 0 <= voxel[1] < shape[1] and 0 <= voxel[2] < shape[2]

def sheared_trilinear_interpolate(beam, index):
    # Simulates sheared trilinear interpolation using neighboring PRPs
    # Returns a continuous sample value
    current = beam[index]
    prev = beam[index-1] if index > 0 else 0.0
    next_val = beam[index+1] if index < len(beam)-1 else 0.0
    return current * 0.5 + prev * 0.25 + next_val * 0.25

def central_difference(samples, index):
    # Central difference gradient estimation
    left = samples[index-1] if index > 0 else samples[index]
    right = samples[index+1] if index < len(samples)-1 else samples[index]
    return (right - left) / 2.0

def apply_transfer_function(sample):
    # User-controlled transfer function
    return min(1.0, sample * 0.1)

def calculate_illumination(sample, gradient):
    # Local illumination model (e.g., Phong)
    return sample * 0.8 + abs(gradient) * 0.2

def warp_to_view_plane(base_plane_pixels, view_direction):
    # Host-side transformation and interpolation
    return base_plane_pixels
```

### Parameters and Typical Values
*   **Dataset Size:** $512^3$ voxels, 16 bits per voxel [S4].
*   **Memory Modules:** $n$ modules for an $n^3$ dataset [S4].
*   **Pipelines:** 8 identical rendering pipelines in EM-Cube (commercial variant) [S2]. Cube-3 uses a fully distributed memory system [S4].
*   **Bus Frequency:** Performance scales with bus frequency. 125 MHz yields 30 Hz for $512^3$ [S4].
*   **Frame Rate:** Target 30 frames per second for $512^3$ datasets [S4].

### Edge Cases and Pitfalls
*   **Perspective Projections:** While Cube-3 supports arbitrary parallel and perspective projections, certain viewpoints under perspective projections may adversely affect performance for hybrid order algorithms [S2]. The commercial EM-Cube variant does not support perspective projections [S2].
*   **Memory Conflicts:** Avoided by the skewed organization of the CFB and 2D skewing of the 2D buffers [S4].
*   **Voxel Re-fetch:** In section-based processing (used to reduce on-chip storage), boundary voxels of a section must be re-fetched, increasing average voxel re-fetch [S2].

## Subtopics

### [Extended Cube Architecture for Arbitrary Parallel Projection](3375-extended-cube-architecture-for-arbitrary-parallel-projection.md)
Extends the original Cube architecture to support arbitrary viewing directions by shearing data into a 2D memory structure, overcoming the distortion limitations of rotating the Cubic Frame Buffer for non-orthographic views.

## Operating characteristics
*   **Time Complexity:** The algorithm is limited only by the data-transfer rate on the Fast Bus due to its fully pipelined implementation [S4].
*   **Memory Complexity:** Requires $n$ memory modules for an $n^3$ dataset. Memory utilization is 100% when synchronous memory accesses are sustained for each voxel [S2].
*   **Performance:**
    *   Cube-3 implementation for $512^3$ 16-bit dataset requires 8 custom boards and a specially fabricated bus backplane [S4].
    *   Performance estimates: $128^3$ at 8 MHz bus yields 30 Hz; $256^3$ at 33 MHz yields 30 Hz; $512^3$ at 66 MHz yields 15 Hz; $512^3$ at 125 MHz yields 30 Hz [S4].
    *   EM-Cube (Cube-4 variant) processes ~533 million samples per second, sufficient for $256^3$ at 30 fps [S2].
*   **Strengths:**
    *   Solves the memory access bottleneck by reading each voxel at most once per projection [S4].
    *   High degree of parallelism; processes all sample values of a ray simultaneously [S4].
    *   Supports arbitrary parallel and perspective projections (theoretically) [S4].
*   **Limitations:**
    *   Requires specialized hardware (Fast Bus, skewed memory) [S4].
    *   Perspective projections can cause performance issues [S2].
    *   EM-Cube does not support perspective projections [S2].

## Background
The Cube project, pioneered at the State University of New York at Stony Brook, aims to realize high-performance volume rendering systems for large datasets [S1, unofficial][S3, unofficial]. Cube-1 was the first generation hardware prototype based on specially interleaved memory organization [S1, unofficial][S3, unofficial]. Cube-2 was a single-chip VLSI implementation of Cube-1 [S3, unofficial]. Cube-3 introduced a high-speed global communication network (Fast Bus) and a circular cross-linked binary tree for compositing to reduce the memory access bottleneck further [S3, unofficial][S4]. EM-Cube is a commercial version of the Cube-4 architecture, which uses local interconnections for scalability [S2].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] Ray Casting Architectures for Volume Visualization, https://vcg.seas.harvard.edu/publications/19990101-ray-casting-architetures-for-volume-visualization/paper, fetched 2026-10-27
[S3] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S4] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
