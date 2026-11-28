# Particle Sprite Rendering

Also called: Billboard Rendering, Particle System Renderer, Point Sprite Rendering, particle system

## Overview
Particle Sprite Rendering displays 3D data points as members of a particle system without surface fitting or interpolation, using animation for depth cues [S1, unofficial]. It differs from surface rendering by not mapping data points onto a surface, avoiding surface fitting overhead and interpolation bias [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Radar Cross Section (RCS) data points (phi, theta, rad, strength, polarity) [S1, unofficial].
*   **Output:** A sequence of frames (images) stored in Utah Raster Toolkit (RLE) format [S1, unofficial].

**Data Structures**
*   **Particle List:** A dynamic linked list storing particles and their respective centroid [S1, unofficial].
*   **Centroid:** Defined by user-specified origin in world coordinates and shape (e.g., spherical) [S1, unofficial].
*   **View Volume:** Defined by Viewer Position (PRP), Camera Looking Direction (CW), View Plane Normal (VPN), and View Up Vector (VUP) [S1, unofficial].

**Numbered Steps**
1.  **Initialization:** All particles start at the same time. The starting position is determined by the centroid shape and coordinate system (spherical or cartesian). For RCS data, particles start at a sphere around the centroid with radius zero [S1, unofficial].
2.  **Time Step Loop:**
    1.  **Check for Spawning:** Iterate through the particle list for new particles. If spawning, generate the new system and add to the linked list [S1, unofficial].
    2.  **Check for Death:** If a particle is dead, mark it and ignore it in subsequent steps [S1, unofficial].
    3.  **Move Particle:** Update particle position based on velocity. Velocity is determined by the strength of the RCS return [S1, unofficial].
    4.  **Render Particles:** Render the current state of the particle system [S1, unofficial].
    5.  **Increment Time Step:** Advance the simulation time [S1, unofficial].

**Formulas**
*   **Particle Position:** Derived from RCS data phi and theta, with radial distance starting at zero [S1, unofficial].
*   **Particle Alpha:**
    $$ \alpha = \text{clamp}(1.0 - \text{distanceFromCenter} \times 2.0, 0.0, 1.0) \times uParticleAlpha $$
    Where `distanceFromCenter` is the distance from the center of the point sprite [S2, unofficial].
*   **Particle Color (Modulated by Shadow Opacity):**
    $$ \text{color} = (1.0 - vOpacity \times 0.75) \times uParticleColor $$
    Where `vOpacity` is the shadow opacity and `uParticleColor` is the base color [S2, unofficial].
*   **Output Premultiplied Alpha Color:**
    $$ \text{FragColor} = \text{vec4}(\text{color} \times \alpha, \alpha) $$
    [S2, unofficial]

**Parameters**
*   **Centroid Shape:** Spherical (chosen for RCS to represent the center of a spherical coordinate system) [S1, unofficial].
*   **Coordinate System:** Spherical or Cartesian [S1, unofficial].
*   **Clipping:** World limits including front and back [S1, unofficial].
*   **Memory Requirement:** At least 16 megabytes [S1, unofficial].
*   **Filtering (Polarity Specific):**
    *   **W Polarity:** Low pass filter at 20 dBsm, High pass filter at 65 dBsm. Particle color: Red [S1, unofficial].
    *   **HH Polarity:** High pass and low pass filter limits set to 0. Particle color: Blue [S1, unofficial].
    *   **HV and VH Polarity:** High pass and low pass filter limits set to 0. Particle color: Green [S1, unofficial].
    *   **Multi-Polarity:** Low pass filter set to 0, High pass filter set to 70 [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data structures
class Particle:
    position: Vector3
    velocity: float
    color: Color
    is_dead: bool
    spawn_time: float
    lifetime: float

class ParticleSystem:
    particles: List[Particle]
    centroid: Centroid
    time_step: float
    current_time: float

def render_particle_system(system: ParticleSystem, view_params: ViewParams):
    """
    Renders the particle system for a single frame.
    """
    system.current_time += system.time_step
    
    # 1. Check for spawning
    new_particles = generate_new_particles(system.centroid, system.current_time)
    for p in new_particles:
        system.particles.append(p)
        
    # 2. Check for death and move
    alive_particles = []
    for p in system.particles:
        if p.is_dead:
            continue
            
        # Check if particle has exceeded lifetime
        if system.current_time - p.spawn_time > p.lifetime:
            p.is_dead = True
            continue
            
        # Move particle
        # Velocity is determined by strength of RCS return
        p.position += p.velocity * system.time_step
        
        alive_particles.append(p)
        
    system.particles = alive_particles
    
    # 3. Render particles
    # Use Z-Buffer and Wu antialiased lines for rendering
    for p in alive_particles:
        render_particle(p, view_params)

def render_particle(p: Particle, view_params: ViewParams):
    """
    Renders a single particle as a sprite.
    """
    # Project particle to screen space
    screen_pos = project(p.position, view_params)
    
    # Determine size based on perspective (handled by vertex shader in modern pipelines)
    # or fixed size for legacy Wu line rendering
    size = calculate_sprite_size(p.position, view_params)
    
    # Draw sprite
    # Legacy: Wu antialiased lines drawn by incrementing one pixel at a time
    # along the major axis. For each increment, two pixels are drawn bracketing the axis.
    # Intensity determined by distance from actual line.
    draw_wu_line_sprite(screen_pos, size, p.color)
    
    # Modern Fragment Shader approach (if using point sprites):
    # if (distanceFromCenter > 0.5) discard
    # alpha = clamp(1.0 - distanceFromCenter * 2.0, 0.0, 1.0) * uParticleAlpha
    # color = (1.0 - vOpacity * 0.75) * uParticleColor
    # FragColor = vec4(color * alpha, alpha)

def generate_new_particles(centroid: Centroid, time: float) -> List[Particle]:
    """
    Generates new particles based on centroid and time.
    All particles start at the same time.
    """
    # For RCS data, particles start at a sphere around the centroid with radius zero
    # Coordinates derived from phi and theta of RCS data
    particles = []
    for data_point in centroid.data:
        p = Particle()
        p.position = spherical_to_cartesian(data_point.phi, data_point.theta, 0.0, centroid.origin)
        p.velocity = data_point.strength # Velocity determined by strength
        p.color = get_color_by_polarity(data_point.polarity)
        p.spawn_time = time
        p.lifetime = calculate_lifetime(data_point.strength)
        p.is_dead = False
        particles.append(p)
    return particles

def get_color_by_polarity(polarity: str) -> Color:
    if polarity == 'W':
        return Color(1.0, 0.0, 0.0) # Red
    elif polarity == 'HH':
        return Color(0.0, 0.0, 1.0) # Blue
    elif polarity in ['HV', 'VH']:
        return Color(0.0, 1.0, 0.0) # Green
    else:
        return Color(1.0, 1.0, 1.0) # Default White
```

**Edge Cases and Pitfalls**
*   **Random Placement:** Random particle placement is not used for RCS data because data is mapped to specific locations [S1, unofficial].
*   **Static Images:** Static images lack sufficient cues for the volumetric nature of the data. This is addressed by animating the development over time and allowing view rotation while growing [S1, unofficial].
*   **Resolution Loss:** Significant loss of resolution occurs when transferring images to tape and displaying on NTSC monitors, causing blurring. Lack of standard aspect ratios in NTSC monitors alters the look of data, potentially hiding areas or falsely indicating large returns [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly defined in sources. Rendering time was approximately 95 seconds per frame for the final data set [S1, unofficial].
*   **Memory Complexity:** At least 16 megabytes of memory required [S1, unofficial].
*   **Measured Performance:** Rendering a final data set required 10 minutes to load data files. The final video tape lasted 3 minutes, took 1900 separate images, and required 6 hours to record [S1, unofficial].
*   **Strengths:** Minimal filtering and data manipulation; no surface fitting overhead; no interpolation bias; animation provides additional 3D cues [S1, unofficial].
*   **Limitations:** Not designed for interactive display; significant resolution loss on NTSC monitors; aspect ratio sensitivity [S1, unofficial].
*   **When to Use:** When displaying robust, vectorized data sets (like RCS) where volumetric nature needs to be conveyed through animation and minimal data manipulation is desired [S1, unofficial].
*   **When Not to Use:** When interactive display is required; when high resolution on standard monitors is critical without post-processing [S1, unofficial].

## Background
*   **Origin:** Based on the description by Reeves [Ree 83] [S1, unofficial].
*   **History:** The code was written in AT&T C++ version 2.1 [S1, unofficial]. The system was designed with portability and flexibility as primary considerations [S1, unofficial]. It was used for viewing Radar Cross Section (RCS) data in three dimensions [S1, unofficial].
*   **Usage:** Used for generating frame-by-frame depictions of environments for video recording and playback, providing additional 3D cues for data analysis [S1, unofficial].

## Sources
[S1] DTIC ADA258905: Methods for Viewing Radar Cross Section Data in Three Dimensions, https://archive.org/details/DTIC_ADA258905, fetched 2026-10-26 (unofficial)
[S2] GitHub - zk8085454/VolumeParticleShadows: Half angle slicing technique ..., https://github.com/zk8085454/VolumeParticleShadows/, fetched 2026-10-28 (unofficial)
