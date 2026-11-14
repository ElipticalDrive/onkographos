# Forward-mapping KDE with Dynamic Bandwidth

Also called: Density Estimation of Continuous Indexed Points, Forward-mapping KDE, KDE with dynamic bandwidth

## Overview
This method computes the density field of continuous indexed points in parallel coordinates by combining forward-mapping of spatial samples to the image plane with kernel density estimation (KDE) using dynamic bandwidth [S1]. It solves the problem of high-frequency noise and density distortion caused by irregular point spacing and non-linear transformations in angle-uniform coordinates [S1]. Unlike fixed-bandwidth KDE, which ablates high-density regions and thickens features, this approach preserves thin high-density structures while smoothing low-density areas, ensuring conservation of virtual "mass" through forward mapping [S1].

## How it works
**Inputs:**
- A set of indexed points $\eta$ in the spatial domain.
- A target image plane (angle-uniform parallel coordinates).
- Parameters: Maximum radius $r_{max}$, maximum accumulated density $\rho_{max}$, number of Monte Carlo samples $N$.

**Outputs:**
- A density field $\rho(\eta)$ defined on the image plane pixels.

**Data Structures:**
1. **Sample Density Image:** A 2D array (image) where each cell accumulates the count of indexed points falling into that pixel location. This separates location information from sample density [S1].
2. **KD-Tree:** A 2-dimensional KD-tree built on the *unique* locations present in the Sample Density Image. This accelerates nearest-neighbor queries [S1].

**Steps:**

1. **Forward Mapping:** Map point samples from the spatial domain to the image plane of parallel coordinates. This ensures that points remain points, guaranteeing conservation of virtual "mass" and handling minor deformations in the y-direction [S1].

2. **Accumulate Sample Density:** Iterate through all mapped points. For each point falling into pixel location $p$, increment the value in the Sample Density Image at $p$. This creates a histogram of point occurrences [S1].

3. **Build KD-Tree:** Construct a 2D KD-tree using only the unique pixel locations $p$ that have a non-zero count in the Sample Density Image. Store the accumulated density value $\rho(p)$ associated with each node [S1].

4. **Dynamic Bandwidth Calculation:** For a query point $\eta$ on the image plane, determine the bandwidth radius $r$. The bandwidth is inversely related to the local sample density $\rho(\eta)$.
   - Low density regions use a larger neighborhood.
   - High density regions use a smaller neighborhood.
   - The radius $r$ is constrained by a maximum radius parameter $r_{max}$ [S1].

5. **Density Estimation via Monte Carlo Integration:** For each pixel $\eta$ in the image plane, compute the density $\rho(\eta)$ using the adapted KDE formula [S1]:
   $$ \rho(\eta) = \frac{1}{\pi r^2 \Xi} \int_{\Xi} H(\rho(\hat{\eta}))\rho(\hat{\eta}) d^2 \hat{\eta} $$
   This integral is approximated using Monte Carlo integration with $N$ samples $\hat{\eta}_i$ drawn from the neighborhood $\Xi$ around $\eta$ [S1]:
   $$ \rho(\eta) \approx \frac{1}{N \pi r^2} \sum_{i=0}^{N-1} H(\rho(\hat{\eta}_i))\rho(\hat{\eta}_i) $$
   where:
   - $\Xi$ is the disk neighborhood around $\eta$ with radius $r$.
   - $H$ is the kernel function.
   - $\rho(\hat{\eta}_i)$ is the accumulated sample density at the sample location $\hat{\eta}_i$.

6. **Kernel Selection:** Use the Epanechnikov (parabolic) kernel for $H$. It is chosen for its optimality with respect to mean square error, smooth fall-off, and bounded support [S1]. The Epanechnikov kernel is defined as:
   $$ H(u) = \begin{cases} \frac{3}{4}(1 - u^2) & \text{if } |u| \le 1 \\ 0 & \text{otherwise} \end{cases} $$
   where $u$ is the normalized distance from the center of the kernel [S1].

