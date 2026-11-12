# Billboards and particles
Also called: Billboards, Particles, classical billboards, cross billboards

## Overview
Billboards and particles are a low-cost, artist-driven technique for approximating participating media effects or static geometry (such as forests) by rendering 2D semi-transparent textures that always face the camera [S2, unofficial][S1]. Unlike volume-based splatting methods which compute density integration, this method relies on pre-defined textures placed at static locations or animated particles that move according to rules, making it suitable for real-time industrial simulators where performance is prioritized over physical accuracy [S1][S2, unofficial]. It is distinct from Gaussian splatting or ray-marching as it does not reconstruct a continuous medium but rather composites discrete, camera-facing quads [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of billboards or particles. Each entity $i$ is defined by its position $\mathbf{p}_i$, a 2D texture $T_i$ (color and alpha), and potentially a radius or scale $s_i$. For particles, additional state includes velocity $\mathbf{v}_i$ and lifecycle parameters (birth/death times) [S2, unofficial].
*   **Output:** A 2D image where textures are composited using alpha blending [S1][S2, unofficial].

**Data Structures:**
*   **Billboard:** Struct containing `position` (vec3), `texture_id` (index), `scale` (float), and `rotation` (float, usually 0 as they face camera).
*   **Particle:** Struct containing `position` (vec3), `velocity` (vec3), `texture_id` (index), `scale` (float), `age` (float), `max_age` (float), and `alive` (bool).

**Steps:**
1.  **Initialization:** Define the scene with static billboards at artist-specified locations or initialize a pool of particles with random positions and velocities [S2, unofficial].
2.  **Simulation (Particles Only):** For each particle $i$:
    *   Update position: $\mathbf{p}_i \leftarrow \mathbf{p}_i + \mathbf{v}_i \cdot \Delta t$ [S2, unofficial].
    *   Update age: $age_i \leftarrow age_i + \Delta t$ [S2, unofficial].
    *   Check lifecycle: If $age_i > max\_age$, mark particle as dead or reset it [S2, unofficial].
3.  **View Transformation:** For each visible billboard or particle $i$:
    *   Compute the view vector $\mathbf{v}_{view}$ from the camera to $\mathbf{p}_i$.
    *   Construct a rotation matrix $R$ such that the billboard's normal aligns with $\mathbf{v}_{view}$ (billboarding) [S1][S2, unofficial].
    *   For cross billboards, generate 2 to 4 quads with normals distributed around the view direction (e.g., orthogonal axes) [S1].
4.  **Rasterization:**
    *   Project the transformed quads onto the screen space [S1].
    *   For each pixel covered by a quad, sample the texture $T_i$ to get color $\mathbf{c}$ and opacity $\alpha$ [S2, unofficial].
5.  **Compositing:**
    *   Sort billboards/particles by depth (back-to-front) [S1].
    *   Apply standard alpha blending: $\mathbf{C}_{out} = \mathbf{C}_{in} \cdot (1 - \alpha) + \mathbf{c} \cdot \alpha$ [S1][S2, unofficial].

**Parameters:**
*   **Number of Billboards:** Determined by scene complexity; dense forests require millions, which is expensive [S1].
*   **Cross Billboard Count:** Typically 2 to 4 quads per object [S1].
*   **Particle Count:** High, often thousands, to simulate density [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Parallax Artifacts:** Classical billboards show no parallax when the camera moves, making them acceptable only for axis-symmetrical objects or grazing view angles [S1].
*   **Popping:** In sparse billboard-based forests, a tree slightly behind another may "pop" in front due to small camera motions, requiring careful sparse placement [S1].
*   **Grazing Angle Artifacts:** Cross billboards exhibit visible seams or artifacts when any of the constituent quads is viewed at a grazing angle [S1].
*   **Lighting Dependency:** Billboards are artist-dependent and cannot react to dynamic lighting changes in the environment [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_billboards(billboards, camera):
    # Sort by depth from camera
    sorted_billboards = sort_by_depth(billboards, camera.position)
    
    image = empty_image()
    
    for bb in sorted_billboards:
        # Calculate billboard orientation to face camera
        # [G] Standard billboarding: align normal with view vector
        view_dir = normalize(camera.position - bb.position)
        rotation_matrix = look_at_rotation(bb.position, camera.position, up_vector=[0,1,0])
        
        # Transform quad vertices
        quad_vertices = transform_quad(bb.quad_template, rotation_matrix, bb.position, bb.scale)
        
        # Rasterize and blend
        for pixel in pixels_covered_by(quad_vertices):
            tex_coord = get_texture_coordinate(pixel, quad_vertices)
            color, alpha = sample_texture(bb.texture, tex_coord)
            
            # Alpha blend
            existing_color = image[pixel].color
            existing_alpha = image[pixel].alpha
            
            new_color = existing_color * (1 - alpha) + color * alpha
            new_alpha = existing_alpha + alpha * (1 - existing_alpha)
            
            image[pixel].color = new_color
            image[pixel].alpha = new_alpha

    return image

def update_particles(particles, dt):
    for p in particles:
        if p.alive:
            p.position += p.velocity * dt
            p.age += dt
            if p.age > p.max_age:
                p.alive = False
                # Reset or remove particle logic here
    return [p for p in particles if p.alive]
```

## Subtopics

### Classical Billboards
Single textured quad facing the camera. Cheapest, but suffers from severe parallax artifacts. Choose for axis-symmetrical objects or distant, sparse geometry [S1].

### Cross Billboards
2-4 crossing quads per object. Reduces parallax but introduces grazing angle artifacts. Choose for slightly more robust static geometry representation where performance is still critical [S1].

### Particle Systems
Dynamic billboards with lifecycle rules. Used for transient effects like smoke, fire, or dust. Choose for animated participating media approximations [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of billboards/particles. Sorting adds $O(N \log N)$ [S1].
*   **Memory Complexity:** $O(N)$ for storing particle/billboard data plus texture memory [S1][S2, unofficial].
*   **Performance:** Extremely fast for real-time rendering, making it the preferred choice in industrial simulators despite visual limitations [S1].
*   **Strengths:** Low computational cost, simple implementation, effective for static or pre-defined effects [S1][S2, unofficial].
*   **Limitations:** No parallax (classical), popping artifacts, grazing angle artifacts (cross), artist-dependent, no dynamic lighting interaction [S1][S2, unofficial].
*   **When to Use:** Real-time rendering of forests, static volumetric effects (light beams), or simple particle effects where physical accuracy is not required [S1][S2, unofficial].
*   **When Not to Use:** Dynamic environments with changing lighting, dense media requiring accurate scattering, or when camera movement is significant and parallax is visible [S1][S2, unofficial].

## Background
Billboards have been a common tool for real-time rendering of forests since at least the late 1990s [S1]. The technique was reviewed in the context of efficient forest rendering, highlighting its dominance in industrial simulators due to low cost [S1]. The application of billboards and particles to participating media (like smoke or light beams) was further detailed in research on real-time light transport, noting their utility in approximating complex scattering effects with simple textures [S2, unofficial].

## Sources
[S1] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
[S2] PDF Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
