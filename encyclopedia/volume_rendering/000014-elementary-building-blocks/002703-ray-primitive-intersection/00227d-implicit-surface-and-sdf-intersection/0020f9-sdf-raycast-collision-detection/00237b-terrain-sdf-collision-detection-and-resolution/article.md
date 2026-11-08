# Terrain SDF Collision Detection and Resolution

Also called: SDF Collision Detection, SDF Sign Change Collision

## Overview
This method performs collision detection and resolution for terrain using a Signed Distance Field (SDF) by evaluating the distance at an object's previous and current positions [S1, unofficial]. It identifies collisions via sign changes in the distance values and estimates the surface normal to resolve penetration [S1, unofficial]. It differs from its parent [SDF Raycast Collision Detection](2417-sdf-raycast-collision-detection.md) by explicitly handling the resolution step and normal estimation rather than just reporting intersection [S1, unofficial].

## How it works
**Inputs:**
- `terrainSDFS`: The SDF evaluator for the terrain.
- `oldPosition`: The object's position at the start of the time step.
- `currentPosition`: The object's position at the end of the time step.
- `currentVelocity`: The object's velocity vector.

**Outputs:**
- `resolution`: A velocity vector correction (negative of current velocity if collision detected).
- `justCollided`: A boolean flag indicating if a collision occurred.

**Data Structures:**
- `float`: Signed distance value.
- 3D vector: Position, normal, and velocity components.

**Parameters:**
- `beta`: Damping factor for collision resolution. Typical value: 0.1–0.3 [S1, unofficial].
- `contactOffset`: Contact offset distance. Typical value: 3–10 mm [S1, unofficial].
- `slop`: Slop distance. Typical value: 1–3 mm [S1, unofficial].
- `frictionMu`: Friction coefficient. Typical value: 0.6f [S1, unofficial].

**Steps:**
1. Evaluate the signed distance at the old position:
   $$ d_{\text{old}} = \text{terrainSFS}.\text{EvaluateDistanceAt}(\text{oldPosition}) $$
2. Evaluate the signed distance at the current position:
   $$ d_{\text{new}} = \text{terrainSFS}.\text{EvaluateDistanceAt}(\text{currentPosition}) $$
3. Check for collision by detecting a sign change:
   $$ \text{collision} = (d_{\text{old}} > 0 \land d_{\text{new}} < 0) \lor (d_{\text{old}} < 0 \land d_{\text{new}} > 0) $$
4. If a collision is detected:
   a. Estimate the surface normal at the old position:
      $$ \mathbf{n} = \text{EstimateGradient}(\text{terrainSFS}, \text{oldPosition}) $$
   b. If the new distance is positive (outside), invert the normal:
      $$ \text{if } d_{\text{new}} > 0: \mathbf{n} = -\mathbf{n} $$
   c. Set `justCollided = true`.
   d. Return the resolution result:
      $$ \text{resolution} = -1 \times \text{currentVelocity} $$
5. If no collision is detected:
   a. Set `justCollided = false`.
   b. Return `zero`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def terrain_sdf_collision_detection_resolution(terrain_sdfs, old_position, current_position, current_velocity):
    old_distance = terrain_sdfs.evaluate_distance_at(old_position)
    new_distance = terrain_sdfs.evaluate_distance_at(current_position)
    
    collision = (old_distance > 0 and new_distance < 0) or (old_distance < 0 and new_distance > 0)
    
    if collision:
        surface_normal = estimate_gradient(terrain_sdfs, old_position)
        if new_distance > 0:
            surface_normal = [-x for x in surface_normal]
        just_collided = True
        resolution = [-v for v in current_velocity]
    else:
        just_collided = False
        resolution = [0.0, 0.0, 0.0]
        
    return resolution, just_collided

def estimate_gradient(sdfs, position):
    epsilon = 1e-4
    dx = sdfs.evaluate_distance_at([position[0] + epsilon, position[1], position[2]]) - sdfs.evaluate_distance_at([position[0] - epsilon, position[1], position[2]])
    dy = sdfs.evaluate_distance_at([position[0], position[1] + epsilon, position[2]]) - sdfs.evaluate_distance_at([position[0], position[1] - epsilon, position[2]])
    dz = sdfs.evaluate_distance_at([position[0], position[1], position[2] + epsilon]) - sdfs.evaluate_distance_at([position[0], position[1], position[2] - epsilon])
    magnitude = (dx*dx + dy*dy + dz*dz)**0.5
    if magnitude == 0:
        return [0.0, 0.0, 0.0]
    return [dx/magnitude, dy/magnitude, dz/magnitude]
```

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### [ERP/Baumgarte Collision Resolution](4385-erp-baumgarte-collision-resolution.md)
Uses SDF for depth and normal, applies Baumgarte stabilization for penetration correction and inward velocity damping. Choose when smoother resolution is needed.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
