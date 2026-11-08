# Direct Haptic Rendering of Isosurface by Intermediate Representation

Also called: IR Approach, Intermediate Representation Approach for Haptic Rendering, Intermediate Virtual Plane Method, Intermediate representation approach, Virtual plane method

## Overview
This method provides direct haptic rendering of implicit isosurfaces within volumetric data using a point-based haptic device, eliminating the need to extract polygonal meshes [S1]. It employs a virtual plane as an intermediate geometric representation to decouple the visual rendering process from the high-frequency haptic control loop, thereby maintaining stability and high servo rates for complex volumetric data [S1, S3]. Unlike methods that approximate stiffness via density fields or require expensive gradient computations per loop, this approach uses a locally approximated tangential plane to compute interaction forces, offering a balance between computational efficiency and realistic contact sensation [S1, S3].

## How it works
### Inputs and Outputs
**Inputs:**
*   Volumetric data $V$ (scalar field).
*   Isovalue threshold $\tau$ defining the isosurface.
*   Haptic device parameters: Stiffness $K$, Damping $B$.
*   Update ratio $n$ (integer), determining how often the virtual plane is recomputed relative to the haptic servo loop.
*   Current Haptic Interface Point (HIP) position $P_{hip}$ from the device.
*   Previous Virtual Haptic Interface Point (VHIP) position $P_{vhip}$ and its associated Virtual Plane parameters.

**Outputs:**
*   Interaction Force vector $F$ applied to the haptic device.
*   Updated VHIP position $P_{vhip}$ and Virtual Plane parameters for the next loop.