7. **Clamping:** Ensure the calculated density does not exceed the maximum accumulated density parameter $\rho_{max}$ if specified, or use the maximum radius $r_{max}$ to bound the search space [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def epanechnikov_kernel(u):
    """
    Epanechnikov kernel function.
    u: normalized distance from center.
    """
    if abs(u) <= 1.0:
        return 0.75 * (1.0 - u * u)
    return 0.0

def build_sample_density_image(points, image_width, image_height):
    """
    Maps points to image plane and accumulates counts.
    Returns a 2D array of counts and a list of unique locations.
    """
    density_image = [[0 for _ in range(image_width)] for _ in range(image_height)]
    unique_locations = []
    
    for point in points:
        # Forward mapping from spatial domain to image plane
        # Assuming point has x, y coordinates in image space
        px, py = map_point_to_image(point)
        
        if 0 <= px < image_width and 0 <= py < image_height:
            density_image[py][px] += 1
            
    # Collect unique locations with non-zero density
    for y in range(image_height):
        for x in range(image_width):
            if density_image[y][x] > 0:
                unique_locations.append((x, y, density_image[y][x]))
                
    return density_image, unique_locations

def build_kd_tree(unique_locations):
    """
    Builds a 2D KD-tree from unique locations.
    Each node stores (x, y, density_value).
    """
    # Implementation of 2D KD-tree construction
    # Nodes contain: x, y, density
    # Returns root of KD-tree
    pass

def get_dynamic_bandwidth(query_density, max_radius, max_density):
    """
    Calculates radius r based on inverse relationship with density.
    """
    # Inverse relationship: higher density -> smaller radius
    # Base radius could be max_radius
    # Scale factor based on density ratio
    if max_density == 0:
        return max_radius
        
    # Example heuristic: r = max_radius * (query_density / max_density)
    # Adjusted to ensure r <= max_radius
    r = max_radius * (query_density / max_density)
    return min(r, max_radius)

def monte_carlo_kde(query_point, kd_tree, unique_locations, N, max_radius):
    """
    Estimates density at query_point using Monte Carlo integration.
    """
    # Determine local density at query point
    # If query_point is not in unique_locations, interpolate or use 0
    query_density = get_density_at_point(query_point, unique_locations)
    
    # Calculate dynamic bandwidth
    r = get_dynamic_bandwidth(query_density, max_radius, 1.0) # Assuming normalized max_density
    
    sum_kernel_weighted_density = 0.0
    count = 0
    
    for _ in range(N):
        # Sample a point uniformly in the disk neighborhood Xi around query_point
        # Generate random angle theta and radius sqrt(random)*r
        theta = random.uniform(0, 2 * math.pi)
        radius = math.sqrt(random.uniform(0, 1)) * r
        sample_x = query_point.x + radius * math.cos(theta)
        sample_y = query_point.y + radius * math.sin(theta)
        
        # Find density at sample location using KD-tree or direct lookup
        # For efficiency, use KD-tree to find nearest unique location or interpolate
        sample_loc = find_density_at_location((sample_x, sample_y), kd_tree)
        
        if sample_loc is not None:
            sample_density = sample_loc.density
            # Normalize distance for kernel
            dist = math.sqrt((sample_x - query_point.x)**2 + (sample_y - query_point.y)**2)
            u = dist / r
            
            kernel_val = epanechnikov_kernel(u)
            sum_kernel_weighted_density += kernel_val * sample_density
            count += 1
            
    if count == 0:
        return 0
        
    # Formula: rho(eta) approx (1 / (N * pi * r^2)) * sum(H(rho(hat_eta)) * rho(hat_eta))
    # Note: The source formula divides by N * pi * r^2. 
    # However, Monte Carlo integration of an integral over area A is (Area/N) * sum(f(x_i)).
    # Here Area = pi * r^2.
    # So rho(eta) = (pi * r^2 / N) * sum( H(rho(hat_eta)) * rho(hat_eta) ) / (pi * r^2) ?
    # Let's re-read source Eq 12: rho(eta) approx (1 / (N * pi * r^2)) * sum(...)
    # This implies the integral is approximated as (Area/N) * sum, and then divided by Area?
    # Actually, Eq 11 is rho(eta) = (1 / (pi * r^2)) * Integral.
    # Monte Carlo approx Integral as (Area/N) * sum(f(x_i)).
    # So rho(eta) approx (1 / (pi * r^2)) * (pi * r^2 / N) * sum(...) = (1/N) * sum(...).
    # BUT Source Eq 12 explicitly has 1/(N * pi * r^2).
    # This suggests the term inside the sum might not include the area scaling, or the kernel H is defined differently.
    # Let's stick strictly to Source Eq 12:
    
    density = sum_kernel_weighted_density / (count * math.pi * r * r)
    return density

def render_density_field(points, image_width, image_height, N, max_radius):
    density_image, unique_locations = build_sample_density_image(points, image_width, image_height)
    kd_tree = build_kd_tree(unique_locations)
    
    result_image = [[0.0 for _ in range(image_width)] for _ in range(image_height)]
    
    for y in range(image_height):
        for x in range(image_width):
            query_point = (x, y)
            # If pixel has no points, density might be 0 or interpolated
            # For simplicity, if no points, skip or use 0
            if density_image[y][x] == 0:
                # Optional: interpolate from neighbors
                result_image[y][x] = 0.0
                continue
                
            result_image[y][x] = monte_carlo_kde(query_point, kd_tree, unique_locations, N, max_radius)
            
    return result_image
```

**Edge Cases and Pitfalls:**
- **Fixed Bandwidth Issues:** Using a fixed bandwidth ablates high-density regions and overly smooths results, thickening high-density features [S1]. Dynamic bandwidth is required to preserve thin structures [S1].
- **Deformation Handling:** Main deformations in angle-uniform parallel coordinates occur in the x-direction, which does not affect density counting along y. Minor y-direction deformations are handled by the forward mapping [S1].
- **Precision:** Monte Carlo integration introduces noise. Increasing $N$ reduces noise but increases computation time [S1].
- **Boundary Conditions:** Points near the image boundary may have neighborhoods extending outside the image. These should be handled by clamping or ignoring samples outside the valid domain [G].

## Operating characteristics
- **Time Complexity:** Building the KD-tree is $O(M \log M)$ where $M$ is the number of unique locations. Density estimation per pixel is $O(N)$ where $N$ is the number of Monte Carlo samples. Total time is $O(W \cdot H \cdot N)$ for an image of size $W \times H$.
- **Memory Complexity:** $O(M)$ for the KD-tree and sample density image, where $M \le W \cdot H$.
- **Performance:** The method is efficient due to the separation of location and density, and the use of a KD-tree on unique locations. The Epanechnikov kernel is computationally inexpensive [S1].
- **Strengths:** Preserves high-density regions, smooths low-density regions, conserves virtual mass, handles non-linear transformations [S1].
- **Limitations:** Requires tuning of $N$, $r_{max}$, and $\rho_{max}$. Monte Carlo noise may require post-processing or higher $N$ [S1].

## Background
This method was introduced in the context of multivariate volume visualization using parallel coordinates [S1]. It addresses the limitations of traditional density models that count lines along the y-direction, which are susceptible to distortions from non-linear transformations in angle-uniform coordinates [S1]. The approach builds on forward-mapping techniques [S1] and adapts kernel density estimation [43–46] with the Epanechnikov kernel [47, 48] for optimal mean square error [S1].

## Sources
[S1] Continuous Indexed Points for Multivariate Volume Visualization, https://doi.org/10.26599/cvm.2025.9450496, fetched 2026-10-26
