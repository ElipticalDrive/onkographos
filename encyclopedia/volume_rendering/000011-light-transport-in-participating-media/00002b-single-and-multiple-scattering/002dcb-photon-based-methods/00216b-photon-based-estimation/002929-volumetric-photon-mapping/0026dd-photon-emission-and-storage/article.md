# Photon Emission and Storage

Also called: Photon Mapping with Volume Storage, emitPhotons

## Overview
Photon Emission and Storage is the preprocessing phase of Volumetric Photon Mapping that generates photon paths from light sources and records their positions, directions, and energy densities into spatial data structures for both surface intersections and volumetric scattering events [S1, unofficial]. It solves the problem of efficiently capturing indirect lighting and caustics in participating media by decoupling the complex light transport simulation from the final image synthesis, allowing for density estimation during a subsequent gather pass [S1, unofficial]. Unlike direct integration methods, it pre-computes photon interactions, and it is distinguished from its sibling [Beam Radiance Estimate](1283-beam-radiance-estimate.md) by storing discrete photon packets rather than continuous beam radiance estimates [S1, unofficial].

## How it works
**Inputs:**
- Light source position `Light` (Vector3).
- Light source radius `light_bulb_r` (double, typically 0.1) [S1, unofficial].
- Scene geometry defined by a list of objects `objects` with types (e.g., `TYPE_SPHERE`) and optical properties [S1, unofficial].
- Room boundary constraints (e.g., $|x| \le 1.5, |y| \le 1.2, z \ge 0$) [S1, unofficial].
- Total photon count `nrPhotons` (int) [S1, unofficial].
- Maximum bounces `nrBounces` (int) [S1, unofficial].
- Reflection limit `reflection_limit` (int, typically 4) [S1, unofficial].
- Rendering mode flag `view3D` (bool) [S1, unofficial].

**Outputs:**
- Surface photon map: Array of photons stored at intersection points with position, direction, and color.
- Volume photon map (`vol_photons`): Array of photons stored in volume buffers for volumetric scattering or direct visualization [S1, unofficial].

**Data Structures:**
- `vol_photons`: A dynamic array or buffer storing volume photons. Each entry contains:
  - `vol_photons[i][0]`: Position (Vector3).
  - `vol_photons[i][1]`: Direction (Vector3).
  - `vol_photons[i][2]`: Energy/Color (Vector3).
- `num_vol`: Integer counter for the number of stored volume photons [S1, unofficial].
- `numPhotons[t]`: Integer array tracking photon counts per object `t` [S1, unofficial].

**Algorithm Steps:**

1. **Initialization:**
   - Reset photon counters: `numPhotons[t] = 0` for all objects, `num_vol = 0` [S1, unofficial].
   - Determine total photons to emit: `num_photon = nrPhotons * 3.0` if `view3D` is true, else `nrPhotons` [S1, unofficial].
   - Initialize random seed for reproducibility if required (e.g., `srand(0)`) [S1, unofficial].

2. **Optional Fixed Source Emission (Testing/Specific Objects):**
   - Emit a fixed number of photons (e.g., 100) from a specific object (e.g., `objects[5]`) [S1, unofficial].
   - For each photon:
     - Generate random direction `ray = randDir(1.0)` [S1, unofficial].
     - Calculate start position: `from = Light + stepout(ray, light_bulb_r) + epsilon` [S1, unofficial].
     - Store photon at `from` in volume buffer: `storePhoton_vol(from, ray, rgb)` [S1, unofficial].
     - Store photon at intersection with `objects[5]`: `storePhoton(objects[5], pnt, ray, rgb)` [S1, unofficial].

