# Kernel Density Estimation (KDE)

Also called: Parzen Window Estimation, Smooth Point Cloud Representation, Point cloud density volume rendering

## Overview
Kernel Density Estimation (KDE) volume rendering is a method that transforms a discrete point cloud into a smooth, continuous scalar field to reveal interior structures and contextual relationships that are obscured by high-frequency noise or occlusion in direct point visualization [S1, unofficial]. It sets itself apart from isosurface rendering, which misses interior features, and direct point rendering, which lacks continuity, by providing a volumetric representation where density gradients drive lighting and silhouette edge detection [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ particles (points) $P = \{p_1, p_2, \dots, p_N\}$ in 3D space, where $p_i = (p_{ix}, p_{iy}, p_{iz})$.
*   **Output:** A 3D density volume stored as a 3D texture. The RGB channels store the normalized gradient vector $\nabla\rho$, and the Alpha channel stores the scaled density value $\rho$ [S1, unofficial].

**Data Structures:**
1.  **KDTree:** A spatial partitioning structure built on the CPU containing the particle coordinates. This is uploaded to GPU memory for efficient neighbor queries [S1, unofficial].
2.  **Density Volume:** A 3D array (texture) of single-precision floating-point numbers. Dimensions correspond to the voxel grid resolution. Each voxel stores 4 floats: $(R, G, B, A)$ representing $(\nabla\rho_x, \nabla\rho_y, \nabla\rho_z, \rho)$ [S1, unofficial].

**Parameters:**
*   $k$: Constant multiplier for the kernel radius. Controls visual abstraction and smoothness. Typical value: $12$ [S1, unofficial].
*   $r$: Kernel radius. Calculated as $r = k \cdot r_{base}$, where $r_{base}$ is derived from the volume of the initial point cloud $V$ and number of points $N$: $r_{base} = \sqrt[3]{\frac{V}{N}}$ (solving for the radius of a sphere with volume $V/N$) [S1, unofficial].

**Algorithm Steps:**
1.  **KDTree Construction:** Spatially partition the input points $P$ into a KDTree on the CPU [S1, unofficial].
2.  **Upload:** Transfer the KDTree structure to GPU memory [S1, unofficial].
3.  **Thread Allocation:** Allocate one OpenCL thread for each voxel center in the target density volume grid [S1, unofficial].
4.  **Neighbor Query:** Each thread traverses the KDTree to find all particles $p_i$ such that the distance $\|p - p_i\| \le r$, where $p$ is the current voxel center [S1, unofficial].
5.  **Density and Gradient Calculation:** For each voxel center $p$, compute the density $\rho(p)$ and gradient $\nabla\rho(p)$ using the following formulas:
    $$
    \rho(p) = \sum_{\|p - p_i\| \le r} \left(1 - \frac{\|p - p_i\|^2}{r^2}\right)^3
    $$
    $$
    \nabla\rho(p) = \sum_{\|p - p_i\| \le r} \begin{bmatrix} -3\left(1 - \frac{\|p - p_i\|^2}{r^2}\right)^2 \frac{2(p_x - p_{ix})}{r^2} \\ -3\left(1 - \frac{\|p - p_i\|^2}{r^2}\right)^2 \frac{2(p_y - p_{iy})}{r^2} \\ -3\left(1 - \frac{\|p - p_i\|^2}{r^2}\right)^2 \frac{2(p_z - p_{iz})}{r^2} \end{bmatrix}
    $$
    where $p = (p_x, p_y, p_z)$ is the sample location and $p_i = (p_{ix}, p_{iy}, p_{iz})$ is a particle [S1, unofficial].
6.  **Write Results:** Write $\rho(p)$ and $\nabla\rho(p)$ to the corresponding voxel in the density volume array [S1, unofficial].
7.  **Normalization:** Find the maximum density value $\rho_{max}$ across all time steps (if applicable) and normalize all density values by this maximum [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_kde_density(particles, voxel_centers, k, volume_of_cloud):
    """
    Calculate KDE density and gradient for a set of voxel centers.
    
    Args:
        particles: List of tuples (px, py, pz)
        voxel_centers: List of tuples (vx, vy, vz)
        k: Constant multiplier for radius (default 12)
        volume_of_cloud: Volume of the bounding box or initial seeding region
        
    Returns:
        densities: List of floats
        gradients: List of tuples (gx, gy, gz)
    """
    n_particles = len(particles)
    n_voxels = len(voxel_centers)
    
    # Calculate base radius and kernel radius
    # Solving for radius of a sphere with volume V/N: (4/3)*pi*r^3 = V/N => r = (3*V/(4*pi*N))^(1/3)
    # Note: The source says "dividing the volume... by the number of points, solving for the radius... then multiplying by k".
    # Standard sphere volume formula is implied by "solving for the radius of a single particle".
    base_radius = math.pow((3.0 * volume_of_cloud) / (4.0 * math.pi * n_particles), 1.0/3.0)
    r = k * base_radius
    r_sq = r * r
    
    densities = [0.0] * n_voxels
    gradients = [(0.0, 0.0, 0.0)] * n_voxels
    
    # In a real GPU implementation, this loop is parallelized per voxel.
    # Here we simulate the logic for a single voxel center p.
    # For implementation completeness, we iterate all voxels.
    
    for i in range(n_voxels):
        px, py, pz = voxel_centers[i]
        rho = 0.0
        grad_x = 0.0
        grad_y = 0.0
        grad_z = 0.0
        
        for j in range(n_particles):
            ppx, ppy, ppz = particles[j]
            
            dx = px - ppx
            dy = py - ppy
            dz = pz - ppz
            
            dist_sq = dx*dx + dy*dy + dz*dz
            
            # Check if particle is within kernel radius
            if dist_sq <= r_sq:
                # Calculate kernel term
                term = 1.0 - (dist_sq / r_sq)
                term_cubed = term * term * term
                term_sq = term * term
                
                # Density contribution
                rho += term_cubed
                
                # Gradient contribution
                # d/dx of (1 - d^2/r^2)^3 = 3*(1 - d^2/r^2)^2 * (-2*dx/r^2)
                # Source formula: -3 * (1 - ...)^2 * (2*dx)/r^2
                grad_x += -3.0 * term_sq * (2.0 * dx) / r_sq
                grad_y += -3.0 * term_sq * (2.0 * dy) / r_sq
                grad_z += -3.0 * term_sq * (2.0 * dz) / r_sq
        
        densities[i] = rho
        gradients[i] = (grad_x, grad_y, grad_z)
        
    return densities, gradients
```

**Edge Cases and Pitfalls:**
*   **Uniformity Assumption:** The method assumes the initial seeding of the point cloud is relatively uniform because it uses a uniform radius for all kernels. Non-uniform distributions may lead to inaccurate density estimates [S1, unofficial].
*   **Boundary Conditions:** The KDTree traversal must correctly handle particles near the boundaries of the point cloud to avoid missing neighbors or including invalid distances [G].
*   **Precision:** Single-precision floating-point numbers are used for the density volume, which may introduce quantization errors in very large or small scale features [S1, unofficial].

## Subtopics

### [Forward-mapping KDE with Dynamic Bandwidth](14765-forward-mapping-kde-with-dynamic-bandwidth.md)
This method computes the density field of continuous indexed points in parallel coordinates by combining forward-mapping of spatial samples to the image plane with KDE using dynamic bandwidth. It solves the problem of high-frequency noise and density distortion caused by irregular point spacing and non-linear transformations in angle-uniform coordinates. Unlike fixed-bandwidth KDE, which ablates high-density regions and thickens features, this approach preserves thin high-density structures while smoothing low-density areas, ensuring conservation of virtual "mass" through forward mapping.

### Scatter Density Visualization
A visualization method that creates a scatter plot where point colors represent local data density, using stratified sampling to manage large datasets and Gaussian KDE for density estimation [S2, unofficial]. It calculates a linear fit slope and offset, determines a sample size capped at the dataset length, and optionally uses stratified sampling to select indices. It computes Gaussian KDE on the full dataset, evaluates density at each point, sorts points by ascending density, and plots them with a density colormap, overlaying a 1:1 reference line and linear fit line [S2, unofficial]. Parameters include `n_sample` (default 10000) and `stratified` (boolean flag) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The density calculation for each voxel requires querying neighbors within radius $r$. In the worst case (uniform distribution, large $r$), this is $O(N \cdot V)$ where $N$ is the number of points and $V$ is the number of voxels. However, using a KDTree reduces the average neighbor query time to $O(\log N)$, making the total complexity approximately $O(V \log N)$ for sparse neighborhoods, or $O(V \cdot k_{avg})$ where $k_{avg}$ is the average number of neighbors within radius $r$ [S1, unofficial].
*   **Memory Complexity:** The KDTree requires $O(N)$ space. The density volume requires $O(V)$ space, where $V$ is the number of voxels. Each voxel stores 4 floats (16 bytes) [S1, unofficial].
*   **Performance:** The method is performed on the GPU using OpenCL, allowing for interactive frame rates when combined with a GPU-based raycaster (GLSL) [S1, unofficial].
*   **Strengths:** Provides insight into interior features and context for how regions relate to one another. Reduces high-frequency noise present in point-based rendering. Allows for emphasis on specific density regions via opacity multipliers [S1, unofficial].
*   **Limitations:** Lacks the ability to track changes in the point cloud over time in a single rendering; sequential volume renderings are required to observe evolution. Assumes uniform initial point cloud seeding [S1, unofficial].
*   **When to Use:** When the goal is to visualize the interior structure and density context of a particle system or point cloud, especially when occlusion or high-frequency noise in direct rendering obscures features [S1, unofficial].
*   **When Not to Use:** When tracking individual particle trajectories over time is required, or when the point cloud is highly non-uniform and the uniform radius assumption leads to significant distortion [S1, unofficial].

## Background
The method originates from the work of J. Chandler, H. Obermaier, and K. I. Joy in "Illustrative Rendering of Particle Systems" (2013) [S1, unofficial]. It was developed to address the limitations of direct point visualization (high frequencies, occlusion) and isosurface rendering (missing interior features) by providing a smooth, continuous density representation suitable for volume rendering techniques [S1, unofficial]. The approach abstracts from individual particles to emphasize the continuum of the data, using silhouette edges based on density gradients to highlight features [S1, unofficial].

## Sources
[S1] Illustrative Rendering of Particle Systems, https://doi.org/10.2312/pe.vmv.vmv13.177-185, fetched 2026-10-26 (unofficial)
[S2] lforster/hybridCT, https://github.com/lforster/hybridCT, fetched 2026-10-27 (unofficial)
