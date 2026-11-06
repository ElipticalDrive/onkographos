# Finite Layer Cloud Shape Generation

Also called: Layer Cloud Side Profile, Natural XY Edge Envelope

## Overview
Finite Layer Cloud Shape Generation is a procedural method for constructing the 3D volume shapes of layer cloud types (Cirrus, Altostratus, Nimbostratus, Stratocumulus, Stratus) by assembling a chain of connected 3D ellipsoidal lobes [S1, unofficial]. It solves the problem of generating realistic, non-cylindrical cloud boundaries by ensuring vertical presence variation and horizontal tapering, avoiding the artifact of simply extruding a 2D contour into a column [S1, unofficial]. It differs from its siblings by focusing on horizontal layering and lobe chaining for flat cloud structures, whereas [Cumulonimbus and Cumulus Shape Generation](8296-cumulonimbus-and-cumulus-shape-generation.md) uses distinct lobe strategies for vertical development and irregular tops [S1, unofficial].

## How it works
**Inputs:**
- Cloud type ID (`type_id`): Integer identifying the cloud type. Must be in `LAYER_CLOUD_TYPE_IDS` (3, 5, 6, 7, 8) [S1, unofficial].
- Grid coordinates: `x`, `y`, `z` arrays of shape `(H, W, D)` representing the 3D volume space [S1, unofficial].
- Random seed/state: For procedural variation [S1, unofficial].

**Outputs:**
- Density field: A float32 array of shape `(H, W, D)` representing the organic cloud density [S1, unofficial].

**Data Structures:**
- `lobe_spec`: Tuple `(center_x, center_y, radius_x, radius_y, weight, lobe_z, lobe_z_radius)` [S1, unofficial].
- `shape_profile`: Tuple `(center_z, half_height, top_wobble, bottom_wobble, thickness_variance, cap_taper, side_edge_variation)` [S1, unofficial].

**Parameters:**
- `LAYER_CLOUD_TYPE_IDS`: `frozenset((3, 5, 6, 7, 8))` [S1, unofficial].
- Persistent support values (map `type_id` to float): `{3: 0.36, 5: 0.40, 6: 0.50, 7: 24, 8: 0.44}` [S1, unofficial]. Note: Source lists `7: 0.24` in one place and `7: 24` in another; the context of "persistent support" and other values (0.36, 0.40, etc.) implies `0.24` is the correct value [S1, unofficial].
- Type 6 (Nimbostratus) profile defaults: `center=-0.03`, `half=0.58`, `top_wobble=0.180`, `bottom_wobble=0.285`, `thickness_variance=0.40`, `cap_taper=0.39`, `side_edge_variation=1.10` [S1, unofficial].

**Steps:**

1. **Generate Lobe Specifications:**
   a. Determine `major_angle = uniform(-pi, pi)` [S1, unofficial].
   b. Calculate `direction = (cos(major_angle), sin(major_angle))` and `perpendicular = (-direction[1], direction[0])` [S1, unofficial].
   c. Set `major_length = uniform(0.45, 0.72)` and `cross_width = uniform(0.22, 0.46)` [S1, unofficial].
   d. Get `shape_profile` for `type_id`. If `type_id` not in `LAYER_CLOUD_TYPE_IDS`, use profile ID 5 [S1, unofficial]. Extract `center_z` and `half_height` [S1, unofficial].
   e. Generate `count` positions (uniformly 7-13) along the major axis: `positions_along = linspace(-0.88, 0.88, count) + normal(0.0, 0.09, size=count)` [S1, unofficial].
   f. For each `position_along`:
      - `along = position_along * major_length`
      - `cross = normal(0.0, cross_width * 0.62)`
      - `position = (center_x, center_y) + direction * along + perpendicular * cross` [S1, unofficial].
      - `radius_x = uniform(0.14, 0.30)`, `radius_y = uniform(0.12, 0.28)` [S1, unofficial].
      - `lobe_z = center_z + uniform(-0.48, 0.48) * half_height`
      - `lobe_z_radius = uniform(0.58, 1.04) * half_height` [S1, unofficial].
      - `weight = uniform(0.54, 0.98)` [S1, unofficial].
      - Append `(position[0], position[1], radius_x, radius_y, weight, lobe_z, lobe_z_radius)` to `edge_lobe_specs` [S1, unofficial].
   g. Add 1-3 side lobes to prevent degeneracy:
      - `along = uniform(-0.58, 0.58) * major_length`
      - `cross = choice(-1, 1) * uniform(0.60, 1.18) * cross_width`
      - Generate `position` similarly to step f, using same radius/weight distributions [S1, unofficial].