3. **Main Photon Emission Loop:**
   - Iterate `i` from 0 to `num_photon - 1`:
     - **Step 3.1: Photon Generation**
       - Initialize color `rgb = white` (1.0, 1.0, 1.0) [S1, unofficial].
       - Initialize bounce counter `bounces = 1` [S1, unofficial].
       - Generate random direction `ray = randDir(1.0)` [S1, unofficial].
       - Generate initial position `from`:
         - Start at `Light`.
         - If `from.y() >= Light.y()`, regenerate `from = randDir(1.0) + Light` until `from.y() < Light.y()` [S1, unofficial].
         - Alternatively, use `from = Light + stepout(ray, light_bulb_r + len)` where `len` is sampled from a distribution (e.g., `randlen()`) [S1, unofficial].
     - **Step 3.2: Validation**
       - Check if `from` is outside the room boundaries (e.g., $|x| > 1.5$ or $|y| > 1.2$). If so, mark invalid: `bounces = nrBounces + 1` [S1, unofficial].
       - Check if `from` is inside any object (e.g., sphere). For each object `ob`:
         - If `ob` is a sphere and `distance(from, center) < radius`, mark invalid: `bounces = nrBounces + 1` [S1, unofficial].
       - If invalid, `continue` to next photon [S1, unofficial].
     - **Step 3.3: Volume Storage (Initial)**
       - If valid, store the initial photon state in the volume buffer: `storePhoton_vol(from, ray, rgb)` [S1, unofficial].
     - **Step 3.4: Ray Tracing and Bouncing**
       - Trace ray from `from` with direction `ray` to find first intersection `istat` [S1, unofficial].
       - While `istat.dist < NOT_INTERSECTED` and `bounces <= nrBounces`:
         - Calculate intersection point `pnt = from + ray * istat.dist` [S1, unofficial].
         - Handle optical interactions (reflection/refraction) based on `istat.obj->getOptics()` up to `reflection_limit` [S1, unofficial].
         - Update color: `col = mulColor(rgb, istat.obj)` [S1, unofficial].
         - Attenuate color by bounces: `rgb = col * (1.0 / sqrt(bounces))` [S1, unofficial].
         - Store photon at intersection: `storePhoton(istat.obj, pnt, ray, rgb)` [S1, unofficial].
         - Trace shadow ray: `shadowPhoton(ray, pnt)` to determine visibility and store shadow photon if applicable [S1, unofficial].
         - Reflect/Refract ray: `ray = reflect(istat.obj, pnt, ray, from)` [S1, unofficial].
         - Update `from = pnt` [S1, unofficial].
         - Trace next intersection: `istat = raytrace(ray, from)` [S1, unofficial].
         - Increment `bounces` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def emitPhotons(Light, light_bulb_r, objects, nrPhotons, nrBounces, reflection_limit, view3D):
    # Initialize counters
    numPhotons = [0] * len(objects)
    num_vol = 0
    vol_photons = []
    
    # Determine total photons
    num_photon = nrPhotons * 3.0 if view3D else nrPhotons
    
    white = Vector3(1.0, 1.0, 1.0)
    
    # Optional fixed emission for testing
    for i in range(100):
        ray = randDir(1.0)
        rr = stepout(ray, light_bulb_r)
        pnt = rr + Light + Vector3(0.0001, 0.0001, 0.0001)
        rgb = Vector3(50, 50, 50) # Example color
        storePhoton(objects[5], pnt, ray, rgb)
        storePhoton_vol(pnt, ray, rgb)
        num_vol += 1
        numPhotons[5] += 1

    for i in range(num_photon):
        bounces = 1
        rgb = white
        ray = randDir(1.0)
        from_pos = Light
        
        # Randomize location to ensure below light source
        while from_pos.y() >= Light.y():
            from_pos = randDir(1.0) + Light
            
        # Validate room boundaries
        if abs(from_pos.x()) > 1.5 or abs(from_pos.y()) > 1.2:
            bounces = nrBounces + 1
            
        # Validate inside objects
        for dx in range(len(objects)):
            ob = objects[dx]
            if ob.getType() != TYPE_SPHERE:
                continue
            center = Vector3(ob.coords)
            radius = ob.coords[3]
            if distance(from_pos, center) < radius:
                bounces = nrBounces + 1
                break
        
        if bounces > nrBounces:
            continue
            
        # Store initial volume photon
        storePhoton_vol(from_pos, ray, rgb)
        num_vol += 1
        
        # Trace first intersection
        istat = raytrace(ray, from_pos)
        
        while istat.dist < NOT_INTERSECTED and bounces <= nrBounces:
            pnt = from_pos + ray * istat.dist
            
            # Handle optics (reflection/refraction)
            ref = 0
            while istat.obj.getOptics() != OPT_NONE and ref < reflection_limit:
                # Reflect or refract ray
                ray = reflect(istat.obj, pnt, ray, from_pos)
                ref += 1
                from_pos = pnt
                istat = raytrace(ray, from_pos)
                if istat.dist >= NOT_INTERSECTED:
                    break
                else:
                    pnt = from_pos + ray * istat.dist
            
            if istat.dist >= NOT_INTERSECTED:
                break
                
            # Update color and store
            col = mulColor(rgb, istat.obj)
            rgb = col * (1.0 / sqrt(bounces))
            
            storePhoton(istat.obj, pnt, ray, rgb)
            numPhotons[istat.obj.getIndex()] += 1
            
            # Shadow ray
            shadowPhoton(ray, pnt)
            
            # Prepare for next bounce
            ray = reflect(istat.obj, pnt, ray, from_pos)
            from_pos = pnt
            bounces += 1
            istat = raytrace(ray, from_pos)
            
    return vol_photons, numPhotons
