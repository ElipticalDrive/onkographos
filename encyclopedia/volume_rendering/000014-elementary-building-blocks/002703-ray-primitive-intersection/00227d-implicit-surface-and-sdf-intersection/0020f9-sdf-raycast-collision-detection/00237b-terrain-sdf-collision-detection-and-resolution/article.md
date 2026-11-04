# Terrain SDF Collision Detection and Resolution

Also called: SDF Collision Detection, SDF Sign Change Collision

## Overview
This method detects collisions between a moving object and a terrain represented by a Signed Distance Field (SDF) by evaluating the distance at the object's previous and current positions. It identifies collisions via sign changes in the distance values and estimates the surface normal to resolve penetration. It distinguishes itself from continuous collision detection by relying on discrete time-step checks, which is simpler to implement but prone to tunneling at high velocities [S1, unofficial].

## How it works
**Inputs:**
- `terrainSDFS`: The terrain SDF evaluator.
- `oldPosition`: The object's position at the start of the time step.
- `currentPosition`: The object's position at the end of the time step.
- `currentVelocity`: The object's velocity vector.

**Outputs:**
- `justCollided`: Boolean flag indicating if a collision occurred.
- `resolution`: A velocity correction vector (or impulse) to resolve the collision.

**Data Structures:**
- `float`: Signed distance value. Positive indicates outside the terrain, negative indicates inside.
- `Vector3`: 3D position and normal vectors.

**Parameters:**
- `beta`: Damping factor for collision resolution. Typical value: 0.1–0.3 [S1, unofficial].
- `contactOffset`: Contact offset distance. Typical value: 3–10 mm [S1, unofficial].
- `slop`: Slop distance. Typical value: 1–3 mm [S1, unofficial].
- `frictionMu`: Friction coefficient. Typical value: 0.6f [S1, unofficial].

**Steps:**
1. Evaluate the signed distance at the old position:
   $$ d_{old} = \text{terrainSFS}.EvaluateDistanceAt(\text{oldPosition}) $$
2. Evaluate the signed distance at the current position:
   $$ d_{new} = \text{terrainSFS}.EvaluateDistanceAt(\text{currentPosition}) $$
3. Check for collision by detecting a sign change:
   $$ \text{collision} = (d_{old} > 0 \land d_{new} < 0) \lor (d_{old} < 0 \land d_{new} > 0) $$
4. If a collision is detected:
   a. Estimate the surface normal at the old position using the gradient of the SDF:
      $$ \mathbf{n} = \text{EstimateGradient}(\text{terrainSFS}, \text{oldPosition}) $$
   b. If the new distance is positive (object is now outside), invert the normal:
      $$ \text{if } d_{new} > 0: \mathbf{n} = -\mathbf{n} $$
   c. Set `justCollided = true`.
   d. Return the resolution result. The source specifies returning the negative of the current velocity:
      $$ \text{resolution} = -1 \times \text{currentVelocity} $$
      [G] In practice, this resolution is often modified by the damping factor `beta` and friction `frictionMu` to produce a physically plausible impulse, though the source code snippet explicitly returns `-1 * currentVelocity` as a placeholder for `TerrainCollision.Resolve(...)`.
5. If no collision is detected:
   a. Set `justCollided = false`.
   b. Return `Vector3.zero`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def TerrainSDFCollisionDetectionAndResolution(terrainSFS, oldPosition, currentPosition, currentVelocity):
    # Evaluate distances
    oldDistance = terrainSFS.EvaluateDistanceAt(oldPosition)
    newDistance = terrainSFS.EvaluateDistanceAt(currentPosition)
    
    # Check for sign change
    if (oldDistance > 0 and newDistance < 0) or (oldDistance < 0 and newDistance > 0):
        # Collision detected
        justCollided = True
        
        # Estimate surface normal at old position
        surfaceNormal = EstimateGradient(terrainSFS, oldPosition)
        
        # Invert normal if object is now outside
        if newDistance > 0:
            surfaceNormal = -surfaceNormal
            
        # Resolve collision
        # Note: Source returns -1 * currentVelocity directly
        # In a full physics system, this would be processed with beta, frictionMu, etc.
        resolution = -1 * currentVelocity
        
        return justCollided, resolution
    else:
        # No collision
        justCollided = False
        return justCollided, Vector3.zero

def EstimateGradient(terrainSFS, position):
    # [G] Standard central difference approximation for gradient
    epsilon = 0.001
    dx = terrainSFS.EvaluateDistanceAt(position + Vector3(epsilon, 0, 0)) - terrainSFS.EvaluateDistanceAt(position - Vector3(epsilon, 0, 0))
    dy = terrainSFS.EvaluateDistanceAt(position + Vector3(0, epsilon, 0)) - terrainSFS.EvaluateDistanceAt(position - Vector3(0, epsilon, 0))
    dz = terrainSFS.EvaluateDistanceAt(position + Vector3(0, 0, epsilon)) - terrainSFS.EvaluateDistanceAt(position - Vector3(0, 0, epsilon))
    
    gradient = Vector3(dx, dy, dz) / (2 * epsilon)
    return gradient.normalized
```

**Edge Cases and Pitfalls:**
- **Tunneling:** If the object moves very fast, it may pass through the terrain without the sign change being detected if the step size is larger than the terrain thickness or if the SDF is not evaluated densely enough. This method does not guarantee continuous collision detection.
- **Precision:** Floating-point precision issues may cause false sign changes or missed collisions near the surface. Using `contactOffset` and `slop` can help mitigate this by defining a tolerance zone.
- **Gradient Estimation:** The gradient estimation requires multiple SDF evaluations. If the SDF is not smooth or has discontinuities, the normal estimation may be inaccurate.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ per object per time step, assuming constant-time SDF evaluation. The gradient estimation adds a constant factor (typically 6 evaluations for central differences).
- **Memory Complexity:** $O(1)$ auxiliary memory.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of objects. Performance is dominated by the SDF evaluation cost.
- **Strengths:** Simple to implement; low computational overhead per object; works well for slow-moving objects.
- **Limitations:** Prone to tunneling at high velocities; does not handle continuous penetration; resolution is simplistic (source returns `-1 * velocity` which may cause jitter or unrealistic behavior without further physics processing).
- **When to use:** Suitable for simple games or simulations where objects move slowly and high precision is not critical.
- **When not to use:** Not suitable for high-speed projectiles or precise physics simulations where tunneling is unacceptable.

## Background
This method is derived from basic SDF collision detection techniques used in real-time graphics and physics. The specific implementation details, including the sign change check and the simplistic resolution, are found in the Marching Cubes Terrain project by 2JacT [S1, unofficial]. The use of parameters like `beta`, `contactOffset`, and `slop` suggests an integration with a constraint-based physics solver, although the core detection logic remains discrete.

## Sources
[S1] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
