# Ray Marching with Refraction and Beer's Law Absorption

Also called: renderRefraction, transparent raymarching

## Overview
This method extends distance-field ray marching to simulate transparent dielectric objects by handling refraction, total internal reflection (TIR), and exponential light absorption (Beer's Law) over multiple bounces [S1, unofficial]. Unlike standard ray marching which terminates at the first surface hit, this approach continues the ray through transparent boundaries, flipping the normal direction to track interior traversal and accumulating distance for attenuation [S1, unofficial]. It distinguishes itself from [Approximate Refraction](3072-approximate-refraction.md) by explicitly computing the refracted ray direction and handling TIR, and from [Cone Marching](3808-cone-marching.md) by using a single ray path rather than volumetric cone integration for absorption [S1, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{ro}$, ray direction $\mathbf{rd}$, scene signed distance function (SDF) `sceneSDF`, maximum refraction bounces `MAX_REFRACTIONS`, and per-material absorption vector $\mathbf{a}$ (RGB).
**Outputs:** Final pixel color $C_{out}$.

**Data Structures:**
*   `SdResult`: Struct containing `dist` (float, signed distance), `mat` (material struct), and potentially other metadata. `dist < 0` indicates inside the object.
*   `Material`: Contains `refraction_index` (float, IOR) and `absorption` (vec3).

**Parameters:**
*   `MAX_REFRACTIONS`: Integer, typically 4 [S1, unofficial].
*   `absorption`: RGB vector controlling exponential light attenuation per unit distance inside the medium [S1, unofficial].

**Algorithm:**
1.  Initialize `color = vec3(0)`, `invert = -1.0` (indicating outside initially, but logic flips based on hit sign), `absorb_dist = 0.0`.
2.  Loop `i` from 0 to `MAX_REFRACTIONS - 1`:
    a.  Cast ray: If `invert < 0`, call `castRayDI(ro, rd)` (interior cast); else call `castRayD(ro, rd)` (exterior cast). Let result be `sd`.
    b.  If `invert < 0`, add `sd.dist` to `absorb_dist`.
    c.  If `sd.dist < 0`:
        i.  If `invert > 0`, render background and break loop.
        ii. Else, continue to next iteration (ray exited object, `invert` will flip).
    d.  Compute intersection point: $\mathbf{p} = \mathbf{ro} + \mathbf{rd} \cdot sd.dist$.
    e.  Compute normal: $\mathbf{n} = \text{getNormalFast}(\mathbf{p}) \cdot \text{invert}$.
    f.  Compute reflection direction: $\mathbf{ref} = \text{reflect}(\mathbf{rd}, \mathbf{n})$.
    g.  Add lighting contribution: `color += light(sd.mat, ro, ref, p, n)`.
    h.  If `invert > 0`, break loop (ray exited object).
    i.  Compute Index of Refraction (IOR):
        i.  If `invert < 0` (inside), `ior = sd.mat.refraction_index`.
        ii. If `invert > 0` (outside), `ior = 1.0 / sd.mat.refraction_index`.
    j.  Compute refraction direction: $\mathbf{raf} = \text{refract}(\mathbf{rd}, \mathbf{n}, \text{ior})$.
    k.  If $\mathbf{raf} == \mathbf{0}$ (Total Internal Reflection), set $\mathbf{rd} = \mathbf{ref}$. Else, set $\mathbf{rd} = \mathbf{raf}$.
    l.  Update ray origin to prevent self-intersection: $\mathbf{ro} = \mathbf{p} + \mathbf{rd} \cdot (0.01 / |\mathbf{rd} \cdot \mathbf{n}|)$.
    m.  Update `invert`: If TIR occurred (`tif` is true), keep `invert`; else `invert = invert * -1.0`.
3.  Apply Beer's Law absorption: $C_{out} = \text{color} \cdot e^{-\mathbf{a} \cdot \text{absorb\_dist}}$.

**Formulas:**
*   Beer's Law Absorption:
    $$C_{out} = C_{in} \cdot e^{-\mathbf{a} \cdot d}$$
    where $\mathbf{a}$ is the absorption vector and $d$ is the accumulated distance traveled inside transparent media [S1, unofficial].
*   Self-intersection offset:
    $$\mathbf{ro}_{new} = \mathbf{p} + \mathbf{rd} \cdot \frac{0.01}{|\mathbf{rd} \cdot \mathbf{n}|}$$
    This offset prevents floating-point inaccuracies at sharp angles from causing immediate re-intersection [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_refraction(ro, rd, max_refractions=4):
    color = vec3(0.0)
    invert = -1.0  # -1 for outside, 1 for inside logic adjustment
    absorb_dist = 0.0
    
    for i in range(max_refractions):
        # Cast ray based on current state
        if invert < 0:
            sd = castRayDI(ro, rd)
        else:
            sd = castRayD(ro, rd)
            
        # Accumulate distance if inside
        if invert < 0:
            absorb_dist += sd.dist
            
        # Handle exit or background
        if sd.dist < 0:
            if invert > 0:
                return background(ro, rd)
            else:
                # Exited object, loop continues, invert flips next
                pass
        
        # Intersection point
        p = ro + rd * sd.dist
        
        # Normal with correct orientation
        n = getNormalFast(p) * invert
        
        # Reflection component
        ref = reflect(rd, n)
        color += light(sd.mat, ro, ref, p, n)
        
        # If we hit from outside (invert > 0 means we just entered or are outside?),
        # The source logic: "If invert>0, break" after lighting. 
        # This implies if we are outside and hit, we stop refracting further?
        # Actually, standard transparent raymarching:
        # invert tracks if we are currently "inside" the material for absorption.
        # Let's follow the source strictly:
        if invert > 0:
            break
            
        # Compute IOR for refraction
        if invert < 0:
            ior = sd.mat.refraction_index
        else:
            ior = 1.0 / sd.mat.refraction_index
            
        # Compute refraction
        raf = refract(rd, n, ior)
        
        # Check for Total Internal Reflection (TIR)
        tif = (raf == vec3(0.0))
        
        if tif:
            rd = ref
        else:
            rd = raf
            
        # Offset origin to prevent self-intersection
        ro = p + rd * (0.01 / abs(dot(rd, n)))
        
        # Update invert state
        if tif:
            pass # Keep invert
        else:
            invert = invert * -1.0
            
    # Apply Beer's Law
    absorption = sd.mat.absorption # Assuming material has absorption
    color *= exp(-dot(absorption, absorb_dist))
    
    return color
```
*[G] The pseudocode assumes `castRayDI` and `castRayD` are distinct SDF casters handling interior/exterior distance queries appropriately, and `refract` returns zero vector on TIR. The source logic for `invert` flipping is specific to the implementation's state machine for tracking interior distance.*

## Subtopics

### [Transmission & Chromatic Dispersion Ray Marching](3718-transmission-chromatic-dispersion-ray-marching.md)
This method extends [Ray Marching with Refraction and Beer's Law Absorption](1947-ray-marching-with-refraction-and-beer-s-law-absorption.md) by simulating chromatic dispersion, where the index of refraction (IOR) varies by wavelength. It splits the ray into Red, Green, and Blue channels, refracting each with a distinct IOR to produce spectral separation effects, while falling back to a single IOR when dispersion is negligible.

## Operating characteristics
*   **Time Complexity:** Linear with respect to `MAX_REFRACTIONS` and the cost of SDF evaluation per bounce. Each bounce requires a full ray cast and normal computation.
*   **Memory Complexity:** Constant, $O(1)$, as it only stores current ray state and accumulated color/distance.
*   **Strengths:** Physically plausible transparency, handles complex internal reflections and absorption.
*   **Limitations:** Computationally expensive due to multiple bounces. Sensitive to floating-point precision at grazing angles (mitigated by offset). Does not account for subsurface scattering or volumetric scattering within the medium, only surface refraction and path absorption.
*   **When to use:** Rendering transparent dielectrics like glass, water, or gems where internal reflections and color absorption are important.
*   **When not to use:** Opaque objects, or when performance is critical and approximate transparency suffices.

## Background
This method is implemented in the `cahekp/Raymarching` repository [S1, unofficial]. It builds upon standard ray marching techniques by adding transparency handling. The use of Beer's Law for absorption is a standard technique in computer graphics for simulating light attenuation in participating media or transparent solids.

## Sources
[S1] cahekp/Raymarching, https://github.com/cahekp/Raymarching, fetched 2026-10-27 (unofficial)
[S2] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