```

**Parameters:**
- `nrBounces`: Maximum number of bounces. Value not explicitly defined in sources, but used as a threshold [S1, unofficial].
- `reflection_limit`: Maximum number of optical interactions per bounce, typically 4 [S1, unofficial].
- `light_bulb_r`: Light source radius, typically 0.1 [S1, unofficial].
- `collect_r`: Radius for photon collection during density estimation, typically 0.7 [S1, unofficial].
- `power`: Photon power scaling factor, typically 500 [S1, unofficial].
- `exposure`: Exposure factor for lighting calculation, value not specified [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Use a small epsilon (e.g., 1.0e-5) when checking intersection distances to avoid self-intersection artifacts [S1, unofficial].
- **Boundary Conditions:** Photons starting outside the room or inside objects are discarded by setting `bounces` to an invalid value [S1, unofficial].
- **Degenerate Input:** If `nrBounces` is 0, no photons are stored [G].

## Subtopics

### [Photon Mapping for Participating Media](5547-photon-mapping-for-participating-media.md)
Extends surface-based photon mapping to handle light transport within volumetric media by decoupling light transport simulation from final image synthesis. It is distinguished by specific handling of volumetric scattering events and a distinct radiance estimate formula for volume photons.

### [Beam Radiance Estimate](1283-beam-radiance-estimate.md)
Uses continuous beam radiance estimates instead of discrete photon packets for density estimation.

### [Photon Beam Tracing](5401-photon-beam-tracing.md)
Traces beams of photons rather than single rays to improve efficiency in participating media.

### [Progressive Photon Mapping](6672-progressive-photon-mapping.md)
Iteratively refines the photon map and radius to converge to the correct solution.

## Operating characteristics
- **Time Complexity:** $O(N \cdot B \cdot R)$, where $N$ is the number of photons, $B$ is the maximum bounces, and $R$ is the cost of ray-object intersection tests [S1, unofficial].
- **Memory Complexity:** $O(N + V)$, where $N$ is the number of surface photons and $V$ is the number of volume photons [S1, unofficial].
- **Measured Performance:** Specific performance metrics are not provided in the sources. The method uses a simple rejection sampling for photon generation, which can be inefficient if the acceptance rate is low [S1, unofficial].
- **Strengths:** Captures complex indirect lighting effects like caustics and subsurface scattering in non-homogeneous media [S1, unofficial]. Decouples light transport from rendering [S1, unofficial].
- **Limitations:** High memory usage for large photon counts. Density estimation requires careful tuning of the collection radius `collect_r` [S1, unofficial].
- **When to Use:** When accurate indirect lighting in participating media is required and pre-computation is acceptable [S1, unofficial].
- **When Not to Use:** For real-time applications or scenes with dynamic geometry where pre-computation is not feasible [G].

## Background
Photon Emission and Storage is part of the Volumetric Photon Mapping technique, an extension of the surface-based Photon Mapping algorithm by Henrik Wann Jensen [S1, unofficial]. The specific implementation described in the sources is from an unofficial repository by jacklv123, which adapts the technique for participating media by storing photons in volume buffers [S1, unofficial]. The method builds on the principles of global illumination by storing photon paths in a spatial data structure for later density estimation [S1, unofficial].

## Sources
[S1] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
