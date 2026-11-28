# Particle-Based Convective Cloud Simulation (Neyret)
Also called: Bubble Model
## Overview
Particle-Based Convective Cloud Simulation is an early physics-driven procedural method for simulating cumulus clouds by modeling air parcels as 2D particles, or "bubbles" [S1]. It solves the problem of generating realistic, animated cloud shapes and movement on a landscape scale by leveraging high-level physical variables such as buoyancy and condensation [S1]. Unlike later GPU-accelerated particle systems that use position-based dynamics or adaptive splitting/merging [S1], this method explicitly models mixing dynamics, inter-particle attraction, and attached vortices to enhance small-scale shape and movement features [S1].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Simulation domain parameters; initial conditions for bubble temperature, density, and velocity; atmospheric profiles determining buoyancy and condensation thresholds.
*   **Outputs:** Animated positions, sizes, and shapes of bubbles representing the cloud volume, suitable for volume ray casting in a shader [S1].

**Data Structures:**
Each bubble is characterized by the following fields [S1]:
*   `position`: 2D coordinates $(x, y)$ in the simulation domain.
*   `velocity`: 2D vector $(v_x, v_y)$.
*   `temperature`: Scalar value.
*   `density`: Scalar value.
*   `size`: Scalar value (radius or equivalent area metric).
*   `shape`: Parameters defining the blob's geometry (subject to "blob warping").

**Numbered Steps:**
1.  **Initialization:** Create initial bubbles based on convective triggers. Assign initial temperature, density, and velocity [S1].
2.  **Buoyancy Calculation:** Calculate the buoyant force acting on each bubble. Bubbles rise due to this force [S1]. The force is derived from the temperature and density difference between the bubble and the surrounding atmosphere [S1].
3.  **Condensation Check:** Determine if a bubble becomes visible. A bubble becomes visible (and thus part of the cloud) upon condensation, which is triggered when the bubble's temperature drops below the dew point or similar physical threshold [S1].
4.  **Mixing Dynamics:** Update the bubble's thermodynamic properties. The bubble undergoes mixing dynamics with the surrounding atmosphere and with neighboring bubbles [S1]. This process typically involves smoothing scalar fields like temperature to reduce value differences between neighboring particles [S1].
5.  **Inter-particle Forces:** Calculate attractive forces exerted by neighboring bubbles. Each bubble exerts an attractive force toward its neighbors, influencing their trajectories [S1].
6.  **Vortex Application:** Apply vortices attached to the bubbles. These vortices enhance small-scale shape and movement features on the clouds [S1].
7.  **Advection and Diffusion:** Advect the bubbles as a result of the diffusion process. The blobs change in size and shape (termed "blob warping") and are advected through the domain [S1].
8.  **Dynamic Creation/Destruction:** Manage bubble lifecycle. Bubbles can be dynamically created or destroyed in response to cloud evolution, such as dissipation or new updrafts [S1].
9.  **Rendering:** Generate the animated cloud cover through volume ray casting in the shader using the final positions and properties of the bubbles [S1].

**Formulas:**
*   **Buoyant Force:** The specific formula is not provided in the sources, but the force is determined by the buoyancy derived from temperature and density differences [S1].
*   **Blob Warping:** The change in size and shape is a result of the diffusion process and mixing dynamics [S1]. Exact warping functions are not detailed in the provided text.

**Parameters:**
*   Typical values for physical constants (e.g., gravity, specific heat) are not given in the sources [S1].
*   The method operates in 2D for the particle representation [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.
*   **Degenerate Input:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Particle-Based Convective Cloud Simulation (Neyret)
# Note: Specific physical constants and detailed warping functions are not provided in the sources [S1].

class Bubble:
    def __init__(self, x, y, vx, vy, temp, density, size):
        self.position = Vector2(x, y)
        self.velocity = Vector2(vx, vy)
        self.temperature = temp
        self.density = density
        self.size = size
        self.visible = False

    def update(self, atmosphere, neighbors, dt):
        # 1. Calculate Buoyancy
        # Buoyant force is derived from temperature/density difference with atmosphere [S1]
        buoyancy = calculate_buoyancy(self, atmosphere)
        
        # 2. Apply Buoyancy to Velocity
        self.velocity += buoyancy * dt
        
        # 3. Condensation Check
        # Bubble becomes visible upon condensation [S1]
        if self.temperature < atmosphere.dew_point_at(self.position):
            self.visible = True
            
        # 4. Mixing Dynamics
        # Mix with surrounding atmosphere and neighbors [S1]
        self.temperature = mix_temperature(self.temperature, atmosphere.temp_at(self.position))
        for neighbor in neighbors:
            self.temperature = mix_temperature(self.temperature, neighbor.temperature)
            self.density = mix_density(self.density, neighbor.density)
            
        # 5. Inter-particle Attraction
        # Exert attractive force toward neighbors [S1]
        attraction = calculate_attraction(self, neighbors)
        self.velocity += attraction * dt
        
        # 6. Vortex Application
        # Apply vortices to enhance small-scale features [S1]
        vortex_force = apply_vortices(self, dt)
        self.velocity += vortex_force
        
        # 7. Advection and Diffusion (Blob Warping)
        # Advect position and update size/shape [S1]
        self.position += self.velocity * dt
        self.size = warp_size(self.size, self.temperature, self.density)
        
        # 8. Dynamic Creation/Destruction
        # Handled externally based on cloud evolution [S1]
        if self.should_destroy():
            return False # Remove bubble
        return True # Keep bubble

def simulate_clouds(bubbles, atmosphere, dt):
    new_bubbles = []
    for bubble in bubbles:
        # Find neighbors (e.g., within a radius)
        neighbors = get_neighbors(bubble, bubbles)
        if bubble.update(atmosphere, neighbors, dt):
            new_bubbles.append(bubble)
    return new_bubbles
```
## Subtopics

### Sounding Data Based Cloud Simulation
Simulates real-time cumulus clouds based on data from SkewT/LogP diagrams, obtaining physical parameters from sounding data instead of determining them or setting initial conditions [S1]. The data provide the buoyant force to solve the equation of motion of the air parcels without solving differential equations [S1]. It ensures the evolution of parcels from birth to death through explicit steps, whereas other methods may handle this naturally as part of the simulation [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths and Limitations:** The method is one of the earliest to use high-level physics variables for convective cloud simulation [S1]. It enhances small-scale shape and movement features via vortices [S1]. However, it is a 2D particle model, which may limit volumetric detail compared to 3D methods.
*   **When to Use:** When a physics-driven, early-stage approach to convective cloud animation is required, particularly for understanding the foundational principles of bubble-based cloud modeling.
*   **When Not to Use:** When high-performance real-time rendering on modern GPUs is required, as later methods (e.g., Barbosa et al.) offer GPU acceleration and adaptive particles [S1].
## Background
Origin: This method is attributed to Neyret [S1]. It is one of the earliest works to obtain a simulation of convective clouds based on high-level physics variables, modeling parcels as 2D particles (bubbles) [S1].
## Sources
[S1] A survey of modeling, rendering and animation of clouds in computer graphics, https://doi.org/10.1007/s00371-020-01953-y, fetched 2026-10-26
