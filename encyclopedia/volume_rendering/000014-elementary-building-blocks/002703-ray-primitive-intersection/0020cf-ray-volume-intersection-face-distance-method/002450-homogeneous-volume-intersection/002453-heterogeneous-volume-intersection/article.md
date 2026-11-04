# Heterogeneous Volume Intersection

Also called: Majorant-based Ray Marching, Null Rejection Sampling, Delta Tracking

## Overview
Heterogeneous Volume Intersection computes the first interaction point of a ray within a participating medium with spatially varying density, or determines if the ray exits the medium without interacting. It differs from [Homogeneous Volume Intersection](3260-homogeneous-volume-intersection.md) by handling non-constant density fields using a constant majorant density for rejection sampling, allowing it to model complex, spatially varying media like clouds or smoke without requiring a uniform grid subdivision.

## How it works
**Inputs:**
- Ray $r(t) = o + t \cdot d$, where $o$ is the origin, $d$ is the direction, and $t \ge 0$.
- A heterogeneous volume object containing:
  - A boundary geometry (e.g., a bounding box or mesh) defining the spatial extent of the medium.
  - A density texture function $\rho(p)$ that returns the local density at point $p$.
  - A majorant density $\rho_{max}$, a constant value such that $\rho(p) \le \rho_{max}$ for all $p$ in the volume.
  - A material (optional, for scattering properties, though this article focuses on intersection).

**Outputs:**
- A hit record indicating the first interaction point $p$ and the distance $t$ along the ray, if an interaction occurs.
- A flag indicating whether the ray exited the volume without interaction.
- The transmittance from the entry point to the interaction point (or exit point), calculated as $T = e^{-\rho_{max} \cdot \Delta t}$, where $\Delta t$ is the distance traveled.

**Data Structures:**
The volume is represented by a class `HeterogeneousVolume` with the following fields:
- `Object *boundary_`: The enclosing geometry used to compute the initial ray-volume intersection.
- `double density_majorant_`: The constant upper bound on density ($\rho_{max}$).
- `Texture *density_texture_`: The spatial density function $\rho(p)$.
- `Material *material_`: The scattering material associated with the volume.

**Steps:**
1. **Ray-Volume Boundary Intersection:**
   Intersect the ray $r(t)$ with the boundary geometry (`boundary_`). This yields two intersection distances, $t_{in}$ and $t_{out}$, representing the entry and exit points of the ray through the volume.
   - If the ray does not intersect the boundary, return "no hit".
   - If the ray originates inside the volume, $t_{in}$ may be negative. Clamp $t_{in}$ to $0$ if the ray starts inside, or handle appropriately based on the specific boundary intersection logic. For standard ray marching from an outside origin, $t_{in} \ge 0$.
   - If $t_{in} \ge t_{out}$, the ray is tangent or misses; return "no hit".

2. **Initialization:**
   Set the current distance along the ray, $t_{current} = t_{in}$.
   Initialize the accumulated optical depth $\tau = 0$.