### Data Structures
*   **Virtual Plane:** Defined by a point $p$ (intersection point on isosurface) and a normal vector $N$. Represented as a tuple $(N, p)$.
*   **State Variables:**
    *   $P_{hip}$: Current device position.
    *   $P_{vhip}$: Constrained point on the surface.
    *   $c$: Counter for virtual plane updates, initialized to 0.
    *   $V_{hip}$: Voxel value at $P_{hip}$.
    *   $V_{vhip}$: Voxel value at $P_{vhip}$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def haptic_control_loop(V, tau, K, B, n, P_hip, P_vhip, Virtual_Plane, c):
    """
    Single iteration of the haptic control loop.
    
    Args:
        V: Volumetric data array.
        tau: Isovalue threshold.
        K: Stiffness constant.
        B: Damping constant.
        n: Update ratio (virtual plane updates every n loops).
        P_hip: Current Haptic Interface Point position.
        P_vhip: Previous Virtual Haptic Interface Point position.
        Virtual_Plane: Tuple (N, p) from previous loop, or (0,0) if none.
        c: Current update counter.
        
    Returns:
        F: Interaction force vector.
        P_vhip: Updated VHIP position.
        Virtual_Plane: Updated Virtual Plane parameters.
        c: Updated counter.
    """
    
    # 1. Sample Voxel Values
    V_hip = tri_linear_interpolate(V, P_hip)
    V_vhip = tri_linear_interpolate(V, P_vhip)
    
    # 2. Binary Segmentation
    # B(v) = 0 if outside (background), 1 if inside (object)
    B_hip = 1 if V_hip > tau else 0
    B_vhip = 1 if V_vhip > tau else 0
    
    # 3. Determine Intersection Status
    # Case A: Both outside, both inside, or exiting (VHIP inside, HIP outside)
    # No force generated. VHIP moves to HIP.
    if B_hip == 0 and B_vhip == 0:
        P_vhip = P_hip
        Virtual_Plane = (0, 0) # Invalidate plane
        c = 0
        F = (0, 0, 0)
        return F, P_vhip, Virtual_Plane, c
        
    if B_hip == 1 and B_vhip == 1:
        P_vhip = P_hip
        Virtual_Plane = (0, 0) # Invalidate plane
        c = 0
        F = (0, 0, 0)
        return F, P_vhip, Virtual_Plane, c
        
    if B_vhip == 1 and B_hip == 0:
        # Exiting the object
        P_vhip = P_hip
        Virtual_Plane = (0, 0) # Invalidate plane
        c = 0
        F = (0, 0, 0)
        return F, P_vhip, Virtual_Plane, c
        
    # Case B: Penetration (VHIP outside, HIP inside)
    # Ray from P_vhip to P_hip intersects isosurface
    if B_vhip == 0 and B_hip == 1:
        # Check if we need to update the virtual plane
        c += 1
        if c >= n:
            c = 0
            # Recompute Virtual Plane
            # Linear interpolation along ray to find intersection point p where v = tau
            # Ray: R(t) = P_vhip + t * (P_hip - P_vhip)
            # We need t such that V(R(t)) = tau
            # Using linear interpolation of voxel values for simplicity as per source logic
            # Note: Source uses linear interpolation of position based on values
            # t = (tau - V_vhip) / (V_hip - V_vhip)
            t = (tau - V_vhip) / (V_hip - V_vhip)
            p = P_vhip + t * (P_hip - P_vhip)
            
            # Compute Normal N at p using central difference
            N = compute_surface_normal(V, p)
            
            Virtual_Plane = (N, p)
            
        # Get current plane parameters
        N, p = Virtual_Plane
        
        # 4. Constrain VHIP to Virtual Plane
        # Project HIP onto the plane defined by N and p
        # Distance from HIP to plane: d_plane = N . (HIP - p)
        # If HIP is on the "inside" side relative to normal, or if plane is invalid
        # The source logic: IF N.(HIP-p) < 0 THEN constrain.
        # This implies N points outward? Or inward?
        # Standard plane equation: N . (x - p) = 0.
        # If N.(HIP - p) < 0, HIP is on the negative side.
        # The VHIP is moved to the point on the plane minimizing distance to HIP.
        
        # Calculate projection
        # Vector from p to HIP
        vec = P_hip - p
        # Dot product
        dot = dot_product(N, vec)
        
        if dot < 0:
            # Project HIP onto plane
            # P_proj = HIP - (N . (HIP - p)) * N
            P_vhip = P_hip - dot * N
        else:
            # If HIP is on the positive side (or plane invalid/zero), 
            # and we are in penetration case, this branch might not be taken 
            # if the plane was just computed correctly for the intersection.
            # However, if the plane is old or invalid, handle gracefully.
            # If Virtual_Plane was (0,0), this block shouldn't execute 
            # because B_vhip would likely be 1 or logic would have reset.
            # Assuming valid plane here:
            P_vhip = P_hip - dot * N
            
        # 5. Compute Force
        # d is the displacement vector between HIP and VHIP
        d = P_hip - P_vhip
        
        # Velocity along d (v_d)
        # Approximate velocity as change in d or device velocity
        # Source: v_d is velocity along d. 
        # In discrete loop, v_d approx (d_current - d_prev) / dt or device velocity projected.
        # Let's assume device velocity v_dev is available or approximated.
        # For strict implementation from source: v_d is velocity along d.
        # If exact velocity not provided, use finite difference of d.
        # [G] Assuming a previous displacement d_prev is stored or velocity is derived from HIP movement.
        # Here we use the change in d as a proxy for velocity along the constraint normal if explicit velocity isn't passed.
        # However, standard impedance control uses device velocity. 
        # Source formula: F = Kd - Bv_d.
        # Let's assume v_d is the component of HIP velocity along the normal N.
        # v_d = dot(N, velocity_HIP)
        # If velocity_HIP is not an input, we approximate v_d from d change.
        # To be self-contained and safe, we note v_d requires velocity.
        # [G] We will assume a variable `v_hip` (velocity of HIP) is available from the device driver.
        # If not, v_d = dot(N, (P_hip - P_vhip_prev) / dt). 
        # For this pseudocode, we assume `v_hip` is an external input or computed from previous step.
        # Let's define v_d explicitly.
        
        # v_d is the velocity of the HIP projected onto the normal N
        # v_d = dot(N, v_hip)
        # [G] v_hip must be provided by the haptic device driver.
        
        F = K * d - B * dot_product(N, v_hip)
        
        return F, P_vhip, Virtual_Plane, c

def compute_surface_normal(V, p):
    """
    Computes surface normal at p using central difference approximation.
    """
    # [G] Assumes V is a 3D array and p is in voxel coordinates.
    # If p is in world coordinates, it must be converted to voxel coordinates first.
    # Central difference:
    # dx = (V(p + ex) - V(p - ex)) / 2
    # dy = (V(p + ey) - V(p - ey)) / 2
    # dz = (V(p + ez) - V(p - ez)) / 2
    
    # Handle boundaries by clamping or using one-sided differences
    # For simplicity, assume p is not on the boundary of the volume
    grad_x = (V[p.x + 1, p.y, p.z] - V[p.x - 1, p.y, p.z]) / 2.0
    grad_y = (V[p.x, p.y + 1, p.z] - V[p.x, p.y - 1, p.z]) / 2.0
    grad_z = (V[p.x, p.y, p.z + 1] - V[p.x, p.y, p.z - 1]) / 2.0
    
    N = (grad_x, grad_y, grad_z)
    # Normalize N
    norm = magnitude(N)
    if norm > 0:
        N = (N.x / norm, N.y / norm, N.z / norm)
    else:
        N = (0, 0, 0) # Fallback
    return N

