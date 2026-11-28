# Spherical Voxel Grid Setup

Also called: SVR grid setup, Spherical voxel grid, SphericalVoxelGrid Initialization

## Overview
Spherical Voxel Grid Setup initializes the geometric data structures required for efficient ray casting traversal in a spherical coordinate system [S1, unofficial]. It builds on [Voxel Space Setup](3520-voxel-space-setup.md) by defining a spherical bounding volume and precomputing traversal constants [S1, unofficial].

## How it works
**Inputs:**
- `min_bound`, `max_bound`: `SphereBound` structs defining the lower and upper limits for radial ($r$), polar ($\theta$), and azimuthal ($\phi$) coordinates [S1, unofficial].
- `num_radial_sections`, `num_polar_sections`, `num_azimuthal_sections`: Integer counts defining the number of voxels along the radial, polar, and azimuthal axes, respectively [S1, unofficial].
- `sphere_center`: 3D Cartesian coordinates $(x, y, z)$ of the sphere's center [S1, unofficial].

**Outputs:**
- A `SphericalVoxelGrid` object containing precomputed deltas, trigonometric lookup tables, line segment arrays, and bound vectors [S1, unofficial].

**Data Structures:**
1. `SphereBound`:
   - `radial`: `double`
   - `polar`: `double`
   - `azimuthal`: `double`
2. `LineSegment`:
   - `P1`: `double`
   - `P2`: `double`
3. `TrigonometricValues`:
   - `cosine`: `double`
   - `sine`: `double`
4. `SphericalVoxelGrid` (internal fields):
   - `num_radial_sections_`, `num_polar_sections_`, `num_azimuthal_sections_`: `std::size_t`
   - `sphere_center_`: `Vec3` (3D Cartesian)
   - `sphere_max_radius_`: `double`
   - `sphere_max_diameter_`: `double`
   - `delta_radius_`, `delta_theta_`, `delta_phi_`: `double`
   - `delta_radii_sq_`: `std::vector<double>` of size $N_r + 1$
   - `polar_trig_values_`: `std::vector<TrigonometricValues>` of size $N_p + 1$
   - `azimuthal_trig_values_`: `std::vector<TrigonometricValues>` of size $N_a + 1$
   - `P_max_polar_`: `std::vector<LineSegment>` of size $N_p + 1$
   - `P_max_azimuthal_`: `std::vector<LineSegment>` of size $N_a + 1$
   - `center_to_polar_bound_vectors_`: `std::vector<Vec3>` of size $N_p + 1$
   - `center_to_azimuthal_bound_vectors_`: `std::vector<Vec3>` of size $N_a + 1$ [S1, unofficial]

**Steps:**

1. **Compute Deltas:**
   Calculate the step size for each spherical dimension using the bounds and section counts [S1, unofficial]:
   $$ \Delta r = \frac{r_{max} - r_{min}}{N_r} $$
   $$ \Delta \theta = \frac{\theta_{max} - \theta_{min}}{N_p} $$
   $$ \Delta \phi = \frac{\phi_{max} - \phi_{min}}{N_a} $$
   Where $r_{max}, r_{min}$ correspond to `max_bound.radial` and `min_bound.radial`, and $N_r, N_p, N_a$ are the respective section counts [S1, unofficial].

2. **Initialize Delta Radii Squared:**
   Create `delta_radii_sq_` of size $N_r + 1$ [S1, unofficial].
   - Set $R_{start} = r_{max}$ (the source uses `max_bound.radial` as the absolute starting radius for the squared array) [S1, unofficial].
   - Initialize `current_r` = $R_{start}$ [S1, unofficial].
   - For $i$ from 0 to $N_r$:
     - `delta_radii_sq_[i]` = `current_r`$^2$
     - `current_r` -= $\Delta r$ [S1, unofficial].

3. **Initialize Trigonometric Values:**
   Create `polar_trig_values_` (size $N_p + 1$) and `azimuthal_trig_values_` (size $N_a + 1$) [S1, unofficial].
   - For polar: Start with $\theta = \theta_{min}$ [S1, unofficial]. For $i$ from 0 to $N_p$:
     - `polar_trig_values_[i].cosine` = $\cos(\theta)$
     - `polar_trig_values_[i].sine` = $\sin(\theta)$
     - $\theta$ += $\Delta \theta$ [S1, unofficial].
   - For azimuthal: Start with $\phi = \phi_{min}$ [S1, unofficial]. For $i$ from 0 to $N_a$:
     - `azimuthal_trig_values_[i].cosine` = $\cos(\phi)$
     - `azimuthal_trig_values_[i].sine` = $\sin(\phi)$
     - $\phi$ += $\Delta \phi$ [S1, unofficial].

