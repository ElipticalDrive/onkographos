# Nested Raymarching for Refraction

Also called: GLASSREFLECT, Glass Refraction via Nested Raymarching, Glass-like surface modeling

## Overview
Nested Raymarching for Refraction models a glass-like surface by refracting the ray direction upon entering the object, marching inside the object, and refracting again upon exiting [S1, unofficial]. It differs from [Ray Marching with Reflections](1963-ray-marching-with-reflections.md) by handling transmission through transparent volumes rather than surface reflection, and from [Approximate Refraction](3072-approximate-refraction.md) by performing explicit nested raymarching rather than approximating the exit point.

## How it works
**Inputs and Outputs:**
*   Input: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, signed distance field (SDF) function $S(\mathbf{p})$, refractive index ratio $\eta$ (ratio of incident medium to glass), and maximum march steps.
*   Output: Color at the final intersection point or background color if the ray escapes.

**Data Structures:**
*   Standard raymarching state: current position $\mathbf{p}$, current direction $\mathbf{d}$, accumulated color/transmittance.
*   No additional complex structures required beyond standard SDF evaluation.

**Steps:**
1.  **Initial March:** Perform standard raymarching along $\mathbf{d}$ using the SDF. Let the intersection point be $\mathbf{p}_{entry}$ and the surface normal be $\mathbf{n}_{entry} = \nabla S(\mathbf{p}_{entry})$.
2.  **Entry Refraction:** Compute the new ray direction $\mathbf{d}_{in}$ using the GLSL `refract` function:
    $$ \mathbf{d}_{in} = \text{refract}(\mathbf{d}, \mathbf{n}_{entry}, \eta) $$
    Here, $\mathbf{d}$ is the incident direction, $\mathbf{n}_{entry}$ is the normal, and $\eta$ is the ratio of refractive indices (e.g., air to glass) [S1, unofficial].
3.  **Nested March (Inside Glass):** Initialize a nested loop for marching inside the glass.
    *   Set current position $\mathbf{p}_{curr} = \mathbf{p}_{entry}$.
    *   Set current direction $\mathbf{d}_{curr} = \mathbf{d}_{in}$.
    *   **Crucial Adjustment:** Since the ray is now inside the object, the signed distance values returned by the SDF will be negative. To march forward, subtract the distance step from the position, or equivalently, treat the distance as negative when calculating step size [S1, unofficial].
    *   Loop:
        *   Evaluate $d = S(\mathbf{p}_{curr})$.
        *   If $d > 0$: The ray has exited the object. Break the nested loop.
        *   If $d$ is sufficiently small (surface hit): Update $\mathbf{p}_{curr}$ and continue.
        *   Step: $\mathbf{p}_{curr} \leftarrow \mathbf{p}_{curr} + \mathbf{d}_{curr} \cdot \max(|d|, \epsilon)$ [G]. Note: The source specifies subtracting distances because they become negative, implying the step logic must account for the sign of $d$ to move in the direction of $\mathbf{d}_{curr}$ [S1, unofficial].
4.  **Exit Refraction:** When the loop breaks because $d > 0$ (exit detected):
    *   The current position $\mathbf{p}_{exit}$ is near the exit surface.
    *   Compute the exit normal $\mathbf{n}_{exit} = \nabla S(\mathbf{p}_{exit})$.
    *   **Normal Negation:** The normal at the exit point points away from the object center (outward). For the refraction calculation from glass to air, the source specifies using the negated normal $-\mathbf{n}_{exit}$ [S1, unofficial].
    *   **Index Inversion:** Use the reciprocal of the entry refractive index, $1/\eta$, because the ray is moving from glass to air [S1, unofficial].
    *   Compute the final direction $\mathbf{d}_{out}$:
        $$ \mathbf{d}_{out} = \text{refract}(\mathbf{d}_{curr}, -\mathbf{n}_{exit}, 1/\eta) $$
5.  **Final March:** Continue raymarching along $\mathbf{d}_{out}$ in the external medium to find the final intersection or background.

