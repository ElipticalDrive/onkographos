# Spherical Volume Ray Walking

Also called: Amanatides and Woo extension to spherical grids, Fast Voxel Traversal Over Spherical Grids, Spherical Volume Rendering (SVR) Algorithm, walkSphericalVolume

## Overview
Spherical Volume Ray Walking is a fast voxel traversal algorithm for rendering volumetric data defined on a spherical coordinate grid [S1, unofficial]. It extends the cartesian fast voxel traversal method by Amanatides and Woo to non-cartesian grids, iterating through radial, polar, and azimuthal voxel boundaries to determine the sequence of voxels intersected by a ray [S1, unofficial]. Unlike [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md) which assumes axis-aligned bounding boxes, this method handles the curved boundaries of spherical sectors, using line-sphere intersection for radial steps and 2D segment intersection for angular boundaries [S1, unofficial].

## How it works
### Inputs and Outputs
- **Inputs:** `ray` (origin $O$ as bound vector, direction $D$ as unit free vector), `grid` (spherical voxel grid with parameters `sphereCenter`, `numRadialSections`, `numPolarSections`, `numAzimuthalSections`, `minBound`, `maxBound`, `deltaRadiiSquared(r)`, `deltaPhi()`, `deltaTheta()`, `sphereMaxBoundAzi()`, `sphereMinBoundAzi()`, `sphereMaxBoundPolar()`, `sphereMinBoundPolar()`, `sphereMaxRadius()`, `pMaxPolar(i)`, `pMaxAzimuthal(i)`, `centerToPolarBound(i)`, `centerToAzimuthalBound(i)`), and time range `t_begin`, `t_end` [S1, unofficial].
- **Outputs:** List of `Voxel` structs containing `{radial, polar, azimuthal}` indices and optional `exit_t` [S1, unofficial].

