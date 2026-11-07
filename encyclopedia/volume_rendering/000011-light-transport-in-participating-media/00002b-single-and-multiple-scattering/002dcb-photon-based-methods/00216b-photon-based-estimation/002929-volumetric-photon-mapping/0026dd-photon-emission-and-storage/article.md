# Photon Emission and Storage

Also called: Photon Mapping with Volume Storage, emitPhotons

## Overview
Photon Emission and Storage is the data generation phase of Volumetric Photon Mapping, responsible for tracing photons from light sources through participating media and surfaces to populate a spatial data structure with scattering events [S1, unofficial]. It solves the problem of approximating indirect illumination and inscattering by storing photon positions, directions, and energies [S1, unofficial]. It differs from its sibling [Beam Radiance Estimate](1283-beam-radiance-estimate.md) by storing discrete point samples rather than beam-like structures, and from [Photon Beam Tracing](5401-photon-beam-tracing.md) by using standard ray-object intersection for storage rather than beam tracing through volumes [G].

## How it works
**Inputs:**
- Light source position `Light` (Vector3) [S1, unofficial].
- Light source radius `light_bulb_r` (double, typically 0.1) [S1, unofficial].
- Scene objects `objects` (array of `CObj*`), each with type, geometry, color, and optics [S1, unofficial].
- Room boundaries for validity checks (e.g., $|x| \le 1.5, |y| \le 1.2$) [S1, unofficial].
- Maximum bounces `nrBounces` (integer) [S1, unofficial].
- Reflection limit `reflection_limit` (integer, typically 4) [S1, unofficial].
- Total photon count `nrPhotons` (integer) [S1, unofficial].
- Rendering mode flag `view3D` (boolean) [S1, unofficial].

**Outputs:**
- Volume photon buffer `vol_photons`: Array storing photons interacting with volumes. Each entry contains position, direction, and energy [S1, unofficial].
- Surface photon map: Spatial data structure storing photons at surface intersections [S1, unofficial].

**Data Structures:**
- `vol_photons`: A flat array or list. Each element stores: `pos` (Vector3), `dir` (Vector3), `energy` (Vector3) [S1, unofficial].
- `num_vol`: Integer counter for the number of volume photons stored [S1, unofficial].
- `numPhotons[t]`: Integer array tracking surface photons per object `t` [S1, unofficial].

**Parameters:**
- `power`: Scalar energy multiplier (e.g., 500) [S1, unofficial].
- `collect_r`: Radius for gathering photons (e.g., 0.7) [S1, unofficial].
- `exposure`: Exposure factor for weighting (value not specified in source, treated as parameter) [S1, unofficial].
- `NOT_INTERSECTED`: Threshold distance indicating no intersection [S1, unofficial].

**Steps:**