2. **Evaluate Density Field:**
   a. Rotate coordinates: `local_x, local_y = rotated_xy(x, y, variation)` where `variation` is derived from `shape_profile` [S1, unofficial].
   b. Initialize `lobe_field = zeros_like(x, dtype=float32)` [S1, unofficial].
   c. For each `spec` in `edge_lobe_specs` (`center_x, center_y, radius_x, radius_y, weight, lobe_z, lobe_z_radius`):
      - Rotate lobe center: `lobe_center_x = cos(variation) * center_x + sin(variation) * center_y`, `lobe_center_y = -sin(variation) * center_x + cos(variation) * center_y` [S1, unofficial].
      - Calculate 2D lobe: `lobe = exp(-(((local_x - lobe_center_x) / (radius_x * 1.15))**2 + ((local_y - lobe_center_y) / (radius_y * 1.15))**2))` [S1, unofficial].
      - Calculate vertical presence:
        - `persistent_support = persistent_support_map[type_id]`
        - `vertical_presence = persistent_support + (1.0 - persistent_support) * exp(-((z - lobe_z) / max(lobe_z_radius, 0.001))**2)` [S1, unofficial].
      - Add to field: `lobe_field += float32(weight * vertical_presence) * lobe` [S1, unofficial].
   d. Calculate horizontal taper:
      - Get `center_z, half_height, _, _, _, horizontal_taper, side_variation` from `shape_profile` [S1, unofficial].
      - `normalized_height = abs(z - center_z) / max(half_height, 0.001)` [S1, unofficial].
      - `cap_taper = horizontal_taper * smoothstep((normalized_height - 0.12) / 0.88)` [S1, unofficial].
   e. Calculate contour:
      - `edge_noise`: Not defined in sources. [G] Assume a uniform random noise field in [0, 1] or a specific noise function provided by the parent context. Given the subtraction of 0.5, it is likely a noise value in [0, 1].
      - `contour = lobe_field + (edge_noise - 0.5) * (0.62 + 0.30 * side_variation) - cap_taper` [S1, unofficial].
   f. Apply smoothstep: `organic = smoothstep((contour - 0.30) / 0.48)` [S1, unofficial].
   g. Apply final edge guard:
      - `screen_distance = min(x + 1.0, 1.0 - x, y + 1.0, 1.0 - y)` [S1, unofficial].
      - `final_guard = safe_edge_fade(screen_distance, 0.0, 0.045)` [S1, unofficial].
   h. Return `organic * final_guard` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def smoothstep(t):
    t = np.clip(t, 0.0, 1.0)
    return t * t * (3.0 - 2.0 * t)

def safe_edge_fade(distance, min_val, max_val):
    # Assuming linear fade from 0 to max_val over distance range [0, max_val]
    # If distance > max_val, return 1.0? Or 0?
    # Source says "cleans only outermost voxels... main contour naturally goes to zero".
    # This implies fade out to 0.
    # safe_edge_fade(screen_distance, 0.0, 0.045)
    # Likely: if screen_distance < 0.045, fade from 1 to 0? Or 0 to 1?
    # "final_guard" is multiplied. If it's a guard, it likely fades to 0 at the edge.
    # Let's assume it fades from 1.0 at distance=0.045 to 0.0 at distance=0.0.
    # Or vice versa? "cleans only the outermost voxels".
    # If screen_distance is small (near edge), we want density to be 0.
    # So if screen_distance < 0.045, fade to 0.
    # Let's implement: fade from 1.0 at 0.045 to 0.0 at 0.0.
    t = np.clip(distance / 0.045, 0.0, 1.0)
    return 1.0 - smoothstep(t)

def rotated_xy(x, y, variation):
    # Variation is likely an angle in radians
    cos_v = np.cos(variation)
    sin_v = np.sin(variation)
    local_x = cos_v * x + sin_v * y
    local_y = -sin_v * x + cos_v * y
    return local_x, local_y

