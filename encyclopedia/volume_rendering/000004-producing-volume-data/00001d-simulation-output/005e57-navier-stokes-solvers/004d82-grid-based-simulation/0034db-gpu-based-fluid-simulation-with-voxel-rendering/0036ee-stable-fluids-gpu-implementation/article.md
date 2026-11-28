# Stable Fluids (GPU Implementation)
Also called: Jacobi Pressure Solver, Jacobi method for pressure-Poisson system, Semi-Lagrangian Advection

## Overview
Stable Fluids (GPU Implementation) is a method for computing fluid velocity by discretizing the momentum equation on a fixed Eulerian grid and solving for velocity via advection, external forces, and pressure projection [S1, unofficial]. It solves the problem of real-time incompressible fluid simulation on graphics hardware by leveraging 3D textures and slice-based rendering to map grid cells directly to voxels [S1, unofficial]. It sets itself apart from its siblings by using the Jacobi method for the pressure-Poisson system due to its GPU efficiency, and Semi-Lagrangian advection for unconditional stability [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D grid of cubical cells representing the simulation domain; initial velocity field $\mathbf{u}$; pressure field $p$; external forces $\mathbf{f}$; time step $\Delta t$ [S1, unofficial].
*   **Outputs:** Updated velocity field $\mathbf{u}^{n+1}$; updated pressure field $p^{n+1}$ [G].

**Data Structures:**
State is stored in 3D textures mapping grid cells to voxels [S1, unofficial].
*   **Velocity Texture:** Stores vector quantities (velocity) at cell centers [S1, unofficial].
*   **Pressure Texture:** Stores scalar quantities (pressure) at cell centers [S1, unofficial].
*   **Geometry Shader Output Struct (`GS_OUTPUT_FLUIDSIM`):** Used to pass neighbor coordinates to pixel shaders [S1, unofficial].
    ```python
    class GS_OUTPUT_FLUIDSIM:
        def __init__(self, cell_index, center, left, right, bottom, top, down, up, pos, rt_index):
            self.cellIndex = cell_index
            self.CENTERCELL = center
            self.LEFTCELL = left
            self.RIGHTCELL = right
            self.BOTTOMCELL = bottom
            self.TOPCELL = top
            self.DOWNCELL = down
            self.UPCELL = up
            self.pos = pos
            self.RTIndex = rt_index
    ```

**Numbered Steps:**
The momentum equation is split into successive operations: advection, application of external forces, and pressure projection [S1, unofficial].

1.  **Domain Discretization:** Subdivide the rectilinear volume into a regular grid of cubical cells. Store scalar (pressure) and vector (velocity) quantities at cell centers [S1, unofficial].
2.  **Advection (Semi-Lagrangian):** Trace back along the characteristic from the current cell position to find the source velocity [S1, unofficial].
    *   Calculate position: $\mathbf{pos} = \mathbf{current\_pos} - \Delta t \cdot \mathbf{cellVelocity}$ [S1, unofficial].
    *   Convert index to texture coordinates [S1, unofficial].
    *   Sample the velocity field at the traced position using linear interpolation [S1, unofficial].
    *   *Note:* This scheme is unconditionally stable but introduces numerical smoothing [S1, unofficial].
3.  **External Forces:** Apply forces (e.g., gravity) to the advected velocity field using forward Euler integration: $\mathbf{u}^* = \mathbf{u}^{advected} + \Delta t \cdot \mathbf{f}$ [S1, unofficial].
4.  **Divergence Calculation:** Compute the divergence of the intermediate velocity field $\mathbf{u}^*$ using central differences on neighboring cell velocities [S1, unofficial].
    $$ \nabla \cdot \mathbf{u}^* = 0.5 \cdot ((u^*_R - u^*_L) + (v^*_T - v^*_B) + (w^*_U - w^*_D)) $$
    Where $R, L, T, B, U, D$ refer to Right, Left, Top, Bottom, Up, Down neighbors respectively [S1, unofficial].
5.  **Pressure Solve (Jacobi):** Solve the pressure-Poisson system $\nabla^2 p = \nabla \cdot \mathbf{u}^*$ iteratively [S1, unofficial].
    *   Initialize pressure field (e.g., to zero) [G].
    *   For each iteration:
        $$ p_{new} = \frac{p_L + p_R + p_B + p_T + p_U + p_D - \nabla \cdot \mathbf{u}^*}{6.0} $$
    *   Repeat for a fixed number of iterations [S1, unofficial]. Liquids require more iterations to maintain volume than smoke/fire [S1, unofficial].
6.  **Projection:** Subtract the pressure gradient from the velocity to enforce incompressibility ($\nabla \cdot \mathbf{u} = 0$) [G].
    *   Compute gradient: $\nabla p = 0.5 \cdot \mathbf{Vector3}(p_R - p_L, p_T - p_B, p_U - p_D)$ [S1, unofficial].
    *   Update velocity: $\mathbf{u}^{n+1} = \mathbf{u}^* - \nabla p$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __add__(self, o): return Vector3(self.x+o.x, self.y+o.y, self.z+o.z)
    def __sub__(self, o): return Vector3(self.x-o.x, self.y-o.y, self.z-o.z)
    def __mul__(self, s): return Vector3(self.x*s, self.y*s, self.z*s)
    def __getitem__(self, i): return (self.x, self.y, self.z)[i]

class Vector4:
    def __init__(self, x, y, z, w): self.x, self.y, self.z, self.w = x, y, z, w
    def __getitem__(self, i): return (self.x, self.y, self.z, self.w)[i]

def sample_clamp(tex, coord, width, height, depth):
    x = max(0.0, min(1.0, coord.x))
    y = max(0.0, min(1.0, coord.y))
    z = max(0.0, min(1.0, coord.z))
    return tex[int(x*width), int(y*height), int(z*depth)]

def sample_linear(tex, coord, width, height, depth):
    x = max(0.0, min(1.0, coord.x))
    y = max(0.0, min(1.0, coord.y))
    z = max(0.0, min(1.0, coord.z))
    return tex[int(x*width), int(y*height), int(z*depth)]

def cell_idx_to_tex(idx, width, height, depth):
    return Vector3(idx.x / width, idx.y / height, (idx.z + 0.5) / depth)

def make_neighbor_coords(idx, width, height, depth):
    c = cell_idx_to_tex(idx, width, height, depth)
    return GS_OUTPUT_FLUIDSIM(idx, c,
        cell_idx_to_tex(Vector3(idx.x-1, idx.y, idx.z), width, height, depth),
        cell_idx_to_tex(Vector3(idx.x+1, idx.y, idx.z), width, height, depth),
        cell_idx_to_tex(Vector3(idx.x, idx.y-1, idx.z), width, height, depth),
        cell_idx_to_tex(Vector3(idx.x, idx.y+1, idx.z), width, height, depth),
        cell_idx_to_tex(Vector3(idx.x, idx.y, idx.z-1), width, height, depth),
        cell_idx_to_tex(Vector3(idx.x, idx.y, idx.z+1), width, height, depth),
        Vector3(0,0,0,1), 0)

def advect_vel(in_data, vel_tex, dt, width, height, depth):
    pos = in_data.cellIndex - dt * sample_clamp(vel_tex, in_data.CENTERCELL, width, height, depth).xyz
    pos = cell_idx_to_tex(pos, width, height, depth)
    return sample_linear(vel_tex, pos, width, height, depth)

def compute_divergence(in_data, vel_tex, width, height, depth):
    fl = sample_clamp(vel_tex, in_data.LEFTCELL, width, height, depth)
    fr = sample_clamp(vel_tex, in_data.RIGHTCELL, width, height, depth)
    fb = sample_clamp(vel_tex, in_data.BOTTOMCELL, width, height, depth)
    ft = sample_clamp(vel_tex, in_data.TOPCELL, width, height, depth)
    fd = sample_clamp(vel_tex, in_data.DOWNCELL, width, height, depth)
    fu = sample_clamp(vel_tex, in_data.UPCELL, width, height, depth)
    return 0.5 * ((fr.x - fl.x) + (ft.y - fb.y) + (fu.z - fd.z))

def jacobi_solve(in_data, pres_tex, div_tex, width, height, depth):
    d = sample_clamp(div_tex, in_data.CENTERCELL, width, height, depth)
    p = (sample_clamp(pres_tex, in_data.LEFTCELL, width, height, depth) +
         sample_clamp(pres_tex, in_data.RIGHTCELL, width, height, depth) +
         sample_clamp(pres_tex, in_data.BOTTOMCELL, width, height, depth) +
         sample_clamp(pres_tex, in_data.TOPCELL, width, height, depth) +
         sample_clamp(pres_tex, in_data.DOWNCELL, width, height, depth) +
         sample_clamp(pres_tex, in_data.UPCELL, width, height, depth) - d) / 6.0
    return p

def project_velocity(in_data, pres_tex, vel_tex, width, height, depth):
    p = Vector3(sample_clamp(pres_tex, in_data.RIGHTCELL, width, height, depth) - sample_clamp(pres_tex, in_data.LEFTCELL, width, height, depth),
                sample_clamp(pres_tex, in_data.TOPCELL, width, height, depth) - sample_clamp(pres_tex, in_data.BOTTOMCELL, width, height, depth),
                sample_clamp(pres_tex, in_data.UPCELL, width, height, depth) - sample_clamp(pres_tex, in_data.DOWNCELL, width, height, depth)) * 0.5
    v_old = sample_clamp(vel_tex, in_data.CENTERCELL, width, height, depth).xyz
    return Vector4((v_old - p).x, (v_old - p).y, (v_old - p).z, 0)

def simulate_step(velocity_tex, pressure_tex, divergence_tex, time_step, num_jacobi_iters, force_field, width, height, depth):
    # 1. Advection
    new_velocity = [[None]*depth for _ in range(width*height)]
    for z in range(depth):
        for y in range(height):
            for x in range(width):
                idx = Vector3(x, y, z)
                in_data = make_neighbor_coords(idx, width, height, depth)
                in_data.RTIndex = z
                new_velocity[y*width + x][z] = advect_vel(in_data, velocity_tex, time_step, width, height, depth)

    # 2. External Forces
    intermediate_velocity = [[None]*depth for _ in range(width*height)]
    for z in range(depth):
        for y in range(height):
            for x in range(width):
                adv = new_velocity[y*width + x][z]
                force = force_field(x, y, z)
                intermediate_velocity[y*width + x][z] = adv + force * time_step

    # 3. Divergence
    div_tex = [[None]*depth for _ in range(width*height)]
    for z in range(depth):
        for y in range(height):
            for x in range(width):
                idx = Vector3(x, y, z)
                in_data = make_neighbor_coords(idx, width, height, depth)
                in_data.RTIndex = z
                div_tex[y*width + x][z] = compute_divergence(in_data, intermediate_velocity, width, height, depth)

    # 4. Jacobi Iterations (Ping-pong buffers)
    p_prev = pressure_tex
    p_curr = [[0.0]*depth for _ in range(width*height)]
    for _ in range(num_jacobi_iters):
        for z in range(depth):
            for y in range(height):
                for x in range(width):
                    idx = Vector3(x, y, z)
                    in_data = make_neighbor_coords(idx, width, height, depth)
                    in_data.RTIndex = z
                    p_curr[y*width + x][z] = jacobi_solve(in_data, p_prev, div_tex, width, height, depth)
        p_prev, p_curr = p_curr, p_prev

    # 5. Projection
    final_velocity = [[None]*depth for _ in range(width*height)]
    for z in range(depth):
        for y in range(height):
            for x in range(width):
                idx = Vector3(x, y, z)
                in_data = make_neighbor_coords(idx, width, height, depth)
                in_data.RTIndex = z
                final_velocity[y*width + x][z] = project_velocity(in_data, p_prev, intermediate_velocity, width, height, depth)
    return final_velocity
```

**Parameters:**
*   `timeStep` ($\Delta t$): The duration of the time step used in semi-Lagrangian advection [S1, unofficial].
*   `Iterations`: The number of Jacobi iterations. Adjustable based on resources. Smoke/fire may use fewer; liquids require more to prevent volume loss [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **GPU Rendering Limitation:** GPUs render into 2D buffers. 3D volume kernels must be executed once per slice by iterating over slice indices and using `SV_RenderTargetArrayIndex` [S1, unofficial].
*   **Numerical Smoothing:** Semi-Lagrangian advection is unconditionally stable but introduces smoothing, making water look viscous or smoke lose detail [S1, unofficial].
*   **Volume Loss:** Too few Jacobi iterations for liquids cause volume loss, making water sink through the tank bottom because pressure from the floor cannot counteract gravity [S1, unofficial].
*   **Boundary Conditions:** The basic implementation stores quantities at cell centers [S1, unofficial]. Staggered MAC grids are more robust for boundary conditions but are not the default in this specific implementation [S1, unofficial].

## Subtopics

### [Free-Slip Boundary Condition Enforcement](7908-free-slip-boundary-condition-enforcement.md)
This method enforces a free-slip boundary condition for fluid simulations involving dynamic obstacles represented as voxelized solids. It ensures that fluid velocity normal to the obstacle surface matches the solid's normal velocity, allowing tangential slip, while preventing mass leakage into solid interiors. It differs from no-slip conditions by retaining tangential fluid velocity and from static boundary handling by explicitly updating obstacle velocity fields and modifying the pressure solver to ignore solid cell contributions.

### [Projection Method](8336-projection-method.md)
The Projection Method enforces incompressibility on a fluid velocity field by subtracting the gradient of a pressure field, ensuring the resulting velocity is divergence-free. It serves as the incompressibility enforcement step within the Stable Fluids framework [G].

### [Operator Splitting for Fluid Dynamics](12465-operator-splitting-for-fluid-dynamics.md)
Operator splitting decomposes the complex dynamic equations of fluid dynamics into sequential, simpler sub-steps to facilitate numerical integration on a grid. It solves the problem of enforcing incompressibility while applying external forces and advecting the fluid state. It sets itself apart from the parent [Stable Fluids (GPU Implementation)](7906-stable-fluids-gpu-implementation.md) by explicitly structuring the time integration into three distinct, ordered phases: advection, force application, and pressure projection.

### [GPU-based Eulerian Fluid Solver](14187-gpu-based-eulerian-fluid-solver.md)
A GPU-based Eulerian fluid solver that simulates and renders volumetrics at VR resolutions and refresh rates by using operator splitting to approximately solve the Navier-Stokes equations. It distinguishes itself by integrating a custom injection kernel for source terms and a geometric multigrid pressure solver optimized for half-precision 3D textures, enabling interactive frame rates on consumer hardware.

### Lagrangian Advection
A method to advect fluid by moving particles according to their velocities, where physical quantities like mass and velocity are bound to particles [S2, unofficial]. The process moves all particles according to their velocities $v_{t-1}$ (average velocity over $t-1 \to t$), then applies external force and pressure projection to the advected velocities to get final velocities at the next time step $t$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The dominant cost is the pressure solve [S1, unofficial]. With Jacobi iteration, the cost is $O(N \cdot I)$, where $N$ is the number of grid cells and $I$ is the number of iterations [G].
*   **Memory Complexity:** Storage is proportional to the grid size [S1, unofficial]. Table 30-1 in [S1, unofficial] indicates ~9 bytes per cell for simulation (velocity RGBA16, inside-outside R8, temporary RGBA16) and ~20 bytes per pixel for rendering (off-screen targets, ray data, depth) [S1, unofficial].
*   **Measured Performance:** Designed for real-time applications [S1, unofficial]. Performance depends on the number of Jacobi iterations [S1, unofficial]. Smoke/fire can tolerate fewer iterations for higher frame rates, while liquids require more [S1, unofficial].
*   **Strengths:** Unconditional stability of semi-Lagrangian advection; efficient GPU implementation using 3D textures and geometry shaders; simple to implement [S1, unofficial].
*   **Limitations:** Numerical smoothing from semi-Lagrangian advection; volume loss if Jacobi iterations are insufficient for liquids; cell-centered storage is less robust than staggered grids for complex boundaries [S1, unofficial].
*   **When to use:** Real-time fluid simulation where unconditional stability is prioritized, particularly for smoke and fire; liquids require careful iteration tuning to prevent volume loss [S1, unofficial].
*   **When not to use:** High-fidelity liquid simulation requiring strict volume conservation without excessive iterations; simulations requiring complex boundary conditions handled by no-slip or dynamic obstacles (use [Free-Slip Boundary Condition Enforcement](7908-free-slip-boundary-condition-enforcement.md) or [GPU-based Eulerian Fluid Solver](14187-gpu-based-eulerian-fluid-solver.md)) [S1, unofficial].

## Background
The "stable fluids" method for computing velocity was introduced by Jos Stam in 1999 [S1, unofficial]. A GPU implementation of this method for 2D fluids was presented by Mark Harris in 2004 [S1, unofficial]. The 3D GPU implementation described in the sources extends these concepts to volumetric grids using Direct3D 10 features like 3D textures and geometry shaders [S1, unofficial]. Alternatives to the Jacobi solver include Conjugate Gradient (Bolz et al. 2003), Multigrid (Goodnight et al. 2003), and Cyclic reduction (Kass et al. 2006) [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] Inferring Fluid Dynamics via Inverse Rendering, https://doi.org/10.48550/arxiv.2304.04446, fetched 2026-10-26 (unofficial)
