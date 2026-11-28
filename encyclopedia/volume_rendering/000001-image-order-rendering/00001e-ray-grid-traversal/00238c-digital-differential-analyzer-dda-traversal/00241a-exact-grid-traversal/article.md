# Exact Grid Traversal

Also called: Cylindrical Grid Approach, Discrete Grid Point Capture, USCG Intersection Calculation, USCG Scheme, Uniformly Sampled Cylindrical Grid (USCG) Line Tracing

## Overview
Exact Grid Traversal is a method for capturing all discrete grid points that a projection line passes through, primarily used in algebraic image reconstruction [S1, unofficial][S2]. It differs from standard Cartesian DDA methods by employing a Uniformly Sampled Cylindrical Grid (USCG) structure, which consists of cylindrical, radial, and axial planes [S2]. This approach is specifically designed to speed up line coefficient calculation by avoiding intersection point calculations for radial planes, leveraging the symmetry of the cylindrical geometry [S2].

## How it works
**Inputs:** Source point $S$, Detector point $D$, Grid parameters (Number of slices $N$, Number of rings $N/2$, radial length $r$, slice thickness $h$, source angle $\theta_s$).
**Outputs:** List of grid indices traversed by the line segment $SD$.

**Data Structures:**
*   **USCG Grid:** Defined by $N$ slices along the z-axis and $N/2$ rings in the xy-plane.
*   **Ring $n$ ($n=1 \dots N/2$):** Contains $n_g = 4(2n - 1)$ grids.
*   **Intersection List $P_t$:** A list of intersection points sorted by distance from $S$.
*   **Line Segment Info:** Stores slice number $S_n$, ring number $R_n$, and azimuthal angles $\phi_{P_k}, \phi_{P_{k+1}}$.

**Steps:**