### Data Structures
- `Voxel`: `{radial: int, polar: int, azimuthal: int, exit_t: float}` [S1, unofficial].
- `HitParameters`: `{tMax: float, tStep: int}` where `tStep` is -1, 0, or 1 [S1, unofficial].
- `RaySegment`: `{P1: BoundVec3, P2: BoundVec3, NZDI: int}` [S1, unofficial].
- `LineSegment`: `{P1: float, P2: float}` [S1, unofficial].
- `BoundVec3`, `FreeVec3`, `UnitVec3`: 3D vector types for bound, free, and normalized coordinates respectively [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def walk_spherical_volume(ray, grid, t_begin, t_end):
    DOUBLE_MAX = float('inf')
    PERTURBED_T = 0.1
    
    if t_end <= 0.0:
        return []

    rsv = grid.sphere_center() - ray.point_at_parameter(0.0)
    sed_from_center = rsv.squared_length()
    
    radial_entrance_voxel = 0
    while sed_from_center < grid.delta_radii_squared(radial_entrance_voxel):
        radial_entrance_voxel += 1
        
    ray_origin_is_outside_grid = (radial_entrance_voxel == 0)
    vector_index = radial_entrance_voxel - (1 if not ray_origin_is_outside_grid else 0)
    entry_radius_squared = grid.delta_radii_squared(vector_index)
    entry_radius = grid.delta_radius() * (grid.num_radial_sections() - vector_index)
    
    rsvd = rsv.dot(rsv)
    v = rsv.dot(ray.direction())
    rsvd_minus_v_squared = rsvd - v * v
    
    if entry_radius_squared <= rsvd_minus_v_squared:
        return []
        
    d = math.sqrt(entry_radius_squared - rsvd_minus_v_squared)
    t_ray_exit = ray.time_of_intersection_at(v + d)
    if t_ray_exit < 0.0:
        return []
    t_ray_entrance = ray.time_of_intersection_at(v - d)
    
    current_radial_voxel = radial_entrance_voxel + (1 if ray_origin_is_outside_grid else 0)
    
    P_polar = [None] * (grid.num_polar_sections() + 1)
    P_azimuthal = [None] * (grid.num_azimuthal_sections() + 1)
    # Sources do not provide the implementation for initialize_voxel_boundary_segments.
    
    if ray_origin_is_outside_grid:
        ray_sphere = grid.sphere_center() - ray.point_at_parameter(t_ray_entrance)
    elif sed_from_center == 0.0:
        ray_sphere = rsv - ray.direction()
    else:
        ray_sphere = rsv
        
    current_polar_voxel = initialize_angular_voxel_id(grid, grid.num_polar_sections(), ray_sphere, P_polar, ray_sphere.y, grid.sphere_center.y, entry_radius)
    if current_polar_voxel >= grid.num_polar_sections():
        return []
        
    current_azimuthal_voxel = initialize_angular_voxel_id(grid, grid.num_azimuthal_sections(), ray_sphere, P_azimuthal, ray_sphere.z, grid.sphere_center.z, entry_radius)
    if current_azimuthal_voxel >= grid.num_azimuthal_sections():
        return []
        
    voxels = []
    voxels.append(Voxel(current_radial_voxel, current_polar_voxel, current_azimuthal_voxel))
    
    t = t_ray_entrance * (1 if ray_origin_is_outside_grid else 0)
    unitized_ray_time = t_end * grid.sphere_max_diameter() + t_ray_entrance * (1 if ray_origin_is_outside_grid else 0)
    
    if ray_origin_is_outside_grid:
        max_t = min(t_ray_exit, unitized_ray_time)
    else:
        max_t = unitized_ray_time
        
    collinear_times = [0.0, ray.time_of_intersection_at(grid.sphere_center())]
    ray_segment = RaySegment(max_t, ray)
    radial_step_has_transitioned = False
    
    while True:
        radial = radial_hit(ray, grid, radial_step_has_transitioned, current_radial_voxel, v, rsvd_minus_v_squared, t, max_t)
        ray_segment.update_at_time(t, ray)
        polar = polar_hit(ray, grid, ray_segment, collinear_times, current_polar_voxel, t, max_t)
        azimuthal = azimuthal_hit(ray, grid, ray_segment, collinear_times, current_azimuthal_voxel, t, max_t)
        
        if current_radial_voxel + radial.t_step == 0 or \
           (radial.t_max == DOUBLE_MAX and polar.t_max == DOUBLE_MAX and azimuthal.t_max == DOUBLE_MAX):
            voxels[-1].exit_t = t_ray_exit
            return voxels
            
        voxel_intersection = minimum_intersection(radial, polar, azimuthal)
        
        if voxel_intersection == VoxelIntersectionType.Radial:
            t = radial.t_max
            current_radial_voxel += radial.t_step
        elif voxel_intersection == VoxelIntersectionType.Polar:
            t = polar.t_max
            if not in_bounds_polar(grid, polar.t_step, current_polar_voxel):
                voxels[-1].exit_t = t_ray_exit
                return voxels
            else:
                current_polar_voxel = (current_polar_voxel + polar.t_step) % grid.num_polar_sections()
        elif voxel_intersection == VoxelIntersectionType.Azimuthal:
            if not in_bounds_azimuthal(grid, azimuthal.t_step, current_azimuthal_voxel):
                voxels[-1].exit_t = t_ray_exit
                return voxels
            else:
                t = azimuthal.t_max
                current_azimuthal_voxel = (current_azimuthal_voxel + azimuthal.t_step) % grid.num_azimuthal_sections()
        elif voxel_intersection == VoxelIntersectionType.RadialPolar:
            t = radial.t_max
            if not in_bounds_polar(grid, polar.t_step, current_polar_voxel):
                voxels[-1].exit_t = t_ray_exit
                return voxels
            else:
                current_radial_voxel += radial.t_step
                current_polar_voxel = (current_polar_voxel + polar.t_step) % grid.num_polar_sections()
        elif voxel_intersection == VoxelIntersectionType.RadialAzimuthal:
            t = radial.t_max
            if not in_bounds_azimuthal(grid, azimuthal.t_step, current_azimuthal_voxel):
                voxels[-1].exit_t = t_ray_exit
                return voxels
            else:
                current_radial_voxel += radial.t_step
                current_azimuthal_voxel = (current_azimuthal_voxel + azimuthal.t_step) % grid.num_azimuthal_sections()
        elif voxel_intersection == VoxelIntersectionType.PolarAzimuthal:
            t = polar.t_max
            if not in_bounds_azimuthal(grid, azimuthal.t_step, current_azimuthal_voxel) or \
               not in_bounds_polar(grid, polar.t_step, current_polar_voxel):
                voxels[-1].exit_t = t_ray_exit
                return voxels
            else:
                current_polar_voxel = (current_polar_voxel + polar.t_step) % grid.num_polar_sections()
                current_azimuthal_voxel = (current_azimuthal_voxel + azimuthal.t_step) % grid.num_azimuthal_sections()
        elif voxel_intersection == VoxelIntersectionType.RadialPolarAzimuthal:
            t = radial.t_max
            if not in_bounds_polar(grid, polar.t_step, current_polar_voxel):
                voxels[-1].exit_t = t_ray_exit
                return voxels
            if not in_bounds_azimuthal(grid, azimuthal.t_step, current_azimuthal_voxel):
                voxels[-1].exit_t = t_ray_exit
                return voxels
            current_radial_voxel += radial.t_step
            current_polar_voxel = (current_polar_voxel + polar.t_step) % grid.num_polar_sections()
            current_azimuthal_voxel = (current_azimuthal_voxel + azimuthal.t_step) % grid.num_azimuthal_sections()
            
        voxels.append(Voxel(current_radial_voxel, current_polar_voxel, current_azimuthal_voxel))

def radial_hit(ray, grid, radial_step_has_transitioned, current_radial_voxel, v, rsvd_minus_v_squared, t, max_t):
    if radial_step_has_transitioned:
        d_b = math.sqrt(grid.delta_radii_squared(current_radial_voxel - 1) - rsvd_minus_v_squared)
        intersection_t = ray.time_of_intersection_at(v + d_b)
        if intersection_t < max_t:
            return HitParameters(intersection_t, -1)
        else:
            previous_idx = min(current_radial_voxel, grid.num_radial_sections() - 1)
            r_a = grid.delta_radii_squared(previous_idx - (1 if grid.delta_radii_squared(previous_idx) < rsvd_minus_v_squared else 0))
            d_a = math.sqrt(r_a - rsvd_minus_v_squared)
            t_entrance = ray.time_of_intersection_at(v - d_a)
            t_exit = ray.time_of_intersection_at(v + d_a)
            if t_entrance > t and is_equal(t_entrance, t_exit):
                radial_step_has_transitioned = True
                return HitParameters(t_entrance, 0)
            elif t_entrance > t and t_entrance < max_t:
                return HitParameters(t_entrance, 1)
            elif t_exit < max_t:
                radial_step_has_transitioned = True
                return HitParameters(t_exit, -1)
            else:
                return HitParameters(float('inf'), 0)
    else:
        return radial_hit(ray, grid, True, current_radial_voxel, v, rsvd_minus_v_squared, t, max_t)

def polar_hit(ray, grid, ray_segment, collinear_times, current_polar_voxel, t, max_t):
    p_one = BoundVec3(grid.p_max_polar(current_polar_voxel).P1, grid.p_max_polar(current_polar_voxel).P2, 0.0)
    p_two = BoundVec3(grid.p_max_polar(current_polar_voxel + 1).P1, grid.p_max_polar(current_polar_voxel + 1).P2, 0.0)
    u_min = grid.center_to_polar_bound(current_polar_voxel)
    u_max = grid.center_to_polar_bound(current_polar_voxel + 1)
    w_min = p_one - ray_segment.P1
    w_max = p_two - ray_segment.P1
    
    perp_uv_min = u_min.x * ray_segment.vector.y - u_min.y * ray_segment.vector.x
    perp_uv_max = u_max.x * ray_segment.vector.y - u_max.y * ray_segment.vector.x
    perp_uw_min = u_min.x * w_min.y - u_min.y * w_min.x
    perp_uw_max = u_max.x * w_max.y - u_max.y * w_max.x
    perp_vw_min = ray_segment.vector.x * w_min.y - ray_segment.vector.y * w_min.x
    perp_vw_max = ray_segment.vector.x * w_max.y - ray_segment.vector.y * w_max.x
    
    return angular_hit(grid, ray, perp_uv_min, perp_uv_max, perp_uw_min, perp_uw_max, perp_vw_min, perp_vw_max, ray_segment, collinear_times, t, max_t, ray.direction.y, grid.sphere_center.y, grid.p_max_polar(), current_polar_voxel)

def azimuthal_hit(ray, grid, ray_segment, collinear_times, current_azimuthal_voxel, t, max_t):
    p_one = BoundVec3(grid.p_max_azimuthal(current_azimuthal_voxel).P1, grid.p_max_azimuthal(current_azimuthal_voxel).P2, 0.0)
    p_two = BoundVec3(grid.p_max_azimuthal(current_azimuthal_voxel + 1).P1, grid.p_max_azimuthal(current_azimuthal_voxel + 1).P2, 0.0)
    u_min = grid.center_to_azimuthal_bound(current_azimuthal_voxel)
    u_max = grid.center_to_azimuthal_bound(current_azimuthal_voxel + 1)
    w_min = p_one - ray_segment.P1
    w_max = p_two - ray_segment.P1
    
    perp_uv_min = u_min.x * ray_segment.vector.y - u_min.y * ray_segment.vector.x
    perp_uv_max = u_max.x * ray_segment.vector.y - u_max.y * ray_segment.vector.x
    perp_uw_min = u_min.x * w_min.y - u_min.y * w_min.x
    perp_uw_max = u_max.x * w_max.y - u_max.y * w_max.x
    perp_vw_min = ray_segment.vector.x * w_min.y - ray_segment.vector.y * w_min.x
    perp_vw_max = ray_segment.vector.x * w_max.y - ray_segment.vector.y * w_max.x
    
    return angular_hit(grid, ray, perp_uv_min, perp_uv_max, perp_uw_min, perp_uw_max, perp_vw_min, perp_vw_max, ray_segment, collinear_times, t, max_t, ray.direction.z, grid.sphere_center.z, grid.p_max_azimuthal(), current_azimuthal_voxel)

def angular_hit(grid, ray, perp_uv_min, perp_uv_max, perp_uw_min, perp_uw_max, perp_vw_min, perp_vw_max, ray_segment, collinear_times, t, max_t, ray_direction_2, sphere_center_2, P_max, current_voxel):
    is_parallel_min = is_equal(perp_uv_min, 0.0)
    is_collinear_min = is_parallel_min and is_equal(perp_uw_min, 0.0) and is_equal(perp_vw_min, 0.0)
    is_parallel_max = is_equal(perp_uv_max, 0.0)
    is_collinear_max = is_parallel_max and is_equal(perp_uw_max, 0.0) and is_equal(perp_vw_max, 0.0)
    
    a, b = 0.0, 0.0
    t_min = collinear_times[1 if is_collinear_min else 0]
    is_intersect_min = False
    
    if not is_parallel_min:
        inv_perp_uv_min = 1.0 / perp_uv_min
        a = perp_vw_min * inv_perp_uv_min
        b = perp_uw_min * inv_perp_uv_min
        if not ((a < 0.0 or a > 1.0) or (b < 0.0 or b > 1.0)):
            is_intersect_min = True
            t_min = ray_segment.intersection_time_at(b, ray)
            
    t_max = collinear_times[1 if is_collinear_max else 0]
    is_intersect_max = False
    
    if not is_parallel_max:
        inv_perp_uv_max = 1.0 / perp_uv_max
        a = perp_vw_max * inv_perp_uv_max
        b = perp_uw_max * inv_perp_uv_max
        if not ((a < 0.0 or a > 1.0) or (b < 0.0 or b > 1.0)):
            is_intersect_max = True
            t_max = ray_segment.intersection_time_at(b, ray)
            
    t_t_max_eq = is_equal(t, t_max)
    t_max_within_bounds = t < t_max and not t_t_max_eq and t_max < max_t
    t_t_min_eq = is_equal(t, t_min)
    t_min_within_bounds = t < t_min and not t_t_min_eq and t_min < max_t
    
    if not t_max_within_bounds and not t_min_within_bounds:
        return HitParameters(float('inf'), 0)
        
    if is_intersect_max and not is_intersect_min and not is_collinear_min and t_max_within_bounds:
        return HitParameters(t_max, 1)
        
    if is_intersect_min and not is_intersect_max and not is_collinear_max and t_min_within_bounds:
        return HitParameters(t_min, -1)
        
    if (is_intersect_min and is_intersect_max) or \
       (is_intersect_min and is_collinear_max) or \
       (is_intersect_max and is_collinear_min):
        min_max_eq = is_equal(t_min, t_max)
        if min_max_eq and t_min_within_bounds:
            perturbed_t = 0.1
            a = -ray.direction.x * perturbed_t
            b = -ray_direction_2 * perturbed_t
            max_radius_over_plane_length = grid.sphere_max_radius() / math.sqrt(a*a + b*b)
            p1 = grid.sphere_center.x - max_radius_over_plane_length * a
            p2 = sphere_center_2 - max_radius_over_plane_length * b
            next_step = abs(current_voxel - calculate_angular_voxel_id_from_points(P_max, p1, p2))
            return HitParameters(t_max, next_step if (ray.direction.x < 0.0 or ray_direction_2 < 0.0) else -next_step)
            
        if t_min_within_bounds and ((t_min < t_max and not min_max_eq) or t_t_max_eq):
            return HitParameters(t_min, -1)
            
        if t_max_within_bounds and ((t_max < t_min and not min_max_eq) or t_t_min_eq):
            return HitParameters(t_max, 1)
            
    return HitParameters(float('inf'), 0)

def calculate_angular_voxel_id_from_points(P_max, p1, p2):
    for i in range(len(P_max) - 1):
        j = i + 1
        X_diff = P_max[j].P1 - P_max[i].P1
        Y_diff = P_max[j].P2 - P_max[i].P2
        X_p1_diff = p1 - P_max[i].P1
        X_p2_diff = p2 - P_max[i].P1
        Y_p1_diff = p1 - P_max[i].P2
        Y_p2_diff = p2 - P_max[i].P2
        
        d1d2 = X_p1_diff**2 + X_p2_diff**2 + Y_p1_diff**2 + Y_p2_diff**2
        d3 = X_diff**2 + Y_diff**2
        
        if d1d2 < d3 or is_equal(d1d2, d3):
            return i
    return len(P_max) + 1

def initialize_angular_voxel_id(grid, num_sections, ray_sphere, P, coord, center_coord, entry_radius):
    if num_sections == 1:
        return 0
    sed = (ray_sphere.x - grid.sphere_center.x)**2 + (ray_sphere.y - grid.sphere_center.y)**2
    if sed == 0.0:
        return 0
    r = entry_radius / math.sqrt(sed)
    p1 = grid.sphere_center.x - ray_sphere.x * r
    p2 = grid.sphere_center.y - ray_sphere.y * r
    return calculate_angular_voxel_id_from_points(P, p1, p2)

def in_bounds_polar(grid, step, current_voxel):
    radian = (current_voxel + 1) * grid.delta_theta()
    angval = radian - abs(step * grid.delta_theta())
    return grid.sphere_max_bound_polar() >= angval >= grid.sphere_min_bound_polar()

def in_bounds_azimuthal(grid, step, current_voxel):
    radian = (current_voxel + 1) * grid.delta_phi()
    angval = radian - abs(step * grid.delta_phi())
    return grid.sphere_max_bound_azi() >= angval >= grid.sphere_min_bound_azi()

def minimum_intersection(radial, polar, azimuthal):
    min_t = radial.t_max
    result = VoxelIntersectionType.Radial
    if polar.t_max < min_t:
        min_t = polar.t_max
        result = VoxelIntersectionType.Polar
    if azimuthal.t_max < min_t:
        min_t = azimuthal.t_max
        result = VoxelIntersectionType.Azimuthal
    if polar.t_max == min_t:
        result = VoxelIntersectionType.RadialPolar
    if azimuthal.t_max == min_t:
        result = VoxelIntersectionType.RadialAzimuthal
    if polar.t_max == min_t and azimuthal.t_max == min_t:
        result = VoxelIntersectionType.RadialPolarAzimuthal
    return result

def is_equal(a, b):
    diff_x = abs(a - b)
    return diff_x <= 1e-12 or diff_x <= max(abs(a), abs(b)) * 1e-8

class Voxel:
    def __init__(self, radial, polar, azimuthal):
        self.radial = radial
        self.polar = polar
        self.azimuthal = azimuthal
        self.exit_t = None

class HitParameters:
    def __init__(self, tMax, tStep):
        self.tMax = tMax
        self.tStep = tStep

class RaySegment:
    def __init__(self, max_t, ray):
        self.P1 = ray.point_at_parameter(0.0)
        self.P2 = ray.point_at_parameter(max_t)
        self.NZDI = ray.non_zero_direction_index()
        self.ray_segment = self.P2 - self.P1
    def update_at_time(self, t, ray):
        self.P1 = ray.point_at_parameter(t)
        self.ray_segment = self.P2 - self.P1
    def vector(self):
        return self.ray_segment
    def intersection_time_at(self, intersect_parameter, ray):
        return (self.P1[self.NZDI] + self.ray_segment[self.NZDI] * intersect_parameter - ray.origin()[self.NZDI]) * ray.inv_direction()[self.NZDI]

class BoundVec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __sub__(self, other):
        return FreeVec3(self.x - other.x, self.y - other.y, self.z - other.z)
    def __getitem__(self, idx):
        return [self.x, self.y, self.z][idx]

class FreeVec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __sub__(self, other):
        return FreeVec3(self.x - other.x, self.y - other.y, self.z - other.z)
    def __getitem__(self, idx):
        return [self.x, self.y, self.z][idx]
    def dot(self, other):
        return self.x * other.x + self.y * other.y + self.z * other.z
    def squared_length(self):
        return self.x**2 + self.y**2 + self.z**2

class UnitVec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __getitem__(self, idx):
        return [self.x, self.y, self.z][idx]

class Ray:
    def __init__(self, origin, direction):
        self.origin_ = origin
        self.direction_ = direction
        self.inverse_direction_ = UnitVec3(1.0/direction.x, 1.0/direction.y, 1.0/direction.z)
        self.NZD_index_ = 0 if direction.x != 0 else (1 if direction.y != 0 else 2)
    def point_at_parameter(self, t):
        return BoundVec3(self.origin_.x + self.direction_.x * t, self.origin_.y + self.direction_.y * t, self.origin_.z + self.direction_.z * t)
    def time_of_intersection_at(self, discriminant_v):
        return self.direction_[self.NZD_index_] * discriminant_v * self.inverse_direction_[self.NZD_index_]
    def non_zero_direction_index(self):
        return self.NZD_index_
    def inv_direction(self):
        return self.inverse_direction_

class VoxelIntersectionType:
    Radial = 1
    Polar = 2
    Azimuthal = 3
    RadialPolar = 4
    RadialAzimuthal = 5
    PolarAzimuthal = 6
    RadialPolarAzimuthal = 7
```

### Parameters
- `DOUBLE_MAX`: `float('inf')`, used as infinity for `tMax` when no intersection is found [S1, unofficial].
- `perturbed_t`: `0.1`, used to perturb the ray direction when handling degenerate cases where `t_min == t_max` in angular hit [S1, unofficial].

### Edge Cases and Pitfalls
- Tangential radial hit: when `t_entrance == t_exit` and `t_entrance > t`, the ray is tangent to the radial sphere boundary. Handled by setting `radial_step_has_transitioned = true` and returning `tStep = 0` [S1, unofficial].
- Degenerate angular intersection: when `t_min == t_max` and both are within bounds, the ray intersects at a single point on the boundary. Handled by perturbing the ray direction by `0.1` units and recalculating the voxel ID to determine the step direction [S1, unofficial].

## Operating characteristics
- Benchmarks on 8 × 1400 MHz CPUs (L1 Data 32 KiB × 4, L1 Instruction 32 KiB × 4, L2 Unified 256 KiB × 4, L3 Unified 6144 KiB × 1) [S1, unofficial].
- 128² rays, 64³ voxels: Mean 103 ms, Median 103 ms, Std Dev 1.1 ms [S1, unofficial].
- 512² rays, 128³ voxels: Mean 3187 ms, Median 3180 ms, Std Dev 8.3 ms [S1, unofficial].
- Scaling is roughly linear with ray count and voxel count, consistent with traversal algorithms [S1, unofficial].
- Strengths: Handles spherical grids efficiently without converting to cartesian [S1, unofficial]. Limitations: Requires precomputed boundary segments and angular voxel ID calculations, which add overhead for non-spherical scenes [S1, unofficial].

## Background
- Origin: Developed by Chris Gyurgyik, Ariel Kellison, and Youhan Yuan as an extension to the yt open-source data analysis and visualization package [S1, unofficial].
- Based on Amanatides and Woo's 1987 seminal paper on fast voxel traversal for ray tracing [S1, unofficial].
- Used for visualizing physical data from non-cartesian grids within yt [S1, unofficial].

## Sources
[S1] spherical-volume-rendering/svr-algorithm, https://github.com/spherical-volume-rendering/svr-algorithm, fetched 2026-10-24 (unofficial)