def tri_linear_interpolate(V, p):
    """
    Performs tri-linear interpolation of voxel value at position p.
    """
    # [G] Standard tri-linear interpolation implementation.
    # 1. Floor p to get integer voxel coordinates (i, j, k).
    # 2. Get weights (fx, fy, fz) as fractional parts.
    # 3. Interpolate along x, y, z axes.
    # Returns scalar value.
    pass
```

### Parameters
*   **Isovalue ($\tau$):** The threshold value defining the isosurface. Typical values depend on the data (e.g., CT Hounsfield units for medical data) [S1].
*   **Stiffness ($K$):** Spring constant in the virtual spring-damper model. Determines the resistance to penetration [S1].
*   **Damping ($B$):** Damping coefficient. Stabilizes the simulation and reduces oscillations [S1].
*   **Update Ratio ($n$):** The ratio of the haptic control loop rate to the virtual plane update rate. The virtual plane is updated every $n$ control loops. Typical values are 5 to 10, balancing smoothness and performance [S1].

### Edge Cases and Pitfalls
*   **Multiple Solutions (Ambiguity):** In concave regions, the condition "surface normal equals HIP-to-VHIP vector" may have multiple solutions. The method uses an "inching" algorithm to move the VHIP incrementally along the surface from its previous position to select the correct, continuous VHIP [S1].
*   **Force Discontinuity:** If the virtual plane update rate is too slow, the VHIP may drop suddenly onto a new plane or become embedded in the surface, causing large force spikes that exceed haptic device bandwidth [S1].
    *   *Mitigation:* Increase the update rate (decrease $n$) or apply "force shading" (using a temporary constraint plane from the previous loop) and "recovery time" (gradually reducing force magnitude to move VHIP out of the surface) [S1].
*   **Embedded VHIP:** If the VHIP is calculated to be inside the surface, it is violently accelerated out. The recovery time method smooths this by reducing force magnitude over a period [S1].
*   **Noisy Data:** Detailed variations in noisy volumetric data (e.g., soft tissue) can cause unrealistic force feedback. Pre-processing with spatial low-pass filtering or morphological operations is recommended [S1].

## Operating characteristics
*   **Time Complexity:** The haptic control loop performs tri-linear interpolation and vector operations. The virtual plane update (every $n$ loops) adds gradient computation (central differences). The cost is significantly lower than methods requiring multiple gradient searches per loop (like Blezek and Robb) but higher than simple density-based methods (like Avila and Sobierajski) [S1].
*   **Memory Complexity:** Minimal additional memory beyond the volumetric data and a few state variables (VHIP, Plane, Counter) [S1].
*   **Measured Performance:** On a SGI Octane workstation with a 128x128x64 lung dataset, the method achieved an average servo time of 46.80 microseconds (21.37 kHz) [S1]. This is comparable to Avila and Sobierajski (21.65 kHz) and faster than Blezek and Robb (17.36 kHz) [S1].
*   **Strengths:**
    *   Avoids preprocessing and polygon extraction, allowing dynamic volumetric data changes [S1].
    *   Decouples visual and haptic rendering, improving stability [S1].
    *   The virtual plane acts as a local smoother for noisy data [S1].
    *   Simulates stiff contact (e.g., bone) more realistically than density-based approximations [S1].
*   **Limitations:**
    *   Slower than direct density-based methods due to gradient calculations [S1].
    *   Force discontinuities can occur in concave regions or at high curvature if update rates are insufficient or smoothing techniques are not applied [S1].
    *   Performance depends on the choice of $n$; too high causes "stepping" artifacts, too low reduces haptic rate [S1].

## Background
The method was proposed by researchers presenting at VRST 2000 in Seoul, Korea [S1]. It extends the "intermediate representation" approach, originally introduced for complex virtual environments, to the domain of haptic rendering of volumetric data [S1]. It addresses the limitations of two prior direct haptic rendering methods: Avila and Sobierajski's density-based approximation (which fails to simulate stiff structures like bone realistically) and Blezek and Robb's constraint-based method (which is computationally expensive due to multiple gradient searches per loop) [S1]. The implementation was tested using the PHANTOM haptic interface and the GHOST SDK [S1].

## Sources
[S1] Direct haptic rendering of isosurface by intermediate representation, https://doi.org/10.1145/502390.502426, fetched 2026-10-27