def generate_finite_layer_cloud(x, y, z, type_id, rng):
    LAYER_CLOUD_TYPE_IDS = frozenset((3, 5, 6, 7, 8))
    persistent_support_map = {3: 0.36, 5: 0.40, 6: 0.50, 7: 0.24, 8: 0.44}
    
    # 1. Determine profile
    if type_id not in LAYER_CLOUD_TYPE_IDS:
        profile_type_id = 5
    else:
        profile_type_id = type_id
        
    # Default profile for type 6 if not provided by external function
    # In real implementation, _layer_shape_profile would be called
    center_z = -0.03
    half_height = 0.58
    horizontal_taper = 0.39
    side_variation = 1.10
    
    # 2. Generate lobe specs
    major_angle = rng.uniform(-np.pi, np.pi)
    direction = np.array([np.cos(major_angle), np.sin(major_angle)])
    perpendicular = np.array([-direction[1], direction[0]])
    
    major_length = rng.uniform(0.45, 0.72)
    cross_width = rng.uniform(0.22, 0.46)
    
    count = rng.integers(7, 14) # 7-13 inclusive
    positions_along = np.linspace(-0.88, 0.88, count) + rng.normal(0.0, 0.09, size=count)
    
    edge_lobe_specs = []
    
    # Main chain
    for pos_along in positions_along:
        along = pos_along * major_length
        cross = rng.normal(0.0, cross_width * 0.62)
        # Assuming center_x, center_y are 0 for simplicity if not provided
        center_x, center_y = 0.0, 0.0
        position = np.array([center_x, center_y]) + direction * along + perpendicular * cross
        
        radius_x = rng.uniform(0.14, 0.30)
        radius_y = rng.uniform(0.12, 0.28)
        
        lobe_z = center_z + rng.uniform(-0.48, 0.48) * half_height
        lobe_z_radius = rng.uniform(0.58, 1.04) * half_height
        weight = rng.uniform(0.54, 0.98)
        
        edge_lobe_specs.append((position[0], position[1], radius_x, radius_y, weight, lobe_z, lobe_z_radius))
        
    # Side lobes
    num_side_lobes = rng.integers(1, 4) # 1-3
    for _ in range(num_side_lobes):
        along = rng.uniform(-0.58, 0.58) * major_length
        cross_sign = rng.choice([-1, 1])
        cross = cross_sign * rng.uniform(0.60, 1.18) * cross_width
        
        center_x, center_y = 0.0, 0.0
        position = np.array([center_x, center_y]) + direction * along + perpendicular * cross
        
        radius_x = rng.uniform(0.14, 0.30)
        radius_y = rng.uniform(0.12, 0.28)
        
        lobe_z = center_z + rng.uniform(-0.48, 0.48) * half_height
        lobe_z_radius = rng.uniform(0.58, 1.04) * half_height
        weight = rng.uniform(0.54, 0.98)
        
        edge_lobe_specs.append((position[0], position[1], radius_x, radius_y, weight, lobe_z, lobe_z_radius))

    # 3. Evaluate field
    # variation for rotation is not explicitly defined in sources, assuming 0 or derived from profile
    variation = 0.0 
    local_x, local_y = rotated_xy(x, y, variation)
    
    lobe_field = np.zeros_like(x, dtype=np.float32)
    
    for spec in edge_lobe_specs:
        center_x, center_y, radius_x, radius_y, weight, lobe_z, lobe_z_radius = spec
        
        lobe_center_x = np.cos(variation) * center_x + np.sin(variation) * center_y
        lobe_center_y = -np.sin(variation) * center_x + np.cos(variation) * center_y
        
        lobe = np.exp(-(((local_x - lobe_center_x) / (radius_x * 1.15))**2 + ((local_y - lobe_center_y) / (radius_y * 1.15))**2))
        
        persistent_support = persistent_support_map.get(type_id, 0.40) # Default to 5 if not found
        vertical_presence = persistent_support + (1.0 - persistent_support) * np.exp(-((z - lobe_z) / max(lobe_z_radius, 0.001))**2)
        
        lobe_field += np.float32(weight * vertical_presence) * lobe
        
    normalized_height = np.abs(z - center_z) / max(half_height, 0.001)
    cap_taper = horizontal_taper * smoothstep((normalized_height - 0.12) / 0.88)
    
    # edge_noise is not defined. [G] Using uniform noise for demonstration
    edge_noise = rng.uniform(0.0, 1.0, size=x.shape)
    contour = lobe_field + (edge_noise - 0.5) * (0.62 + 0.30 * side_variation) - cap_taper
    
    organic = smoothstep((contour - 0.30) / 0.48)
    
    screen_distance = np.minimum.reduce([x + 1.0, 1.0 - x, y + 1.0, 1.0 - y])
    final_guard = safe_edge_fade(screen_distance, 0.0, 0.045)
    
    return organic * final_guard
```

## Subtopics

### [Cumulonimbus and Cumulus Shape Generation](8296-cumulonimbus-and-cumulus-shape-generation.md)
Procedural generation of non-spherical, multi-lobed cloud shapes for Cumulus (type 9) and Cumulonimbus (type 10) clouds, creating irregular tops and bases rather than simple spheres. It differs from its parent [Finite Layer Cloud Shape Generation](8292-finite-layer-cloud-shape-generation.md) by using distinct lobe generation strategies for vertical development (Cumulonimbus) and flat, multi-lobed structures (Cumulus), whereas the parent method focuses on horizontal layering for other cloud types.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels and the number of lobes. Each voxel is evaluated against each lobe.
- **Memory Complexity:** Linear with respect to the number of voxels for the output density field.
- **Performance:** Not given in the sources.
- **Strengths:** Generates realistic, non-cylindrical layer cloud shapes with natural edge variation and vertical presence.
- **Limitations:** Relies on procedural randomization, which may require tuning for specific artistic styles. The method is specific to layer cloud types.
- **When to use:** When generating Cirrus, Altostratus, Nimbostratus, Stratocumulus, or Stratus clouds.
- **When not to use:** For Cumulus or Cumulonimbus clouds, which require different shape generation strategies.

## Background
The method is part of the "Noise-Cloud" project, an unofficial implementation of procedural cloud generation for game engines [S1, unofficial]. It is designed to work within a larger procedural volume cloud generation pipeline, specifically for layer cloud types.

## Sources
[S1] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
