# Hardware Blending Recurrence Evaluation

Also called: Frame buffer read back blending

## Overview
Hardware Blending Recurrence Evaluation is a technique for evaluating the multiple forward scattering recurrence $I_k = g_{k-1} + T_{k-1} I_{k-1}$ by mapping the mathematical operation to GPU hardware blending and pixel read-back operations [S1, unofficial]. It computes the incident light on each particle in a participating medium by using the frame buffer as an accumulator [S1, unofficial]. It differs from [Incremental Lighting for In-Scattering](3401-incremental-lighting-for-in-scattering.md) by explicitly reading back pixel values to resolve the recurrence dependency [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of particles $P$, each with extinction $\tau_k$, albedo $a_k$, radius $r_k$, and position. Light direction $\mathbf{l}$ and light color/intensity [S1, unofficial].
*   **Outputs:** The scattered light contribution $c$ for each particle, which is then rendered toward the eye [S1, unofficial].

**Data Structures:**
*   **Frame Buffer:** Stores the accumulated incident light intensity $I$. Initialized to white (representing $I_0 = 1$) [S1, unofficial].
*   **Particle List:** Sorted by distance to the light source for the preprocessing pass [S1, unofficial].

**Parameters:**
*   **Source Blend Factor ($f_{src}$):** $1$ [S1, unofficial].
*   **Destination Blend Factor ($f_{dest}$):** $T_{k-1} = \exp(-\tau_{k-1})$ (transparency of the previous particle) [S1, unofficial].
*   **Texture/Fragment Mode:** Modulate [S1, unofficial].

**Steps:**
1.  **Preprocessing (Illumination Pass):**
    1.  Clear the frame buffer to white. This represents the initial incident light $I_0$ [S1, unofficial].
    2.  Sort particles $p_k$ by increasing distance to the light source [S1, unofficial].
    3.  For each particle $p_k$ in sorted order:
        a.  **Read Back:** Read the pixel color $x$ at the projection of the particle's center from the frame buffer. This value represents $I_{k-1}$, the incident light on the previous particle [S1, unofficial].
        b.  **Compute Incident Light:** Calculate $I_k = \text{color}(x) \times \text{light\_color}$. This is the light incident on the current particle $p_k$ [S1, unofficial].
        c.  **Compute Scattered Color:** Calculate the scattered light contribution $g_{k-1}$ using the formula:
            $$c = a_k \cdot \tau_k \cdot I_k / 4\pi$$
            where $a_k$ is albedo, $\tau_k$ is extinction, and $I_k$ is incident light [S1, unofficial].
        d.  **Set Render Parameters:**
            *   Set the particle's alpha to $1 - \exp(-\tau_k)$ [S1, unofficial].
            *   Set the particle's color to $c$ [S1, unofficial].
        e.  **Render and Blend:** Render the particle. The hardware blending equation is:
            $$c_{result} = f_{src} \cdot c_{src} + f_{dest} \cdot c_{dest}$$
            Substituting the parameters:
            $$I_k = 1 \cdot g_{k-1} + T_{k-1} \cdot I_{k-1}$$
            Here, $c_{src}$ is the particle's color ($g_{k-1}$), $c_{dest}$ is the frame buffer value ($I_{k-1}$), $f_{src}=1$, and $f_{dest}=T_{k-1}$ [S1, unofficial]. This updates the frame buffer to store $I_k$ for the next particle [S1, unofficial].

2.  **Runtime (Eye Scattering Pass):**
    1.  Sort particles by decreasing distance from the eye [S1, unofficial].
    2.  For each particle, compute the scattering toward the eye using the precomputed $I_k$ and the phase function $phase(\omega, \mathbf{l})$, where $\omega$ is the direction to the viewer [S1, unofficial].
    3.  Render particles with standard compositing using the precomputed illumination [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard operations defined for self-containment
def light_color(): return [1.0, 1.0, 1.0]  # Light intensity/color
def phase_function(omega, l): return 1.0   # Scattering phase function
def render_particle(pos, radius, color, alpha, blend_src, blend_dst):
    # Standard GPU rasterization with specified blend factors
    pass

# Preprocessing Pass
frame_buffer = [1.0, 1.0, 1.0]  # White, representing I_0
particles.sort(key=lambda p: distance(p.position, light_position))

for i, p in enumerate(particles):
    # Read incident light I_{k-1} from frame buffer
    incident_light_prev = frame_buffer[:]
    
    # Compute incident light I_k
    I_k = [c * light_color()[j] for j, c in enumerate(incident_light_prev)]
    
    # Compute scattered light contribution g_{k-1} for the recurrence
    scattered_contribution = [a * p.extinction * I_k[j] / (4 * math.pi) for j, a in enumerate(p.albedo)]
    
    # Apply phase function for directionality
    omega = -light_direction  # Direction from light to particle
    scattered_contribution = [c * phase_function(omega, light_direction) for c in scattered_contribution]
    
    # Set render state for blending
    alpha = 1.0 - math.exp(-p.extinction)
    T_prev = math.exp(-particles[i-1].extinction) if i > 0 else 1.0
    
    # Render particle
    render_particle(
        position=p.position,
        radius=p.radius,
        color=scattered_contribution,
        alpha=alpha,
        blend_src=1.0,
        blend_dst=T_prev  # T_{k-1}
    )
    frame_buffer = [c * 1.0 + T_prev * c_prev for c, c_prev in zip(scattered_contribution, incident_light_prev)]

# Runtime Pass
particles.sort(key=lambda p: -distance(p.position, eye_position))
for p in particles:
    omega = normalize(eye_position - p.position)
    final_color = [c * phase_function(omega, light_direction) for c in p.precomputed_color]
    
    render_particle(
        position=p.position,
        radius=p.radius,
        color=final_color,
        alpha=1.0 - math.exp(-p.extinction),
        blend_src=1.0,
        blend_dst=0.0  # Standard compositing for eye scattering
    )
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision in the frame buffer may accumulate errors over many particles [S1, unofficial].
*   **Boundary Conditions:** The frame buffer must be cleared to the correct initial value ($I_0$) before each pass [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ particles, requiring $N$ pixel read-back operations during preprocessing [S1, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory beyond the frame buffer and particle list [S1, unofficial].
*   **Strengths:** Conceptually simple mapping of the recurrence to hardware blending [S1, unofficial].
*   **Limitations:** Pixel read-back introduces synchronization overhead [S1, unofficial].
*   **When to use:** Suitable for real-time implementations where the recurrence must be resolved per particle [S1, unofficial].

## Background
The method was presented in the paper "Real-Time Cloud Rendering" by Dobashi et al. (2001) [S1, unofficial]. It was developed to efficiently handle multiple forward scattering in cloud rendering by leveraging existing GPU hardware features [S1, unofficial]. The technique is specific to the context of particle-based cloud rendering and relies on the specific recurrence relation for forward scattering [S1, unofficial].

## Sources
[S1] Real-Time Cloud Rendering, https://www.cs.unc.edu/techreports/01-005.pdf, fetched 2026-10-28 (unofficial)