4. **Initialize Max Radius Line Segments:**
   Create `P_max_polar_` (size $N_p + 1$) and `P_max_azimuthal_` (size $N_a + 1$) [S1, unofficial].
   - These arrays store intersection points between voxel boundary lines and the radial voxel [S1, unofficial].
   - Compute Cartesian coordinates for the angular boundaries using the precomputed trigonometric values and `sphere_max_radius_` [S1, unofficial].
   - Store the resulting boundary points as `LineSegment` entries [S1, unofficial].

5. **Compute Bound Vectors:**
   Calculate `center_to_polar_bound_vectors_` and `center_to_azimuthal_bound_vectors_` [S1, unofficial].
   - For each index $i$:
     - `center_to_polar_bound_vectors_[i]` = `sphere_center` - `P_max_polar_[i]`
     - `center_to_azimuthal_bound_vectors_[i]` = `sphere_center` - `P_max_azimuthal_[i]` [S1, unofficial].
   - These vectors represent the direction from the sphere center to the voxel boundary points [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __sub__(self, other):
        return Vec3(self.x - other.x, self.y - other.y, self.z - other.z)

class LineSegment:
    def __init__(self, P1, P2):
        self.P1 = P1
        self.P2 = P2

class TrigonometricValues:
    def __init__(self, cosine, sine):
        self.cosine = cosine
        self.sine = sine

class SphericalVoxelGrid:
    def __init__(self, num_radial, num_polar, num_azimuthal, sphere_center, max_radius, diameter, dr, dt, dp, dr_sq, trig_p, trig_a, seg_p, seg_a, vec_p, vec_a):
        self.num_radial_sections_ = num_radial
        self.num_polar_sections_ = num_polar
        self.num_azimuthal_sections_ = num_azimuthal
        self.sphere_center_ = sphere_center
        self.sphere_max_radius_ = max_radius
        self.sphere_max_diameter_ = diameter
        self.delta_radius_ = dr
        self.delta_theta_ = dt
        self.delta_phi_ = dp
        self.delta_radii_sq_ = dr_sq
        self.polar_trig_values_ = trig_p
        self.azimuthal_trig_values_ = trig_a
        self.P_max_polar_ = seg_p
        self.P_max_azimuthal_ = seg_a
        self.center_to_polar_bound_vectors_ = vec_p
        self.center_to_azimuthal_bound_vectors_ = vec_a

def setup_spherical_voxel_grid(min_bound, max_bound, num_radial, num_polar, num_azimuthal, sphere_center):
    # 1. Compute Deltas
    delta_radius = (max_bound.radial - min_bound.radial) / num_radial
    delta_theta = (max_bound.polar - min_bound.polar) / num_polar
    delta_phi = (max_bound.azimuthal - min_bound.azimuthal) / num_azimuthal
    
    # 2. Initialize Delta Radii Squared
    max_radius = max_bound.radial
    delta_radii_sq = []
    current_r = max_radius
    for i in range(num_radial + 1):
        delta_radii_sq.append(current_r * current_r)
        current_r -= delta_radius
        
    # 3. Initialize Trigonometric Values
    polar_trig = []
    theta = min_bound.polar
    for i in range(num_polar + 1):
        polar_trig.append(TrigonometricValues(math.cos(theta), math.sin(theta)))
        theta += delta_theta
        
    azimuthal_trig = []
    phi = min_bound.azimuthal
    for i in range(num_azimuthal + 1):
        azimuthal_trig.append(TrigonometricValues(math.cos(phi), math.sin(phi)))
        phi += delta_phi
        
    # 4. Initialize Line Segments (P_max)
    # Standard spherical-to-Cartesian conversion [G]
    def spherical_to_cartesian(r, theta, phi):
        x = r * math.sin(theta) * math.cos(phi)
        y = r * math.sin(theta) * math.sin(phi)
        z = r * math.cos(theta)
        return Vec3(x, y, z)

    P_max_polar = []
    for i in range(num_polar + 1):
        theta = min_bound.polar + i * delta_theta
        # Polar boundaries are computed at fixed azimuthal midpoints for the slice [G]
        phi_mid = (min_bound.azimuthal + max_bound.azimuthal) / 2.0
        p1 = spherical_to_cartesian(max_radius, theta, phi_mid)
        p2 = spherical_to_cartesian(max_radius, theta, phi_mid + delta_phi)
        P_max_polar.append(LineSegment(p1, p2))
        
    P_max_azimuthal = []
    for i in range(num_azimuthal + 1):
        phi = min_bound.azimuthal + i * delta_phi
        # Azimuthal boundaries are computed at fixed polar midpoints for the slice [G]
        theta_mid = (min_bound.polar + max_bound.polar) / 2.0
        p1 = spherical_to_cartesian(max_radius, theta_mid, phi)
        p2 = spherical_to_cartesian(max_radius, theta_mid, phi + delta_phi)
        P_max_azimuthal.append(LineSegment(p1, p2))
        
    # 5. Compute Bound Vectors
    center_to_polar = []
    for i in range(num_polar + 1):
        center_to_polar.append(sphere_center - P_max_polar[i].P1)
        
    center_to_azimuthal = []
    for i in range(num_azimuthal + 1):
        center_to_azimuthal.append(sphere_center - P_max_azimuthal[i].P1)
        
    return SphericalVoxelGrid(
        num_radial_sections=num_radial,
        num_polar_sections=num_polar,
        num_azimuthal_sections=num_azimuthal,
        sphere_center=sphere_center,
        max_radius=max_radius,
        diameter=max_radius * 2.0,
        dr=delta_radius,
        dt=delta_theta,
        dp=delta_phi,
        dr_sq=delta_radii_sq,
        trig_p=polar_trig,
        trig_a=azimuthal_trig,
        seg_p=P_max_polar,
        seg_a=P_max_azimuthal,
        vec_p=center_to_polar,
        vec_a=center_to_azimuthal
    )
```

**Parameters:**
- `num_radial_sections`, `num_polar_sections`, `num_azimuthal_sections`: Integer counts defining the resolution along each axis [S1, unofficial].
- `min_bound`, `max_bound`: `SphereBound` structs defining coordinate limits [S1, unofficial].
- `sphere_center`: 3D Cartesian coordinates of the sphere's center [S1, unofficial].

**Edge Cases and Pitfalls:**
- The grid coordinate system does not align with standard spherical coordinates where the polar angle is $[0, \pi]$ [S1, unofficial]. Both polar and azimuthal angles are represented within $[0, 2\pi]$ [S1, unofficial]. A TODO in the source notes a potential future update of the polar grid from $[0, 2\pi]$ to $[0, \pi]$ [S1, unofficial].
- Precision issues may arise in the trigonometric precomputation if not handled carefully, though no specific mitigation is cited [S1, unofficial].

## Subtopics

### [Balanced Spherical Grid](3942-balanced-spherical-grid.md)
A neural volume representation for egocentric omnidirectional view synthesis that replaces Cartesian grids to solve the inefficiency of uniform resolution in large-scale unbounded scenes. It avoids polar singularities by combining two offset quasi-uniform spherical grids (Yin and Yang) and handles unbounded depth by partitioning the radial dimension exponentially and placing an environment map at infinity. Unlike Cartesian feature grids which suffer from non-uniform ray-grid hits and wasted memory on distant objects, this method aligns with egocentric rays and enables factorization for performance enhancement.

## Operating characteristics
- **Time Complexity:** Initialization is $O(N_r + N_p + N_a)$ due to the linear precomputation of deltas, trigonometric values, and line segments [S1, unofficial].
- **Memory Complexity:** $O(N_r + N_p + N_a)$ to store the precomputed arrays [S1, unofficial].
- **Strengths:** Aligns with egocentric rays, reducing wasted memory on distant objects compared to Cartesian grids [S1, unofficial]. Enables factorization for performance enhancement [S1, unofficial].
- **Limitations:** Non-standard angular bounds ($[0, 2\pi]$ for polar) may require careful handling in downstream algorithms [S1, unofficial].
- **Usage:** Instantiated and passed to `walkSphericalVolume` for ray casting traversal [S1, unofficial].

## Background
The Spherical Voxel Grid Setup is part of the Spherical Volume Rendering (SVR) algorithm, designed for egocentric omnidirectional view synthesis [S1, unofficial]. The method was developed as part of the `spherical-volume-rendering` project [S1, unofficial].

## Sources
[S1] spherical-volume-rendering/svr-algorithm, https://github.com/spherical-volume-rendering/svr-algorithm, fetched 2026-10-24 (unofficial)
