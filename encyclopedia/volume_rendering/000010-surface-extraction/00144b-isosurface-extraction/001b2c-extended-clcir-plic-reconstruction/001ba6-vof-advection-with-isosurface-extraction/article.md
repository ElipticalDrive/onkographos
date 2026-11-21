# VOF Advection with Isosurface Extraction

Also called: EMFPA-CLCIR, EMFPA-Youngs, VOF simulation, Volume of Fluid (VOF) Advection with Isosurface Extraction

## Overview
This method simulates multiphase fluid flows by advecting a Volume of Fluid (VOF) indicator function on arbitrary polyhedral grids [S1]. It solves the advection equation geometrically using Edge-Matched Flux Polygon Advection (EMFPA) and reconstructs the fluid interface using the CLCIR method, which relies on isosurface extraction to determine the Piecewise Linear Interface Concept (PLIC) plane orientation [S1]. It is distinguished from standard VOF methods by its ability to handle non-convex and self-intersecting polyhedral cells without decomposition, and from pure isosurface methods by its explicit geometric flux calculation [S1].

## How it works
**Inputs:**
*   Computational grid $\Omega$ consisting of arbitrary polyhedral cells (cubic, tetrahedral, distorted, or non-convex irregular) [S1].
*   Velocity field $\mathbf{v} = (u, v, w)$ defined at cell faces [S1].
*   Initial fluid volume fraction field $F(t^n)$ for each cell [S1].
*   Time step $\Delta t$ [S1].

**Outputs:**
*   Updated fluid volume fraction field $F(t^{n+1})$ [S1].
*   Reconstructed PLIC interface geometry for cells with $10^{-12} < F(t^n) < 1 - 10^{-12}$ [S1].

**Data Structures:**
*   **Grid:** A collection of cells $\Omega_i$, each defined by a set of vertices and faces [S1]. Faces may be non-planar or self-intersecting [S1].
*   **Volume Fraction:** A scalar field $F$ where $F_i \in [0, 1]$ represents the fraction of cell $i$ occupied by the fluid [S1].
*   **Velocity:** Vector components $u_j, v_j, w_j$ stored at the center of each face $j$ [S1].

**Step 1: Initialization**
At $t^n = 0$, compute the initial volume fraction $F(t^n=0)$ for each cell [S1]. For a spherical fluid body of radius $0.15$ centered at $(0.5, 0.75, 0.25)$ in a unit box, use a grid refinement procedure extended to non-convex cells [S1].

**Step 2: Time Step Calculation**
Determine the time step $\Delta t$ using the CFL condition:
$$ \Delta t = \min \left[ \frac{\min(h_{xi})}{\max(u_j)}, \frac{\min(h_{yi})}{\max(v_j)}, \frac{\min(h_{zi})}{\max(w_j)} \right] \times \text{CFL} $$
where:
*   $\text{CFL}$ is the Courant number (typically $0.5$) [S1].
*   $h_{xi}, h_{yi}, h_{zi}$ are the dimensions of the minimum-size rectangular parallelepiped enclosing cell $i$ along the coordinate axes [S1].
*   For cubic grids with $n$ cells, $\min(h_{xi}) = \min(h_{yi}) = \min(h_{zi}) = 1/n^{1/3}$ [S1].
*   For unstructured grids, the min/max operators consider only cells and faces near the interface [S1].

**Step 3: Interface Reconstruction (CLCIR)**
For each cell $i$ where $10^{-12} < F_i(t^n) < 1 - 10^{-12}$:
1.  Construct a scalar field $\tilde{\phi}$ within the cell [S1].
2.  Extract the isosurface $\tilde{\phi} = 0.5$ using the isosurface extraction method described in [Parent](954-extended-clcir-plic-reconstruction.md) [S1].
3.  Compute the weighted normal vector $\mathbf{n}$ of the extracted isosurface [S1].
4.  The interface is reconstructed at instant $t^n$ using an extension to arbitrary grids of the CLCIR method [S1].

**Step 4: Flux Calculation (EMFPA)**
Calculate the net volume of fluid advected out of cell $i$ during $\Delta t$:
$$ \Phi_i = \int_{t^n}^{t^{n+1}} \int_{\partial \Omega_i} \mathbf{v} \cdot \mathbf{n}_f \chi(\mathbf{x}, t) \, dS \, dt $$
This is solved geometrically using the Edge-Matched Flux Polygon Advection (EMFPA) method [S1].
1.  For each face $f$ of cell $i$, construct the flux polygon representing the volume of fluid crossing the face [S1].
2.  Perform repeated intersection operations on the polyhedra formed by the flux regions [S1].
3.  Use geometric tools capable of handling non-convex polyhedra with self-intersecting faces to compute intersections without decomposition [S1].
4.  Sum the signed fluxes over all faces to get $\Phi_i$ [S1].