1. **Initialization:** Reset `num_vol = 0` and `numPhotons[t] = 0` for all objects `t` [S1, unofficial]. Determine total photons to emit: `num_photon = nrPhotons * 3.0` if `view3D` is true, else `nrPhotons` [S1, unofficial].
2. **Optional Test Emission:** Emit 100 photons from a specific object (e.g., `objects[5]`) with fixed properties for testing purposes [S1, unofficial]. This step is conditional and may be skipped in production [G].
3. **Main Photon Emission Loop:** Iterate `i` from 0 to `num_photon - 1` [S1, unofficial]. Initialize photon state: `rgb = (1.0, 1.0, 1.0)` (white), `bounces = 1`, `ray = randDir(1.0)` (random unit vector), `from = Light` [S1, unofficial].
4. **Randomize Start Position:** While `from.y() >= Light.y()`, set `from = randDir(1.0) + Light` [S1, unofficial]. This ensures the photon starts below the light source's Y-coordinate [G].
5. **Validity Checks:** If `fabs(from.x()) > 1.5` or `fabs(from.y()) > 1.2`, mark photon invalid by setting `bounces = nrBounces + 1` [S1, unofficial]. Iterate through all objects `dx`. If object `dx` is a sphere (`TYPE_SPHERE`), calculate `center` from `ob->coords`. If `distance(from, center) < ob->coords[3]` (radius), mark photon invalid by setting `bounces = nrBounces + 1` [S1, unofficial]. If `bounces > nrBounces`, skip to next photon [S1, unofficial].
6. **Volume Photon Storage (Initial):** If the photon is valid, call `storePhoton_vol(from, ray, rgb)` to store the initial photon state in the volume buffer [S1, unofficial]. Increment `num_vol` [S1, unofficial].
7. **Ray Tracing and Bounce Loop:** Perform ray trace: `istat = raytrace(ray, from)` [S1, unofficial]. While `istat.dist < NOT_INTERSECTED` and `bounces <= nrBounces`: Calculate intersection point `pnt = from + ray * istat.dist` [S1, unofficial]. Handle optics (reflection/refraction) based on `istat.obj->getOptics()`, up to `reflection_limit` interactions [S1, unofficial]. Update `ref` counter. Update `from = pnt`. Perform next ray trace: `istat = raytrace(ray, from)` [S1, unofficial]. If no intersection, break loop [S1, unofficial]. Calculate new `pnt` [S1, unofficial].
8. **Surface Photon Storage and Color Update:** If intersection exists: Calculate object color `col = mulColor(rgb, istat.obj)` [S1, unofficial]. Attenuate color by bounces `rgb = col * (1.0 / sqrt(bounces))` [S1, unofficial]. Store photon at surface `storePhoton(istat.obj, pnt, ray, rgb)` [S1, unofficial]. Increment `numPhotons[istat.obj->getIndex()]` [S1, unofficial]. Trace shadow ray `shadowPhoton(ray, pnt)` [S1, unofficial]. Reflect ray `ray = reflect(istat.obj, pnt, ray, from)` [S1, unofficial]. Perform next ray trace `istat = raytrace(ray, pnt)` [S1, unofficial]. Update `from = pnt` [S1, unofficial]. Increment `bounces` [S1, unofficial].
9. **Alternative Emission Mode (Light Bulb Extension):** Iterate `i` from 0 to `photon_num - 1` (e.g., 2000) [S1, unofficial]. Initialize `rgb = white`, `ray = randDir(1.0)` [S1, unofficial]. Calculate random length `len = randlen()` [S1, unofficial]. Calculate start position `from = Light + stepout(ray, light_bulb_r + len)` [S1, unofficial]. Check if `from` is inside the room using `cheak_in_room(from)`. If not, skip [S1, unofficial]. Check if `from` is inside any sphere object. If so, skip [S1, unofficial]. Store volume photon `storePhoton_vol(from, ray, rgb)` [S1, unofficial]. Increment `num_vol` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

# Global data structures
vol_photons = []
num_vol = 0
num_photons_per_obj = []

def rand_dir():
    """Generate a random unit vector using uniform distribution on sphere."""
    while True:
        x = random.uniform(-1, 1)
        y = random.uniform(-1, 1)
        z = random.uniform(-1, 1)
        length_sq = x*x + y*y + z*z
        if 0 < length_sq <= 1.0:
            length = math.sqrt(length_sq)
            return (x/length, y/length, z/length)

def rand_len():
    """Generate random length using rejection sampling as in source."""
    while True:
        len_val = random.randint(0, 299) / 100.0
        p = -2 * len_val + 6
        q = random.randint(0, 599) / 100.0
        if q <= p:
            return len_val

def step_out(ray, radius):
    """Step out from light source along ray by radius."""
    return (ray[0]*radius, ray[1]*radius, ray[2]*radius)

def check_in_room(p):
    """Check if point is within room boundaries."""
    return abs(p[0]) <= 1.5 and abs(p[1]) <= 1.2 and p[2] >= 0

