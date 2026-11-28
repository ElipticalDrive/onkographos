# Visible Light Collection
Also called: ULightComponent Iteration and Filtering

## Overview
Visible Light Collection is a pre-processing step in volumetric rendering that aggregates all active, non-directional light sources affecting the current frame. It solves the problem of identifying which lights are relevant for subsequent shading or shadowing passes by filtering out invisible, directional, or invalid sources. It differs from sibling methods like [Direct Illumination with Shadow Rays](2011-direct-illumination-with-shadow-rays.md) by focusing solely on light source enumeration and culling rather than geometric visibility testing or radiometric integration.

## How it works
**Inputs:**
- The current world context.
- A list of all registered `ULightComponent` instances.

**Outputs:**
- A collection (`OutLights`) of `PointLight` structures containing position and radius data.

**Data Structures:**
- `PointLight`: A structure containing:
  - `PositionWS`: A 3D vector representing the light's position in World Space.
  - `Radius`: A float representing the bounding sphere radius of the light.

**Steps:**
1. Initialize an empty list `OutLights`.
2. Iterate through all `ULightComponent` instances in the world using a global iterator.
3. For each light component, evaluate the following filter conditions:
   - The light must be registered.
   - The light must belong to the current world.
   - The light must affect the world.
   - The light must be visible.
   - The light must **not** be a directional light.
   - The light's bounding sphere radius must be greater than or equal to `KINDA_SMALL_NUMBER` (a minimal epsilon value to avoid degenerate spheres).
4. If all conditions are met:
   - Extract the light's bounding sphere center.
   - Set `Ld.PositionWS` to the center converted to `FVector3f`.
   - Extract the light's bounding sphere radius.
   - Set `Ld.Radius` to the sphere's `W` component (radius).
   - Append `Ld` to `OutLights`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def VisibleLightCollection(world, KINDA_SMALL_NUMBER):
    OutLights = []
    
    # Iterate all light components in the world
    for light_component in world.GetLightComponents():
        # Filter conditions
        if not light_component.IsRegistered():
            continue
        if not light_component.BelongsToCurrentWorld():
            continue
        if not light_component.AffectsWorld():
            continue
        if not light_component.IsVisible():
            continue
        if light_component.IsDirectional():
            continue
        
        bounding_sphere = light_component.GetBoundingSphere()
        
        # Check radius threshold
        if bounding_sphere.radius < KINDA_SMALL_NUMBER:
            continue
            
        # Extract data
        light_data = {
            "PositionWS": bounding_sphere.center.to_fvector3f(),
            "Radius": bounding_sphere.w
        }
        
        OutLights.append(light_data)
        
    return OutLights
```

**Edge Cases and Pitfalls:**
- **Directional Lights:** Explicitly excluded. Directional lights require different handling (e.g., infinite distance approximation) and are not collected here.
- **Degenerate Spheres:** Lights with zero or near-zero radius are filtered out to prevent numerical instability in subsequent shading calculations.
- **Visibility:** Lights occluded by geometry or outside the view frustum may still be included if they are marked "visible" in the component state; further culling (e.g., bounding box tests) is typically performed in later stages.

## Subtopics

### [Light Director Frame Update](19314-light-director-frame-update.md)
Manages the lifecycle of light slots, promoting static emitters based on camera proximity and handling dynamic light tracking. It consumes the output of Visible Light Collection to construct the final render list.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of `ULightComponent` instances in the world, $O(N)$, where $N$ is the total count of lights.
- **Memory Complexity:** Linear with respect to the number of visible, non-directional lights, $O(M)$, where $M \le N$.
- **Strengths:** Simple, deterministic, and fast. It provides a clean separation between light enumeration and shading logic.
- **Limitations:** Does not perform spatial culling (e.g., frustum or distance-based culling) itself; relies on the engine's component registration state. Excludes directional lights, requiring separate handling.

## Background
This method is part of the Froxel lighting system for Unreal Engine 5, as implemented in the unofficial repository by Armin Tajik. It serves as the foundational step for voxel-based volumetric lighting, ensuring that only relevant point and spot lights are processed in subsequent shadow casting and shading passes.

## Sources
[S1] ArminTajik/ue5-froxel-lighting, https://github.com/ArminTajik/ue5-froxel-lighting, fetched 2026-10-27 (unofficial)