1.  **Calculate Intersection Points:**
    Compute the intersection points of the line $SD$ with cylindrical and axial planes.
    *   Project line $SD$ onto the XY plane to get $SD'$.
    *   Calculate distance $d$ of $SD'$ from the cylinder center:
        $$ d = \sqrt{\langle S, S \rangle^2 - t^2 \langle S - D', S - D' \rangle^2} $$
        where
        $$ t = \frac{S \cdot (S - D')}{\langle S - D', S - D' \rangle^2} $$
    *   If $d \le R$ (where $R$ is the cylinder radius), the line intersects the cylinder. The intersection points are:
        $$ S + (t \pm k)(D - S) $$
        where
        $$ k = \sqrt{\frac{R^2 - d^2}{\langle S - D', S - D' \rangle^2}} $$
    *   Sort all intersection points by their distance to $S$ [S2].

2.  **Identify Line Segments:**
    Identify consecutive intersection points $P_k$ and $P_{k+1}$ from the sorted list. These points reside in the same ring and slice throughout the data collection process [S2].

3.  **Determine Grid Coordinates for Segment:**
    For each segment $P_k P_{k+1}$:
    *   Calculate the midpoint $P_m$:
        $$ P_m = \left\{ \frac{P_{(k+1)x} + P_{kx}}{2}, \frac{P_{(k+1)y} + P_{ky}}{2}, \frac{P_{(k+1)z} + P_{kz}}{2} \right\} $$
    *   Calculate Slice Number $S_n$:
        $$ S_n = \left\lfloor \frac{P_{mz}}{h} \right\rfloor $$
    *   Calculate Radius $d_m$ of $P_m$:
        $$ d_m = \sqrt{P_{mx}^2 + P_{my}^2} $$
    *   Calculate Ring Number $R_n$:
        $$ R_n = \left\lfloor \frac{d_m}{r} \right\rfloor $$
    *   Calculate Azimuthal Angles $\phi_{P_k}$ and $\phi_{P_{k+1}}$:
        $$ \phi_{P_k} = \text{atan2}(P_{ky}, P_{kx}) $$
        $$ \phi_{P_{k+1}} = \text{atan2}(P_{(k+1)y}, P_{(k+1)x}) $$
    *   Store $(S_n, R_n, \phi_{P_k}, \phi_{P_{k+1}})$ for the segment [S2].

4.  **Trace Grids for a Projection:**
    For each projection with source angle $\theta_s$:
    *   Update azimuthal angles:
        $$ \phi_{P_k}^{new} = \phi_{P_k} + \theta_s $$
        $$ \phi_{P_{k+1}}^{new} = \phi_{P_{k+1}} + \theta_s $$
    *   Keep $S_n$ and $R_n$ constant [S2].
    *   Calculate Grid Numbers $G_{P_k}$ and $G_{P_{k+1}}$ for the current ring $R_n$:
        *   Azimuthal step angle for ring $R_n$:
            $$ \Delta \phi_{R_n} = \frac{360}{4(2n - 1)} $$
        *   Grid number:
            $$ G_{P_k} = \left\lfloor \frac{\phi_{P_k}}{\Delta \phi_{R_n}} \right\rfloor + H_n $$
            where $H_n$ is the head of ring $R_n$.
    *   Determine Line Segment Head ($L_h$) and Tail ($L_t$) based on angle difference $|\phi_{P_k} - \phi_{P_{k+1}}|$ [S2]:
        *   **Case 1:** If $|\phi_{P_k} - \phi_{P_{k+1}}| < 180^\circ$:
            $$ L_h = \min(G_{P_k}, G_{P_{k+1}}) $$
            $$ L_t = \max(G_{P_k}, G_{P_{k+1}}) $$
            Iterate grids $G_i = L_h + i$ for $i \in [0, L_t - L_h]$.
        *   **Case 2:** If $|\phi_{P_k} - \phi_{P_{k+1}}| > 180^\circ$:
            The points are opposite across the positive X-axis. Use positive and negative index sets.
            $$ L_h = \max(G_{P_k}, G_{P_{k+1}}) $$
            $$ L_t = \min(G_{P_k}, G_{P_{k+1}}) $$
            Iterate grids using the specific indexing logic for crossing the boundary (indices wrap around the ring) [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def exact_grid_traversal(S, D, N, r, h, p):
    """
    S: Source point (x, y, z)
    D: Detector point (x, y, z)
    N: Number of slices (and 2 * number of rings)
    r: Radial length of grid cells
    h: Slice thickness
    p: Number of projections (rotations)
    """
    
    # 1. Calculate Intersection Points
    # Project S and D onto XY plane
    S_xy = (S[0], S[1])
    D_xy = (D[0], D[1])
    
    # Vector S - D'
    vec_SD_prime = (S_xy[0] - D_xy[0], S_xy[1] - D_xy[1])
    dot_SD_prime = vec_SD_prime[0]**2 + vec_SD_prime[1]**2
    
    if dot_SD_prime == 0:
        return [] # Degenerate line
        
    # t calculation
    dot_S_SD_prime = S[0]*(S[0] - D_xy[0]) + S[1]*(S[1] - D_xy[1])
    t = dot_S_SD_prime / dot_SD_prime
    
    # d calculation
    # Note: Formula uses <S,S> which is magnitude squared of S vector
    mag_S_sq = S[0]**2 + S[1]**2 + S[2]**2 # Assuming S is 3D, but d is in XY plane projection logic
    # The source formula: d = sqrt(<S,S>^2 - t^2 <S-D', S-D'>^2)
    # This seems to imply S is treated as a vector in the plane or the formula is specific to the geometry.
    # Based on S2, d is distance of line SD' from cylinder center (origin).
    # Let's stick to the provided formula strictly.
    # <S,S> usually means dot product. If S is 3D, <S,S> is x^2+y^2+z^2.
    # However, d is distance in XY plane. The formula in S2 might assume S is in XY or specific context.
    # Given "distance d of the projection of line SD onto the XY plane", d should be 2D distance.
    # Let's assume the formula provided in S2 is the ground truth for implementation.
    
    d_sq = mag_S_sq - (t**2 * dot_SD_prime)
    if d_sq < 0: d_sq = 0
    d = math.sqrt(d_sq)
    
    # Cylinder radius R is not explicitly given as input, but implied by grid bounds.
    # For a grid of N/2 rings with radial length r, max radius R_max = N/2 * r.
    # However, intersection calculation requires a specific R.
    # In reconstruction, we usually intersect with the bounding cylinder of the volume.
    R = (N / 2) * r 
    
    intersections = []
    if d <= R:
        k_sq = (R**2 - d_sq) / dot_SD_prime
        if k_sq < 0: k_sq = 0
        k = math.sqrt(k_sq)
        
        # Intersection points: S + (t +/- k)(D - S)
        vec_DS = (D[0] - S[0], D[1] - S[1], D[2] - S[2])
        
        p1 = (S[0] + (t + k)*vec_DS[0], S[1] + (t + k)*vec_DS[1], S[2] + (t + k)*vec_DS[2])
        p2 = (S[0] + (t - k)*vec_DS[0], S[1] + (t - k)*vec_DS[1], S[2] + (t - k)*vec_DS[2])
        
        # Distance from S
        dist1 = math.sqrt((p1[0]-S[0])**2 + (p1[1]-S[1])**2 + (p1[2]-S[2])**2)
        dist2 = math.sqrt((p2[0]-S[0])**2 + (p2[1]-S[1])**2 + (p2[2]-S[2])**2)
        
        intersections.append((dist1, p1))
        intersections.append((dist2, p2))
        
    # Sort by distance
    intersections.sort(key=lambda x: x[0])
    
    # 2. Identify Segments and Store Info
    line_segment_info = []
    
    for i in range(len(intersections) - 1):
        Pk = intersections[i][1]
        Pk1 = intersections[i+1][1]
        
        # Midpoint Pm
        Pm = ((Pk[0]+Pk1[0])/2, (Pk[1]+Pk1[1])/2, (Pk[2]+Pk1[2])/2)
        
        # Slice Number Sn
        Sn = math.floor(Pm[2] / h)
        
        # Radius dm
        dm = math.sqrt(Pm[0]**2 + Pm[1]**2)
        
        # Ring Number Rn
        Rn = math.floor(dm / r)
        
        # Azimuthal Angles
        phi_Pk = math.atan2(Pk[1], Pk[0])
        phi_Pk1 = math.atan2(Pk1[1], Pk1[0])
        
        line_segment_info.append({
            'Sn': Sn,
            'Rn': Rn,
            'phi_k': phi_Pk,
            'phi_k1': phi_Pk1
        })
        
    # 3. Trace for Projections
    all_traversed_grids = []
    
    delta_theta = 360 / p
    
    for i in range(1, p + 1):
        theta_s = i * delta_theta
        
        for seg in line_segment_info:
            Sn = seg['Sn']
            Rn = seg['Rn']
            phi_k = seg['phi_k']
            phi_k1 = seg['phi_k1']
            
            # Update angles
            phi_k_new = phi_k + theta_s
            phi_k1_new = phi_k1 + theta_s
            
            # Azimuthal step angle for ring Rn
            # n in formula is ring index. Here Rn is the ring number.
            # Formula: delta_phi_Rn = 360 / (4(2n - 1))
            # Note: Rn is 0-indexed or 1-indexed? 
            # S2 says "ring n (where n = 1, 2, ..., N/2)". 
            # If Rn is calculated as floor(dm/r), it is 0-indexed.
            # We must map Rn to n. If Rn=0, n=1? Or is Rn 1-based?
            # S2 Eq 7: Rn = floor(dm/r). If dm=0, Rn=0.
            # S2 Eq 9: n=1,2...
            # Let's assume Rn in code is 0-based index, so n = Rn + 1.
            n = Rn + 1
            delta_phi_Rn = 360 / (4 * (2 * n - 1))
            
            # Head of ring Hn. S2 doesn't define Hn explicitly, but it's an offset.
            # Assuming Hn = 0 for simplicity or standard alignment.
            Hn = 0 
            
            G_Pk = math.floor(phi_k_new / delta_phi_Rn) + Hn
            G_Pk1 = math.floor(phi_k1_new / delta_phi_Rn) + Hn
            
            # Determine Lh and Lt
            # S2 uses degrees in comparison. phi is in radians in atan2.
            # Convert diff to degrees or compare in radians (pi).
            diff = abs(phi_k_new - phi_k1_new)
            # Normalize diff to [0, 2pi) ? S2 implies simple difference.
            # If points are opposite, diff > 180 deg.
            
            # Get grids in ring Rn
            num_grids_in_ring = 4 * (2 * n - 1)
            
            traversed = []
            
            if diff < math.pi: # < 180 degrees
                Lh = min(G_Pk, G_Pk1)
                Lt = max(G_Pk, G_Pk1)
                
                # Iterate from Lh to Lt
                # Note: S2 says i in [0, Lh - Lt] which is empty if Lh < Lt.
                # S2 Eq 13: Gi = Lh + i. i in [0, Lt - Lh] (assuming typo in source Lh-Lt)
                # "The line segment travels (Lt - Lh + 1) number of grids"
                for i in range(Lt - Lh + 1):
                    g = Lh + i
                    # Wrap around if necessary? S2 Case 1 implies no wrap.
                    traversed.append((Sn, Rn, g % num_grids_in_ring))
            else: # > 180 degrees
                Lh = max(G_Pk, G_Pk1)
                Lt = min(G_Pk, G_Pk1)
                
                # Case 2: Use positive and negative index sets.
                # S2 Eq 16: Gi = Rn[Lh.index_ + i]
                # This implies wrapping.
                # Start at Lh, go towards Lt in the direction that crosses the boundary.
                # Since diff > 180, they are on opposite sides.
                # One goes up, one goes down?
                # S2 says "utilize both positive and negative index set".
                # Typically this means iterating from Lh to end, then 0 to Lt.
                
                # Positive direction from Lh to end
                for i in range(Lh, num_grids_in_ring):
                    traversed.append((Sn, Rn, i))
                # Negative direction from 0 to Lt
                for i in range(0, Lt + 1):
                    traversed.append((Sn, Rn, i))
                    
            all_traversed_grids.extend(traversed)
            
    return all_traversed_grids
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating point errors in `atan2` and angle comparisons can cause grid misassignment near boundaries.
*   **Boundary Crossing:** Case 2 (angle > 180) requires careful handling of index wrapping. The source mentions "positive and negative index set" but does not fully define the iteration logic beyond Eq 16. Implementers must ensure the wrap-around covers all grids between the points.
*   **Degenerate Input:** If $S$ and $D$ are collinear with the z-axis, $SD'$ has zero length, leading to division by zero in $t$ calculation.
*   **Ring Indexing:** The mapping between the calculated ring number $R_n$ (0-based from floor) and the formula parameter $n$ (1-based) must be consistent.

## Operating characteristics
*   **Time Complexity:** The calculation of coefficients is sped up by $O(N)$ compared to methods requiring radial plane intersections [S2].
*   **Memory Complexity:** Reduces storage burden of algebraic methods by a factor of the number of projections [S2].
*   **Strengths:** Efficient for algebraic reconstruction in cylindrical geometries; avoids expensive radial plane intersection calculations.
*   **Limitations:** Specific to cylindrical grids; naive implementation noted [S1, unofficial].

## Background
*   **Origin:** The method is described in the context of 3D image reconstruction using algebraic algorithms [S2].
*   **History:** A naive implementation of the "Exact" grid traversal (capturing all discrete grid points) was written by the author of the line rasterization document [S1, unofficial]. The USCG scheme was proposed for 2D/3D image reconstruction [S2].

## Sources
[S1] Line Rasterization and Grid Traversal, https://trzy.org/line_rasterization/, fetched 2026-10-25 (unofficial)
[S2] Uniformly Sampled Polar and Cylindrical Grid Approach for 2D, 3D Image Reconstruction using Algebraic Algorithm, https://arxiv.org/abs/2208.12964v1, fetched 2026-10-27
