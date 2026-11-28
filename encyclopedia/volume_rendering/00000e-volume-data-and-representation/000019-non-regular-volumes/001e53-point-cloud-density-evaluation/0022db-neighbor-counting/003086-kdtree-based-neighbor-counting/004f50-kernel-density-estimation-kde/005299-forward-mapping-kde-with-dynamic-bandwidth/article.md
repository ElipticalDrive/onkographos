# Forward-mapping KDE with Dynamic Bandwidth

Also called: Density Estimation of Continuous Indexed Points, Forward-mapping KDE, KDE with dynamic bandwidth

## Overview
This method computes the density field of continuous indexed points in parallel coordinates by combining forward-mapping of spatial samples to the image plane with kernel density estimation (KDE) using dynamic bandwidth [S1]. It solves the problem of high-frequency noise and density distortion caused by irregular point spacing and non-linear transformations in angle-uniform coordinates [S1]. Unlike fixed-bandwidth KDE, which ablates high-density regions and thickens features, this approach preserves thin high-density structures while smoothing low-density areas, ensuring conservation of virtual "mass" through forward mapping [S1].

## How it works
**Inputs:**
- A set of indexed points $\eta$ in the spatial domain [S1].
- A target image plane (angle-uniform parallel coordinates) [S1].
- Parameters: maximum radius $r_{max}$, maximum accumulated density $\rho_{max}$, and number of Monte Carlo samples $N$ [S1].

**Outputs:**
- A density field $\rho(\eta)$ defined on the image plane pixels [S1].

**Data Structures:**
1. **Sample Density Image:** A 2D array where each cell accumulates the count of indexed points falling into that pixel location [S1].
2. **KD-Tree:** A 2-dimensional KD-tree built on the *unique* locations present in the Sample Density Image to accelerate nearest-neighbor queries [S1].

**Steps:**

1. **Forward Mapping:** Map point samples from the spatial domain to the image plane of parallel coordinates [S1]. This ensures that points remain points, guaranteeing conservation of virtual "mass" and handling minor deformations in the y-direction [S1].
2. **Accumulate Sample Density:** Iterate through all mapped points and increment the value in the Sample Density Image at the corresponding pixel location [S1].
3. **Build KD-Tree:** Construct a 2D KD-tree using only the unique pixel locations that have a non-zero count in the Sample Density Image [S1].
4. **Dynamic Bandwidth Calculation:** For a query point $\eta$ on the image plane, determine the bandwidth radius $r$ [S1]. The bandwidth is inversely related to the local sample density $\rho(\eta)$, meaning low density regions use a larger neighborhood and high density regions use a smaller neighborhood [S1]. The radius is controlled by a maximum radius parameter and a maximum accumulated density parameter [S1].
5. **Density Estimation via Monte Carlo Integration:** For each pixel $\eta$ in the image plane, compute the density $\rho(\eta)$ using the adapted KDE formula [S1]:
   $$ \rho(\eta) = \frac{1}{\pi r^2} \int_{\Xi} H(\rho(\hat{\eta}))\rho(\hat{\eta}) d^2 \hat{\eta} $$
   This integral is approximated using Monte Carlo integration with $N$ samples $\hat{\eta}_i$ drawn from the neighborhood $\Xi$ around $\eta$ [S1]:
   $$ \rho(\eta) \approx \frac{1}{N \pi r^2} \sum_{i=0}^{N-1} H(\rho(\hat{\eta}_i))\rho(\hat{\eta}_i) $$
   where $\Xi$ is the disk neighborhood around $\eta$ with radius $r$, $H$ is the kernel function, and $\rho(\hat{\eta}_i)$ is the accumulated sample density at the sample location $\hat{\eta}_i$ [S1].
6. **Kernel Selection:** Use the Epanechnikov (parabolic) kernel for $H$ [S1]. It is chosen for its optimality with respect to mean square error, smooth fall-off, and bounded support [S1]. The Epanechnikov kernel is defined as:
   $$ H(u) = \begin{cases} \frac{3}{4}(1 - u^2) & \text{if } |u| \le 1 \\ 0 & \text{otherwise} \end{cases} $$
   where $u$ is the normalized distance from the center of the kernel [G].
7. **Neighborhood Control:** The neighborhood size is controlled using a maximum radius parameter and a maximum accumulated density parameter to prevent excessive search ranges in sparse regions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def epanechnikov_kernel(u):
    """Epanechnikov kernel function [G]."""
    return 0.75 * (1.0 - u * u) if abs(u) <= 1.0 else 0.0

def map_point_to_image(point):
    """Forward mapping from spatial domain to image plane coordinates [G]."""
    return point[0], point[1]

