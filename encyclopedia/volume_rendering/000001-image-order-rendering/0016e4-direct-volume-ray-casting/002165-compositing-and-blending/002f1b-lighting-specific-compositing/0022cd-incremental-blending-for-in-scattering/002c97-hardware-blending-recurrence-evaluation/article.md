# Hardware Blending Recurrence Evaluation

Also called: Frame buffer read back blending

## Overview
Hardware Blending Recurrence Evaluation is a technique for evaluating the multiple forward scattering recurrence $I_k = g_{k-1} + T_{k-1} I_{k-1}$ by mapping the mathematical operation to GPU hardware blending and pixel read-back operations [S1, unofficial]. It solves the problem of efficiently computing the incident light on a particle in a participating medium by leveraging the frame buffer as an accumulator. It differs from [Incremental Lighting for In-Scattering](3401-incremental-lighting-for-in-scattering.md) by using explicit pixel read-back to resolve the recurrence dependency, whereas its sibling uses a different accumulation strategy.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of particles $P$, each with extinction $\tau_k$, albedo $a_k$, radius $r_k$, and position. Light direction $\mathbf{l}$ and light color/intensity.
*   **Outputs:** The scattered light contribution $c$ for each particle, which is then rendered toward the eye.

**Data Structures:**
*   **Frame Buffer:** Stores the accumulated incident light intensity $I$. Initialized to white (representing $I_0 = 1$ or full intensity).
*   **Particle List:** Sorted by distance to the light source for the preprocessing pass.

**Parameters:**
*   **Source Blend Factor ($f_{src}$):** $1$ [S1, unofficial].
*   **Destination Blend Factor ($f_{dest}$):** $T_{k-1} = \exp(-\tau_{k-1})$ (transparency of the previous particle) [S1, unofficial].
*   **Texture/Fragment Mode:** Modulate [S1, unofficial].

**Steps:**
1.  **Preprocessing (Illumination Pass):**
    1.  Clear the frame buffer to white. This represents the initial incident light $I_0$ [S1, unofficial].
    2.  Sort particles $p_k$ by increasing distance to the light source [S1, unofficial].
    3.  For each particle $p_k$ in sorted order:
        a.  **Read Back:** Read the pixel color $x$ at the projection of the particle's center from the frame buffer. This value represents $I_{k-1}$, the incident light on the previous particle (or the transmitted light through it) [S1, unofficial].
        b.  **Compute Incident Light:** Calculate $I_k = \text{color}(x) \times \text{light\_color}$. This is the light incident on the current particle $p_k$ [S1, unofficial].
        c.  **Compute Scattered Color:** Calculate the scattered light contribution $g_{k-1}$ (denoted as $c$ in the source code logic, but representing the source term for the next step) using the formula:
            $$c = a_k \cdot \tau_k \cdot I_k / 4\pi$$
            where $a_k$ is albedo, $\tau_k$ is extinction, and $I_k$ is incident light [S1, unofficial].
        d.  **Set Render Parameters:**
            *   Set the particle's alpha to $1 - \exp(-\tau_k)$ [S1, unofficial].
            *   Set the particle's color to $c$ (modulated by phase function if applicable, see code) [S1, unofficial].
        e.  **Render and Blend:** Render the particle. The hardware blending equation is:
            $$c_{result} = f_{src} \cdot c_{src} + f_{dest} \cdot c_{dest}$$
            Substituting the parameters:
            $$I_k = 1 \cdot g_{k-1} + T_{k-1} \cdot I_{k-1}$$
            Here, $c_{src}$ is the particle's color ($g_{k-1}$), $c_{dest}$ is the frame buffer value ($I_{k-1}$), $f_{src}=1$, and $f_{dest}=T_{k-1}$ [S1, unofficial]. This updates the frame buffer to store $I_k$ for the next particle [S1, unofficial].

2.  **Runtime (Eye Scattering Pass):**
    1.  Sort particles by decreasing distance from the eye [S1, unofficial].
    2.  For each particle, compute the scattering toward the eye using the precomputed $I_k$ and the phase function $phase(\omega, \mathbf{l})$, where $\omega$ is the direction to the viewer [S1, unofficial].
    3.  Render particles with standard compositing (e.g., [Direct Volume Ray Casting](52-name.md) principles) using the precomputed illumination.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing Pass