3. **Ray Marching Loop:**
   Loop indefinitely:
   a. **Sample Step Size:**
      Generate a random number $u \in (0, 1)$.
      Calculate the step size $\Delta t$ using the majorant density:
      $$ \Delta t = -\frac{\ln(u)}{\rho_{max}} $$
      [S1, unofficial]

   b. **Advance Ray:**
      Update the current distance:
      $$ t_{current} \leftarrow t_{current} + \Delta t $$

   c. **Check Exit:**
      If $t_{current} \ge t_{out}$, the ray has exited the volume without an interaction.
      - Calculate the transmittance from $t_{in}$ to $t_{out}$:
        $$ T = e^{-\rho_{max} \cdot (t_{out} - t_{in})} $$
      - Return an exit hit with position $p = r(t_{out})$, transmittance $T$, and a flag indicating transmission.

   d. **Evaluate Local Density:**
      Compute the point $p = r(t_{current})$.
      Evaluate the local density using the density texture:
      $$ \rho_{local} = \rho(p) $$
      [S1, unofficial]

   e. **Rejection Sampling:**
      Generate a new random number $v \in (0, 1)$.
      If $v < \frac{\rho_{local}}{\rho_{max}}$, an interaction is accepted.
      - Return a hit at $p$ with distance $t_{current}$.
      - Calculate the transmittance from $t_{in}$ to $t_{current}$:
        $$ T = e^{-\rho_{max} \cdot (t_{current} - t_{in})} $$
      - The interaction is not a transmission; it is a scattering event.

   f. **Otherwise:**
      If the rejection test fails, continue the loop from step 3a.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def heterogeneous_volume_intersection(ray, volume):
    # Step 1: Intersect with boundary
    hit_in, hit_out = volume.boundary_.intersect(ray)
    
    if hit_in is None or hit_out is None:
        return None  # No intersection with volume
    
    t_in = hit_in.t
    t_out = hit_out.t
    
    # Handle ray starting inside the volume
    if t_in < 0:
        t_in = 0
        
    if t_in >= t_out:
        return None  # Tangent or miss
        
    t_current = t_in
    rho_max = volume.density_majorant_
    
    # Step 3: Ray Marching Loop
    while True:
        # Step 3a: Sample step size
        u = random_double()
        dt = -math.log(u) / rho_max
        
        # Step 3b: Advance ray
        t_current += dt
        
        # Step 3c: Check exit
        if t_current >= t_out:
            # Ray exited without interaction
            transmittance = math.exp(-rho_max * (t_out - t_in))
            return HitRecord(
                point=ray.at(t_out),
                t=t_out,
                transmittance=transmittance,
                is_transmission=True
            )
        
        # Step 3d: Evaluate local density
        p = ray.at(t_current)
        tex_val = volume.density_texture_.value(p)
        # Assuming texture returns RGB, density is average of channels
        rho_local = rho_max * (tex_val.r + tex_val.g + tex_val.b) / 3.0
        
        # Step 3e: Rejection sampling
        v = random_double()
        if v < rho_local / rho_max:
            # Interaction accepted
            transmittance = math.exp(-rho_max * (t_current - t_in))
            return HitRecord(
                point=p,
                t=t_current,
                transmittance=transmittance,
                is_transmission=False
            )
```

**Parameters:**
- `density_majorant_`: Must be $\ge \max(\rho(p))$ for all $p$ in the volume. A tighter majorant improves efficiency. [S1, unofficial]
- `density_texture_`: A texture object implementing a `value(Point3)` method that returns the density at a point. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Precision:** If $\rho_{local}$ is very close to $\rho_{max}$, floating-point errors may affect the rejection test. [G]
- **Boundary Intersection:** The boundary intersection must be robust. If the ray originates exactly on the boundary, $t_{in}$ should be handled carefully to avoid infinite loops or missed interactions. [G]
- **Majorant Selection:** If $\rho_{max}$ is too large, the acceptance rate $\frac{\rho_{local}}{\rho_{max}}$ becomes low, leading to many rejected steps and poor performance. [S1, unofficial]

## Subtopics

### Heterogeneous Volume Representation
Represents a participating medium with spatially varying density using a texture, bounded by geometry, and using a majorant density for null rejection. It inherits from `Object` and includes fields for the boundary, majorant density, density texture, and material. The constructor takes these four parameters. This representation is essential for the intersection method to function, as it provides the necessary data for density evaluation and boundary intersection. [S1, unofficial]

## Operating characteristics
**Time Complexity:**
The expected number of steps is proportional to the optical thickness of the volume divided by the majorant density. In the worst case, if the majorant is much larger than the local density, the number of steps can be large. [S1, unofficial]

**Memory Complexity:**
$O(1)$ additional memory beyond the storage for the density texture and boundary geometry. [S1, unofficial]

**Measured Performance:**
Performance depends heavily on the choice of the majorant density. A tight majorant leads to fewer rejected steps and faster convergence. [S1, unofficial]

**Strengths:**
- Handles arbitrary density distributions without requiring a uniform grid.
- Simple to implement.
- Exact sampling (no discretization error in the density field). [S1, unofficial]

**Limitations:**
- Inefficient if the majorant is much larger than the average density.
- Requires a robust boundary intersection routine. [S1, unofficial]

**When to Use:**
- When the density field is complex and cannot be easily approximated by a uniform grid.
- When exact sampling is required. [S1, unofficial]

**When Not to Use:**
- When the density field is uniform (use [Homogeneous Volume Intersection](3260-homogeneous-volume-intersection.md)).
- When the majorant is significantly larger than the local density, leading to poor performance. [S1, unofficial]

## Background
The method is based on the principle of null rejection sampling, also known as delta tracking, originally proposed by Lathrop [S1, unofficial]. It is widely used in volumetric path tracing for rendering participating media with spatially varying density. The implementation described here is based on the code from the TakTikFR/Volumetric-Path-Tracing repository. [S1, unofficial]

## Sources
[S1] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