def distance(p1, p2):
    return math.sqrt((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2)

def raytrace(ray, origin, objects):
    """Find closest intersection. Returns (obj, dist) or (None, inf)."""
    closest_dist = float('inf')
    closest_obj = None
    for obj in objects:
        dist = ray_object_intersection(obj, ray, origin)
        if dist > 1e-5 and dist < closest_dist:
            closest_dist = dist
            closest_obj = obj
    if closest_obj is None:
        return None, float('inf')
    return closest_obj, closest_dist

def ray_object_intersection(obj, ray, origin):
    """Compute ray-sphere intersection distance."""
    if obj.type == 'TYPE_SPHERE':
        center = obj.coords
        radius = obj.coords[3]
        oc = (origin[0]-center[0], origin[1]-center[1], origin[2]-center[2])
        a = ray[0]**2 + ray[1]**2 + ray[2]**2
        b = 2.0 * (oc[0]*ray[0] + oc[1]*ray[1] + oc[2]*ray[2])
        c = oc[0]**2 + oc[1]**2 + oc[2]**2 - radius**2
        discriminant = b**2 - 4*a*c
        if discriminant < 0:
            return float('inf')
        t1 = (-b - math.sqrt(discriminant)) / (2*a)
        t2 = (-b + math.sqrt(discriminant)) / (2*a)
        t = t1 if t1 > 1e-5 else (t2 if t2 > 1e-5 else float('inf'))
        return t
    return float('inf')

def store_photon_vol(position, direction, energy):
    """Store photon in volume buffer."""
    vol_photons.append({'pos': position, 'dir': direction, 'energy': energy})

def store_photon_surface(obj, position, direction, energy):
    """Store photon in surface map."""
    pass # Spatial data structure insertion

def mul_color(rgb, obj_color):
    """Multiply photon color by object color."""
    return (rgb[0]*obj_color[0], rgb[1]*obj_color[1], rgb[2]*obj_color[2])

def reflect_ray(obj, pnt, ray, from_pos):
    """Reflect ray off surface using standard formula R = D - 2(D.N)N."""
    center = obj.coords
    N = (pnt[0]-center[0], pnt[1]-center[1], pnt[2]-center[2])
    len_N = math.sqrt(N[0]**2 + N[1]**2 + N[2]**2)
    N = (N[0]/len_N, N[1]/len_N, N[2]/len_N)
    dot = ray[0]*N[0] + ray[1]*N[1] + ray[2]*N[2]
    R = (ray[0] - 2*dot*N[0], ray[1] - 2*dot*N[1], ray[2] - 2*dot*N[2])
    return R

def refract_ray(obj, pnt, ray):
    """Refract ray through surface (placeholder for standard Snell's law implementation)."""
    return ray # Placeholder: standard refraction logic omitted for brevity

def shadow_ray(ray, pnt):
    """Trace shadow ray to determine visibility."""
    # Implementation depends on scene
    pass

def emit_photons(objects, light_pos, nr_bounces, nr_photons, view3d):
    global num_vol
    num_vol = 0
    light_bulb_r = 0.1
    power = 500
    exposure = 1.0 # Assumed default
    
    # Reset object photon counts
    num_photons_per_obj = [0] * len(objects)
    
    num_photon = nr_photons * 3.0 if view3d else nr_photons
    
    # Optional test emission
    for i in range(100):
        rgb = (50, 50, 50)
        ray = rand_dir()
        rr = step_out(ray, light_bulb_r)
        pnt = (rr[0]+0.0001, rr[1]+0.0001, rr[2]+0.0001)
        store_photon_surface(objects[5], pnt, rr, rgb)
        
    # Main emission loop
    for i in range(int(num_photon)):
        bounces = 1
        rgb = (1.0, 1.0, 1.0)
        ray = rand_dir()
        from_pos = light_pos
        
        # Randomize start position
        while from_pos[1] >= light_pos[1]:
            from_pos = (rand_dir()[0] + light_pos[0], 
                        rand_dir()[1] + light_pos[1], 
                        rand_dir()[2] + light_pos[2])
            
        # Validity checks
        if abs(from_pos[0]) > 1.5 or abs(from_pos[1]) > 1.2:
            bounces = nr_bounces + 1
            
        for obj in objects:
            if obj.type != 'TYPE_SPHERE':
                continue
            center = obj.coords
            radius = obj.coords[3]
            if distance(from_pos, center) < radius:
                bounces = nr_bounces + 1
                
        if bounces > nr_bounces:
            continue
            
        # Store volume photon
        store_photon_vol(from_pos, ray, rgb)
        num_vol += 1
        
        # Ray tracing loop
        obj, dist = raytrace(ray, from_pos, objects)
        
        while obj is not None and dist < float('inf') and bounces <= nr_bounces:
            pnt = (from_pos[0] + ray[0]*dist, 
                   from_pos[1] + ray[1]*dist, 
                   from_pos[2] + ray[2]*dist)
            
            # Handle optics (reflection/refraction)
            # Limit interactions per bounce
            ref = 0
            while obj.optics != 'OPT_NONE' and ref < 4:
                if obj.optics == 'OPT_REFLECT':
                    ray = reflect_ray(obj, pnt, ray, from_pos)
                elif obj.optics == 'OPT_REFRACT':
                    ray = refract_ray(obj, pnt, ray)
                ref += 1
                from_pos = pnt
                obj, dist = raytrace(ray, from_pos, objects)
                if obj is None or dist >= float('inf'):
                    break
            
            if obj is None or dist >= float('inf'):
                break
                
            # Update color and store
            col = mul_color(rgb, obj.color)
            rgb = (col[0]/math.sqrt(bounces), 
                   col[1]/math.sqrt(bounces), 
                   col[2]/math.sqrt(bounces))
            
            store_photon_surface(obj, pnt, ray, rgb)
            num_photons_per_obj[obj.index] += 1
            
            # Shadow ray
            shadow_ray(ray, pnt)
            
            # Reflect for next bounce
            ray = reflect_ray(obj, pnt, ray, from_pos)
            obj, dist = raytrace(ray, pnt, objects)
            
            if obj is None or dist >= float('inf'):
                break
                
            from_pos = pnt
            bounces += 1

    # Alternative emission mode
    for i in range(2000):
        rgb = (1.0, 1.0, 1.0)
        ray = rand_dir()
        length = rand_len()
        step = step_out(ray, light_bulb_r + length)
        from_pos = (light_pos[0] + step[0], 
                    light_pos[1] + step[1], 
                    light_pos[2] + step[2])
        
        if not check_in_room(from_pos):
            continue
            
        inside_object = False
        for obj in objects:
            if obj.type != 'TYPE_SPHERE':
                continue
            center = obj.coords
            radius = obj.coords[3]
            if distance(from_pos, center) < radius:
                inside_object = True
                break
                
        if inside_object:
            continue
            
        store_photon_vol(from_pos, ray, rgb)
        num_vol += 1
```

## Subtopics

### Photon Mapping for Participating Media
A method for non-homogeneous materials extending photon mapping to participating media. It uses Monte Carlo simulation of photon transport from light sources, accounting for spatially varying optical properties and general phase functions. It records each scattering event in a spatial data structure called a photon map. In a final gather pass from the viewpoint, it collects scattering events relevant to a viewing ray to account for inscattered photons. The method can be slow because many photons are required for accurate convergence. It is less useful for volume rendering of spatially varying data compared to homogeneous materials. [S2, book]

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of photons emitted and the number of bounces per photon. The final gather step is linear with respect to the number of pixels and the number of photons in the search radius.
- **Memory Complexity:** Linear with respect to the number of photons stored.
- **Performance:** The method is computationally expensive due to the large number of photons required for accurate convergence, especially in participating media. [S2, book]
- **Strengths:** Can handle complex lighting scenarios, including indirect illumination and inscattering in participating media. [S2, book]
- **Limitations:** Slow convergence; requires many photons for accurate results. Less effective for spatially varying optical properties in volumes. [S2, book]

## Background
Photon mapping was introduced as a global illumination technique. The extension to participating media was developed by Jensen and Christiansen, who extended photon mapping to handle non-homogeneous materials and general phase functions. This approach simulates photon transport from light sources and stores scattering events in a photon map, which is then used in a final gather pass to compute inscattered light. [S2, book]

## Sources
[S1] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
[S2] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