frame_buffer.clear(value=1.0) # White, representing I_0
particles.sort(key=lambda p: distance(p.position, light_position))

for p in particles:
    # Read incident light I_{k-1} from frame buffer
    # Note: In a real implementation, this requires a read-back operation
    # which is expensive. The source implies this is done per particle.
    incident_light_prev = frame_buffer.read_pixel(p.projected_center)
    
    # Compute incident light I_k
    I_k = incident_light_prev * light_color
    
    # Compute scattered light contribution g_{k-1} for the recurrence
    # Formula: c = a_k * tau_k * I_k / 4pi
    scattered_contribution = p.albedo * p.extinction * I_k / (4 * math.pi)
    
    # Apply phase function for directionality (if needed for the recurrence term)
    # The source code snippet shows: c = pk.color * phase(omega, l)
    # However, the recurrence term g_{k-1} is the light scattered TO the next particle.
    # The direction to the next particle is along the light direction l.
    # So phase(omega, l) where omega is direction to next particle (l) is phase(l, l).
    # The source code snippet seems to mix the runtime phase function with the preprocess.
    # Based on Eq 4 and text: g_{k-1} is the scattered light.
    # We assume the phase function is applied to the scattered light direction.
    # For forward scattering along l, the phase function is phase(l, l).
    scattered_contribution *= phase_function(l, l) 
    
    # Set render state for blending
    # Source color is the scattered contribution g_{k-1}
    # Destination factor is T_{k-1} = exp(-tau_{k-1})
    # Source factor is 1
    
    alpha = 1.0 - math.exp(-p.extinction)
    
    # Render particle
    # The blending operation updates the frame buffer:
    # I_k = 1 * g_{k-1} + T_{k-1} * I_{k-1}
    render_particle(
        position=p.position,
        radius=p.radius,
        color=scattered_contribution,
        alpha=alpha,
        blend_src=1.0,
        blend_dst=math.exp(-p.extinction) # T_{k-1}
    )

# Runtime Pass
particles.sort(key=lambda p: -distance(p.position, eye_position))
for p in particles:
    # Use precomputed I_k
    # Compute scattering toward eye
    omega = eye_position - p.position
    omega = normalize(omega)
    
    # The source code snippet: c = pk.color * phase(omega, l)
    # pk.color was set to a_k * tau_k * I_k / 4pi
    # So final color is (a_k * tau_k * I_k / 4pi) * phase(omega, l)
    final_color = p.precomputed_color * phase_function(omega, l)
    
    render_particle(
        position=p.position,
        radius=p.radius,
        color=final_color,
        alpha=1.0 - math.exp(-p.extinction),
        blend_src=1.0,
        blend_dst=0.0 # Standard alpha blending for eye scattering, not recurrence
    )
```

**Edge Cases and Pitfalls:**
*   **Performance:** Reading back pixels from the frame buffer for every particle is extremely slow on modern GPUs due to cache coherency and synchronization barriers. This method is primarily of theoretical or historical interest for understanding the recurrence mapping [S1, unofficial].
*   **Precision:** Floating-point precision in the frame buffer may accumulate errors over many particles.
*   **Boundary Conditions:** The frame buffer must be cleared to the correct initial value ($I_0$) before each pass.

## Operating characteristics
*   **Time Complexity:** $O(N)$ particles, but with a high constant factor due to $N$ pixel read-back operations.
*   **Memory Complexity:** $O(1)$ additional memory beyond the frame buffer and particle list.
*   **Strengths:** Conceptually simple mapping of the recurrence to hardware blending.
*   **Limitations:** Pixel read-back is a bottleneck. Not suitable for real-time performance with large particle counts.
*   **When to use:** Only for educational purposes or very small particle counts where correctness of the recurrence evaluation is prioritized over performance.

## Background
The method was presented in the paper "Real-Time Cloud Rendering" by Dobashi et al. (2001) [S1, unofficial]. It was developed to efficiently handle multiple forward scattering in cloud rendering by leveraging existing GPU hardware features. The technique is specific to the context of particle-based cloud rendering and relies on the specific recurrence relation for forward scattering.

## Sources
[S1] Real-Time Cloud Rendering, https://www.cs.unc.edu/techreports/01-005.pdf, fetched 2026-10-28 (unofficial)