def build_sample_density_image(points, width, height):
    """Accumulates indexed point samples into a 2D density image [S1]."""
    density_image = [[0.0] * width for _ in range(height)]
    unique_locs = []
    for p in points:
        px, py = map_point_to_image(p)
        ix, iy = int(px), int(py)
        if 0 <= ix < width and 0 <= iy < height:
            density_image[iy][ix] += 1.0
    for y in range(height):
        for x in range(width):
            if density_image[y][x] > 0:
                unique_locs.append((x, y, density_image[y][x]))
    return density_image, unique_locs

class KDTree:
    """Minimal 2D KD-tree for nearest-neighbor queries [G]."""
    def __init__(self, points):
        self.points = points
        self.root = self._build(points, 0)

    def _build(self, pts, depth):
        if not pts: return None
        axis = depth % 2
        pts = sorted(pts, key=lambda p: p[axis])
        mid = len(pts) // 2
        return {'point': pts[mid], 'left': self._build(pts[:mid], depth+1), 'right': self._build(pts[mid+1:], depth+1)}

    def nearest(self, target, best=None, node=None, depth=0):
        if node is None: node = self.root
        if best is None: best = [float('inf'), node['point']]
        axis = depth % 2
        dist = math.hypot(target[0] - node['point'][0], target[1] - node['point'][1])
        if dist < best[0]: best = [dist, node['point']]
        delta = target[axis] - node['point'][axis]
        child = node['left'] if delta < 0 else node['right']
        if child: best = self.nearest(target, best, child, depth+1)
        if abs(delta) < best[0]:
            other = node['right'] if delta < 0 else node['left']
            if other: best = self.nearest(target, best, other, depth+1)
        return best[1]

def get_dynamic_bandwidth(query_density, max_radius, max_density):
    """Calculates radius r based on inverse relationship with density [S1]."""
    if max_density <= 0: return max_radius
    r = max_radius * (query_density / max_density)
    return max(0.0, min(r, max_radius))

def monte_carlo_kde(query_point, kd_tree, unique_locs, N, max_radius, max_density):
    """Estimates density at query_point using Monte Carlo integration [S1]."""
    query_density = 0.0
    for loc in unique_locs:
        if loc[0] == query_point[0] and loc[1] == query_point[1]:
            query_density = loc[2]
            break
    r = get_dynamic_bandwidth(query_density, max_radius, max_density)
    sum_val = 0.0
    for _ in range(N):
        theta = random.uniform(0, 2 * math.pi)
        radius = math.sqrt(random.uniform(0, 1)) * r
        sx = query_point[0] + radius * math.cos(theta)
        sy = query_point[1] + radius * math.sin(theta)
        sample_loc = kd_tree.nearest((sx, sy))
        sample_density = sample_loc[2]
        dist = math.hypot(sx - query_point[0], sy - query_point[1])
        u = dist / r
        sum_val += epanechnikov_kernel(u) * sample_density
    return sum_val / (N * math.pi * r * r)

def render_density_field(points, width, height, N, max_radius, max_density):
    """Main rendering pipeline [S1]."""
    density_image, unique_locs = build_sample_density_image(points, width, height)
    kd_tree = KDTree(unique_locs)
    result = [[0.0] * width for _ in range(height)]
    for y in range(height):
        for x in range(width):
            result[y][x] = monte_carlo_kde((x, y), kd_tree, unique_locs, N, max_radius, max_density)
    return result
```

**Edge Cases and Pitfalls:**
- **Fixed Bandwidth Issues:** Using a fixed bandwidth ablates high-density regions and overly smooths results, thickening high-density features [S1]. Dynamic bandwidth is required to preserve thin structures [S1].
- **Deformation Handling:** Main deformations in angle-uniform parallel coordinates occur in the x-direction, which does not affect density counting along y [S1]. Minor y-direction deformations are handled by the forward mapping [S1].
- **Precision:** Monte Carlo integration introduces noise [G]. Increasing $N$ reduces noise but increases computation time [G].
- **Boundary Conditions:** Points near the image boundary may have neighborhoods extending outside the image [G]. Samples outside the valid domain should be discarded or clamped to the boundary [G].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** The method separates location information and sample density to accelerate computation [S1]. It uses a kd-tree on unique locations to significantly reduce the amount of samples processed [S1].
- **Strengths:** Preserves high-density regions, smooths low-density regions, conserves virtual mass, and handles non-linear transformations [S1].
- **Limitations:** Requires tuning of $N$, $r_{max}$, and $\rho_{max}$ [S1]. Monte Carlo noise may require higher $N$ or post-processing [G].

## Background
This method was introduced in the context of multivariate volume visualization using parallel coordinates [S1]. It addresses the limitations of traditional density models that count lines along the y-direction, which are susceptible to distortions from non-linear transformations in angle-uniform coordinates [S1]. The approach builds on forward-mapping techniques [S1] and adapts kernel density estimation [43–46] with the Epanechnikov kernel [47, 48] for optimal mean square error [S1].

## Sources
[S1] Continuous Indexed Points for Multivariate Volume Visualization, https://doi.org/10.26599/cvm.2025.9450496, fetched 2026-10-26
