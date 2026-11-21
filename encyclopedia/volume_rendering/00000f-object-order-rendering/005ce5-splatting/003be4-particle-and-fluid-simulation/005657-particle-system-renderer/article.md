# Particle System Renderer

Also called: PSR

## Overview
The Particle System Renderer (PSR) is a general-purpose, object-order particle rendering system designed for generating visualizations of discrete particle distributions, originally adapted for Radar Cross Section (RCS) data but maintained as a flexible utility [S1, unofficial]. It solves the problem of converting a set of discrete, time-evolving particles with physical properties into a rasterized image by managing particle lifecycles (spawn, move, die) and projecting them onto a defined viewport [S1, unofficial]. Unlike modern GPU-based splatting methods that rely on fragment shaders and depth buffers, PSR is a CPU-based, linked-list-driven system that explicitly manages particle state transitions and supports procedural generation of particle systems from centroids [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A data file defining particle systems (centroids, shapes, velocities, lifetimes) and a control file defining the environment (viewport, world limits, camera view, time steps) [S1, unofficial].
*   **Output:** RLE (Run-Length Encoded) image files, one per frame, generated using the Utah Raster Toolkit [S1, unofficial].

**Data Structures:**
Particles and centroids are stored in a dynamic linked list [S1, unofficial].
*   **Particle Node:** Contains position, velocity, color, tail length, time of death, time of spawn, and a flag indicating if the particle is dead [S1, unofficial].
*   **Centroid Node:** Defines the origin and shape (spherical or Cartesian) from which particles are generated [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** Parse the control file to set the viewport, world limits, camera view (position, center of interest, view plane normal, view up vector), and time increment parameters [S1, unofficial]. Initialize the linked list of particles and centroids [S1, unofficial].
2.  **Time Step Loop:** For each time step:
    a.  **Spawn Check:** Iterate through the linked list to identify particles marked for spawning (based on `time of spawn` attribute). Generate new particle systems at the current position of the spawning particle. The new system's origin is the parent particle's position [S1, unofficial].
    b.  **Update Physics:** Update particle positions based on velocity. The system currently supports only linear movement [S1, unofficial].
    c.  **Death Check:** Iterate through the list to identify particles whose current time exceeds their `time of death` [S1, unofficial]. Mark these particles as dead.
    d.  **Rendering:** Project the active (non-dead) particles onto the 2D image plane defined by the viewport and camera view [S1, unofficial].
    e.  **Cleanup:** Remove or mark dead particles so they are no longer displayed in subsequent frames [S1, unofficial].
3.  **View Change (Optional):** If a view change is specified in the control file, interpolate the View Plane Normal (VPN) and View Up (VUP) vectors from the start values (`svpn`, `svup`) to the finish values (`fvpn`, `fvup`) over the specified duration or step count. If the `freeze` token is present, particle growth is halted during this transition [S1, unofficial].
4.  **Output:** Write the accumulated pixel data to an RLE file, appending the current frame number to the output filename [S1, unofficial].

**Formulas and Parameters:**
*   **Viewport:** Defined by two min/max integer pairs $(x_{min}, x_{max}, y_{min}, y_{max})$. The range is inclusive. Width $W = x_{max} - x_{min} + 1$ [S1, unofficial].
*   **World Limits:** Defined by min/max integer pairs for clipping, plus front and back clipping planes [S1, unofficial].
*   **View Definition:** Requires camera position $\mathbf{p}_{cam}$, center of interest $\mathbf{c}_{oi}$, view plane normal $\mathbf{n}_{vpn}$, and view up vector $\mathbf{v}_{vup}$ [S1, unofficial].
*   **Particle Origin (Spherical):** Defined by $\phi, \theta, r$ relative to the centroid [S1, unofficial].
*   **Particle Origin (Cartesian):** Defined by XYZ coordinates relative to the centroid [S1, unofficial].
*   **Tail Length:** A float value specifying the maximum length of the particle's tail. Default is $-1$, which disables the feature [S1, unofficial].
*   **Time Increment:** User-specified step size (seconds) and duration (seconds or number of steps) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
class Particle:
    position: Vector3
    velocity: Vector3
    color: Color
    tail_length: float
    time_of_death: float
    time_of_spawn: float
    spawn_source: str # "generate" or "read"
    is_dead: bool
    age: float

class Centroid:
    shape: str # "spherical" or "cartesian"
    origin: Vector3
    diameter: float
    color: Color

class Environment:
    viewport: Tuple[int, int, int, int] # xmin, xmax, ymin, ymax
    world_limits: Tuple[int, int, int, int] # xmin, xmax, ymin, ymax
    clip_front: float
    clip_back: float
    cam_pos: Vector3
    coi: Vector3
    vpn: Vector3
    vup: Vector3
    step_size: float
    duration_seconds: float
    duration_steps: int
    view_change_active: bool
    view_change_start_vpn: Vector3
    view_change_start_vup: Vector3
    view_change_end_vpn: Vector3
    view_change_end_vup: Vector3
    freeze_growth: bool

# Main Loop
def render_particle_system(particles: List[Particle], centroids: List[Centroid], env: Environment):
    current_time = 0.0
    
    while current_time < env.duration_seconds or env.duration_steps > 0:
        # 1. Spawn New Particles
        for p in particles:
            if not p.is_dead and p.time_of_spawn > 0 and p.age >= p.time_of_spawn:
                # Generate new system at p's position
                # Note: Source states only "generate" (random) is currently supported
                new_system = generate_random_system(p.position, centroids)
                particles.extend(new_system)
                p.time_of_spawn = -1 # Prevent re-spawning
        
        # 2. Update Physics
        for p in particles:
            if not p.is_dead:
                p.position += p.velocity * env.step_size
                p.age += env.step_size
                
                # 3. Check Death
                if p.time_of_death > 0 and p.age >= p.time_of_death:
                    p.is_dead = True
        
        # 4. Handle View Change
        if env.view_change_active:
            # Interpolate VPN and VUP
            # Note: Exact interpolation formula not given in source, assume linear
            t = current_time / env.duration_seconds
            env.vpn = lerp(env.view_change_start_vpn, env.view_change_end_vpn, t)
            env.vup = lerp(env.view_change_start_vup, env.view_change_end_vup, t)
            
            if env.freeze_growth:
                # Halt particle growth (implementation detail not specified)
                pass
        
        # 5. Render Active Particles
        image = create_image(env.viewport)
        for p in particles:
            if not p.is_dead:
                # Project particle to screen
                screen_pos = project(p.position, env.cam_pos, env.vpn, env.vup, env.viewport)
                # Check world clipping
                if is_within_world_limits(p.position, env.world_limits, env.clip_front, env.clip_back):
                    draw_particle(image, screen_pos, p.color, p.tail_length)
        
        # 6. Output
        save_rle(image, f"output_frame_{current_time}")
        
        current_time += env.step_size
        env.duration_steps -= 1
```

**Edge Cases and Pitfalls:**
*   **Fade Time:** The feature to fade out particle tails after death is defined as a hook but is not implemented [S1, unofficial].
*   **Direction Attribute:** The `direction` attribute, originally intended to work with velocity, is obsolete [S1, unofficial].
*   **Spawn From:** The `from` token for spawning systems currently only supports the `generate` token (random generation). The `read` token (from file) is not supported [S1, unofficial].
*   **Viewport Inclusion:** The viewport is inclusive; a range of 0 to 10 is 11 pixels wide, not 10 [S1, unofficial].
*   **Tail Length:** A negative tail length disables the tail feature; it does not imply an infinite tail [S1, unofficial].

## Subtopics

### Billboards and particles
Uses 2D textured quads to represent particles; simpler than PSR's explicit linked-list management but lacks the procedural generation features [9045-billboards-and-particles.md].

### Screen-Space Fluid Rendering Pipeline
A GPU-based approach for fluid rendering using screen-space techniques; PSR is a CPU-based, offline renderer [14414-screen-space-fluid-rendering-pipeline.md].

### Particle-Based Fire
Specialized for fire effects with specific emission and turbulence models; PSR is a general-purpose system [14658-particle-based-fire.md].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active particles $N$ per time step, as it iterates through a linked list for spawning, updating, and rendering [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the number of particles, as each particle is stored in a dynamic linked list node [S1, unofficial].
*   **Performance:** Performance is limited by CPU processing speed and the overhead of linked list traversal. The system requires the Utah Raster Toolkit for output, which may add I/O overhead [S1, unofficial].
*   **Strengths:** Flexible utility for generating particle systems from centroids; supports procedural generation and explicit lifecycle management (spawn, die) [S1, unofficial].
*   **Limitations:** No GPU acceleration; limited to linear velocity; no implemented tail fading; spawn from file not supported; obsolete features (direction) remain in the syntax [S1, unofficial].
*   **When to use:** For legacy RCS data visualization or when explicit control over particle lifecycle and procedural generation from centroids is required on a CPU-only system [S1, unofficial].
*   **When not to use:** For real-time rendering, large-scale particle systems requiring GPU acceleration, or when advanced visual effects like tail fading are needed [S1, unofficial].

## Background
The Particle System Renderer (PSR) was developed as part of research into methods for viewing Radar Cross Section (RCS) data in three dimensions [S1, unofficial]. It was designed to be a general-purpose system for generating particle systems, with features added to support RCS data rendering while remaining a flexible utility [S1, unofficial]. The system relies on the Utah Raster Toolkit version 3.0 for generating RLE output files [S1, unofficial].

## Sources
[S1] DTIC ADA258905: Methods for Viewing Radar Cross Section Data in Three Dimensions, https://archive.org/details/DTIC_ADA258905, fetched 2026-10-26 (unofficial)