**Pitfalls:**
*   **Normal Direction:** When exiting, the SDF normal points outward. It must be negated ($-\mathbf{n}$) for the refraction formula to correctly compute the transmission from inside to outside [S1, unofficial].
*   **Index Ratio:** The refractive index ratio must be inverted ($1/\eta$) upon exit, as the ray travels from the denser medium (glass) to the rarer medium (air) [S1, unofficial].
*   **Negative Distances:** Inside the object, SDF values are negative. Marching logic must handle this sign change correctly to avoid stepping backwards or oscillating [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nested_raymarch_refraction(ray_origin, ray_dir, sdf, eta, max_steps):
    # 1. Initial March
    pos = ray_origin
    dir = ray_dir
    
    for i in range(max_steps):
        dist = sdf(pos)
        if dist < 0.001: # Hit surface
            break
        pos += dir * dist
    else:
        return background_color # Ray escaped without hit

    # Intersection found
    normal_in = normalize(sdf_gradient(pos))
    
    # 2. Entry Refraction
    # eta is ratio of incident (air) to glass
    dir_in = refract(dir, normal_in, eta)
    
    # 3. Nested March (Inside Glass)
    pos = pos + dir_in * 0.001 # Offset to avoid self-intersection
    for j in range(max_steps):
        dist = sdf(pos)
        
        # Check for exit: distance becomes positive
        if dist > 0.0:
            break
            
        # March inside: distances are negative, so we subtract distance
        # or use absolute value for step size in direction of dir_in
        step_size = max(abs(dist), 0.001)
        pos += dir_in * step_size
        
    # If loop finished without break, ray is trapped or max steps reached
    if j == max_steps - 1:
        return background_color

    # 4. Exit Refraction
    normal_out = normalize(sdf_gradient(pos))
    
    # Normal points away from surface, negate it for internal-to-external refraction
    # Index ratio is inverted (glass to air)
    dir_out = refract(dir_in, -normal_out, 1.0 / eta)
    
    # 5. Final March
    pos = pos + dir_out * 0.001
    for k in range(max_steps):
        dist = sdf(pos)
        if dist < 0.001:
            break
        pos += dir_out * dist
    else:
        return background_color
        
    return compute_color(pos)
```

## Subtopics

### Two-Sided Refraction Beam Handling
A method to handle light passing through transparent objects by computing refraction beams at both front and back intersections simultaneously [S2]. It runs the refinement process once for both reflection and refraction, as the heuristic depends only on geometry [S2]. It computes the refraction beam at the first intersection and immediately computes the second refraction [S2]. To intersect rays from inside the objects, it uses a distance impostor: a cube map storing surface normals and distance from a reference point (typically the object's center) [S2]. This speeds up rendering but cannot render caustics inside the object [S2].

## Operating characteristics
*   **Complexity:** Adds a nested loop to the standard raymarching process, effectively doubling the computational cost for glass objects compared to opaque ones.
*   **Performance:** Slower than [Approximate Refraction](3072-approximate-refraction.md) due to explicit marching inside the volume.
*   **Accuracy:** Provides physically plausible refraction for simple glass shapes but does not model internal reflections or complex caustics within the glass volume [S1, unofficial].
*   **Limitations:** Cannot handle multiple internal bounces or complex internal structures. The "glass" is treated as a hollow shell defined by the SDF [S1, unofficial].

## Background
This method was described in the COS 426 Final Project Writeup on Raymarching with Signed Distance Fields [S1, unofficial]. It is presented as a "relatively simple glass-like effect" that is "the most tricky to get right" among reflection types [S1, unofficial]. The authors note that a more sophisticated approach modeling glass as a material that both reflects and refracts light was attempted but unsuccessful [S1, unofficial].

## Sources
[S1] PDF COS 426 Final Project Writeup Raymarching with Signed Distance Fields, https://jbaum98.github.io/cos426_final_project/writeup.pdf, fetched 2026-10-27 (unofficial)
[S2] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