**Step 5: Volume Fraction Update**
Update the volume fraction using the integrated advection equation:
$$ F(t^{n+1}) = F(t^n) - \frac{1}{V_{\Omega}} \int_{t^n}^{t^{n+1}} \int_{\Omega} \nabla \cdot [\mathbf{v} \chi(\mathbf{x}, t)] \, d\mathbf{x} \, dt + \frac{F(t^{n+1}) - F(t^n)}{2V_{\Omega}} \int_{t^n}^{t^{n+1}} \int_{\Omega} \nabla \cdot \mathbf{v} \, d\mathbf{x} \, dt $$
In practice, the first integral is approximated by the geometric flux $\Phi_i$ calculated in Step 4 [S1]. The second term (divergence correction) must be null if the velocity vector field $\mathbf{v}$ is discretely solenoidal [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def advect_vof(grid, velocity, F, dt, cfl=0.5):
    """
    Advect VOF field F on arbitrary grid using EMFPA and CLCIR.
    
    Args:
        grid: List of cells. Each cell has vertices, faces, and volume V_omega.
        velocity: Dict mapping face_id to (u, v, w) vector.
        F: Dict mapping cell_id to float (volume fraction).
        dt: Time step size.
        cfl: CFL number (default 0.5).
        
    Returns:
        F_new: Updated volume fractions.
    """
    F_new = {}
    
    for cell_id, cell in grid.cells.items():
        F_old = F[cell_id]
        V_omega = cell.volume
        total_flux = 0.0
        divergence_sum = 0.0

        # Step 3: Interface Reconstruction (CLCIR)
        # Only reconstruct if interface is present: 1e-12 < F_old < 1 - 1e-12 [S1]
        if 1e-12 < F_old < 1 - 1e-12:
            # Construct scalar field, extract isosurface phi=0.5, compute weighted normal n [S1]
            # (Exact geometric construction not detailed in sources [G])
            cell.plic_normal = None 
        else:
            cell.plic_normal = None

        # Step 4: Flux Calculation (EMFPA)
        for face in cell.faces:
            u, v, w = velocity[face.id]
            # Construct flux polygon, perform repeated intersections on non-convex polyhedra [S1]
            # (Exact intersection algorithm not detailed in sources [G])
            flux_vol = 0.0 
            total_flux += flux_vol
            divergence_sum += dot(face.normal, (u, v, w)) * face.area

        # Step 5: Volume Fraction Update
        # Eq. 9: F(t^{n+1}) = F(t^n) - (1/V) * Flux_Integral + (F(t^{n+1})-F(t^n))/(2V) * Div_Integral [S1]
        # First integral approximated by total_flux [S1].
        # Second term is null if velocity is discretely solenoidal [S1].
        div_integral = divergence_sum * dt
        correction = 0.0 # Assumed zero for solenoidal fields [S1]

        F_new[cell_id] = F_old - total_flux / V_omega + correction
        F_new[cell_id] = max(0.0, min(1.0, F_new[cell_id])) # Clamp to [0, 1] [G]
        
    return F_new
```

## Subtopics

### EMFPA-CLCIR
Combines EMFPA advection with CLCIR reconstruction using isosurface extraction [S1]. Chosen for high accuracy on arbitrary grids [S1]. [Link](954-extended-clcir-plic-reconstruction.md)

### EMFPA-Youngs
Combines EMFPA advection with Youngs' method for interface reconstruction [S1]. Chosen for lower CPU time, though with higher error [S1]. [Link](954-extended-clcir-plic-reconstruction.md)

## Operating characteristics
*   **Time Complexity:** Determined by the cost of isosurface extraction and geometric flux polygon intersections [S1]. The EMFPA method avoids decomposition, reducing complexity compared to methods requiring cell subdivision [S1].
*   **Memory Complexity:** Requires storage for grid geometry, velocity field, and volume fractions [S1]. No additional large temporary arrays for decomposition are needed [S1].
*   **Performance:** On a 3D vortex-in-a-box test with CFL=0.5 and $t_{end}=3$:
    *   **Cubic Grids ($n \approx 32^3$):** EMFPA-CLCIR error $3.21 \times 10^{-3}$, CPU time 0.0088s. EMFPA-Youngs error $3.73 \times 10^{-3}$, CPU time 0.0067s [S1].
    *   **Cubic Grids ($n \approx 128^3$):** EMFPA-CLCIR error $2.14 \times 10^{-4}$, CPU time 0.13s. EMFPA-Youngs error $2.91 \times 10^{-4}$, CPU time 0.11s [S1].
    *   **Tetrahedral Grids ($n \approx 128^3$):** EMFPA-CLCIR error $3.51 \times 10^{-4}$, CPU time 0.37s. EMFPA-Youngs error $6.00 \times 10^{-4}$, CPU time 0.25s [S1].
    *   **Non-convex Polyhedral Grids ($n \approx 128^3$):** EMFPA-CLCIR error $3.44 \times 10^{-4}$, CPU time 2.19s. EMFPA-Youngs error $3.76 \times 10^{-4}$, CPU time 1.81s [S1].
*   **Convergence:** Second-order convergence observed for all grid types [S1].
*   **Strengths:** Handles arbitrary polyhedral grids (including non-convex and self-intersecting) without decomposition [S1]. High accuracy compared to Youngs' method [S1].
*   **Limitations:** Slightly higher CPU time than EMFPA-Youngs [S1]. Requires robust geometric intersection tools for non-convex cells [S1].

## Background
The method was proposed as a novel isosurface extraction technique for VOF simulations on arbitrary grids [S1]. It builds on the CLCIR method [3] and the EMFPA advection scheme [46, 47] [S1]. The isosurface extraction component utilizes tools from [26, 48] to handle complex cell geometries [S1]. The method is part of a broader effort to improve multiphase flow simulation on unstructured grids, addressing limitations of previous methods that struggled with non-convex cells or required decomposition [S1].

## Sources
[S1] A new isosurface extraction method on arbitrary grids, https://doi.org/10.1016/j.jcp.2021.110579, fetched 2026-10-26
