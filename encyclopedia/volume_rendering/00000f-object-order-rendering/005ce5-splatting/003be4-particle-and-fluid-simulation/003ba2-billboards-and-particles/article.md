# Billboards and particles
Also called: Billboards, Particles, classical billboards, cross billboards

## Overview
Billboards and particles are a low-cost technique [S1] for approximating participating media effects or static geometry (such as forests) by rendering 2D semi-transparent textures that always face the camera [S2, unofficial][S1]. Unlike volume-based splatting methods which compute density integration, this method relies on pre-defined textures placed at static locations or animated particles that move according to rules, making it suitable for real-time industrial simulators where performance is prioritized over physical accuracy [S1][S2, unofficial]. It is distinct from Gaussian splatting or ray-marching as it does not reconstruct a continuous medium but rather composites discrete, camera-facing quads [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of billboards or particles. Each entity $i$ is defined by its position $\mathbf{p}_i$, a 2D texture $T_i$ (color and alpha), and a scale $s_i$ [G]. For particles, additional state includes velocity $\mathbf{v}_i$ and lifecycle parameters (birth/death times) [G].
*   **Output:** A 2D image where textures are composited using alpha blending [S1][S2, unofficial].

**Data Structures:**
*   **Billboard:** Struct containing `position` (vec3), `texture_id` (index), `scale` (float), and `rotation` (float, usually 0 as they face camera) [G].
*   **Particle:** Struct containing `position` (vec3), `velocity` (vec3), `texture_id` (index), `scale` (float), `age` (float), `max_age` (float), and `alive` (bool) [G].

**Steps:**
1.  **Initialization:** Define the scene with static billboards at artist-specified locations or initialize a pool of particles with positions and velocities [G].
2.  **Simulation (Particles Only):** For each particle $i$:
    *   Update position: $\mathbf{p}_i \leftarrow \mathbf{p}_i + \mathbf{v}_i \cdot \Delta t$ [G].
    *   Update age: $age_i \leftarrow age_i + \Delta t$ [G].
    *   Check lifecycle: If $age_i > max\_age$, mark particle as dead or reset it [G].
3.  **View Transformation:** For each visible billboard or particle $i$:
    *   Compute the view vector $\mathbf{v}_{view}$ from the camera to $\mathbf{p}_i$ [G].
    *   Construct a rotation matrix $R$ such that the billboard's normal aligns with $\mathbf{v}_{view}$ (billboarding) [G].
    *   For cross billboards, generate 2 to 4 quads with normals distributed around the view direction (e.g., orthogonal axes) [S1].
4.  **Rasterization:**
    *   Project the transformed quads onto the screen space [G].
    *   For each pixel covered by a quad, sample the texture $T_i$ to get color $\mathbf{c}$ and opacity $\alpha$ [G].
5.  **Compositing:**
    *   Sort billboards/particles by depth (back-to-front) [S1].
    *   Apply standard alpha blending: $\mathbf{C}_{out} = \mathbf{C}_{in} \cdot (1 - \alpha) + \mathbf{c} \cdot \alpha$ [S1][S2, unofficial].

**Parameters:**
*   **Number of Billboards:** Determined by scene complexity; dense forests require millions, which is expensive [S1].
*   **Cross Billboard Count:** Typically 2 to 4 quads per object [S1].
*   **Particle Count:** High, often thousands, to simulate density [G].

**Edge Cases and Pitfalls:**
*   **Parallax Artifacts:** Classical billboards show no parallax when the camera moves, making them acceptable only for axis-symmetrical objects or grazing view angles [S1].
*   **Popping:** In sparse billboard-based forests, a tree slightly behind another may "pop" in front due to small camera motions, requiring careful sparse placement [S1].
*   **Grazing Angle Artifacts:** Cross billboards exhibit visible seams or artifacts when any of the constituent quads is viewed at a grazing angle [S1].
*   **Lighting Dependency:** Billboards are artist-dependent and cannot react to dynamic lighting changes in the environment [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b):
    return sum(x*y for x,y in zip(a,b))

def cross(a, b):
    return [a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0]]

def normalize(v):
    mag = math.sqrt(sum(x*x for x in v))
    return [x/mag for x in v] if mag > 0 else [0,0,0]

def render_billboards(billboards, camera):
    # Sort by depth from camera (back-to-front)
    sorted_billboards = sorted(billboards, key=lambda bb: sum((bb.position[i]-camera.position[i])**2 for i in range(3)), reverse=True)
    
    # Initialize output image with zeros
    image = [[(0.0, 0.0, 0.0, 0.0) for _ in range(camera.width)] for _ in range(camera.height)]
    
    for bb in sorted_billboards:
        # [G] Standard billboarding: align normal with view vector
        view_dir = normalize([camera.position[i]-bb.position[i] for i in range(3)])
        
        # [G] Construct rotation matrix to face camera
        up = [0, 1, 0]
        right = normalize(cross(view_dir, up))
        up = cross(right, view_dir)
        
        # Transform quad vertices to world space
        quad_vertices = []
        for v in bb.quad_template:
            rx = dot(right, v)*bb.scale + bb.position[0]
            ry = dot(up, v)*bb.scale + bb.position[1]
            rz = dot(view_dir, v)*bb.scale + bb.position[2]
            quad_vertices.append([rx, ry, rz])
        
        # Rasterize and blend
        for py in range(camera.height):
            for px in range(camera.width):
                # [G] Bounding box check for coverage
                if not (min(v[0] for v in quad_vertices) <= px <= max(v[0] for v in quad_vertices) and
                        min(v[1] for v in quad_vertices) <= py <= max(v[1] for v in quad_vertices)):
                    continue
                # [G] Barycentric interpolation for texture coordinates
                min_x, max_x = min(v[0] for v in quad_vertices), max(v[0] for v in quad_vertices)
                min_y, max_y = min(v[1] for v in quad_vertices), max(v[1] for v in quad_vertices)
                u = (px - min_x) / (max_x - min_x) if max_x != min_x else 0.5
                v_uv = (py - min_y) / (max_y - min_y) if max_y != min_y else 0.5
                
                # [G] Sample texture
                color = (0.8, 0.8, 0.8)
                alpha = 0.5
                
                # Alpha blend
                existing_color, existing_alpha = image[py][px]
                new_color = [existing_color[j]*(1-alpha) + color[j]*alpha for j in range(3)]
                new_alpha = existing_alpha + alpha*(1-existing_alpha)
                image[py][px] = (new_color, new_alpha)

    return image

def update_particles(particles, dt):
    for p in particles:
        if p.alive:
            p.position = [p.position[i] + p.velocity[i]*dt for i in range(3)]
            p.age += dt
            if p.age > p.max_age:
                p.alive = False
    return [p for p in particles if p.alive]
```

## Subtopics

### Classical Billboards
Single textured quad facing the camera. Cheapest, but suffers from severe parallax artifacts. Choose for axis-symmetrical objects or distant, sparse geometry [S1].

### Cross Billboards
2-4 crossing quads per object. Reduces parallax but introduces grazing angle artifacts. Choose for slightly more robust static geometry representation where performance is still critical [S1].

### Particle Systems
Dynamic billboards with lifecycle rules. Used for transient effects like smoke, fire, or dust. Choose for animated participating media approximations [S2, unofficial].

### Multi-View Billboard Rendering
Rendering technique using multi-texturing with textures from the two closest cameras, blending billboards back-to-front using the Porter-Duff 'over' operator [S3]. Traverse cells back-to-front [S3]. For each cell, render the billboard using multi-texturing with weighted textures from the two closest cameras [S3]. Blend the RGBA billboard textures using the Porter-Duff 'over' operator [S3].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of billboards/particles. Sorting adds $O(N \log N)$ [S1].
*   **Memory Complexity:** $O(N)$ for storing particle/billboard data plus texture memory [S1][S2, unofficial].
*   **Performance:** Extremely fast for real-time rendering, making it the preferred choice in industrial simulators despite visual limitations [S1].
*   **Strengths:** Low computational cost, simple implementation, effective for static or pre-defined effects [S1][S2, unofficial].
*   **Limitations:** No parallax (classical), popping artifacts, grazing angle artifacts (cross), artist-dependent, no dynamic lighting interaction [S1][S2, unofficial].
*   **When to Use:** Real-time rendering of forests, static volumetric effects (light beams), or simple particle effects where physical accuracy is not required [S1][S2, unofficial].
*   **When Not to Use:** Dynamic environments with changing lighting, dense media requiring accurate scattering, or when camera movement is significant and parallax is visible [S1][S2, unofficial].

## Background
Billboards have been a common tool for real-time rendering of forests since at least the late 1990s [S1]. The technique was reviewed in the context of efficient forest rendering, highlighting its dominance in industrial simulators due to low cost [S1]. The application of billboards and particles to participating media (like smoke or light beams) was further detailed in research on real-time light transport, noting their utility in approximating complex scattering effects with simple textures [S2, unofficial]. Volumetric reconstruction and interactive rendering of trees from photographs also utilized these techniques, demonstrating their adaptability to photographic data [S3].

## Sources
[S1] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
[S2] PDF Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
[S3] Volumetric reconstruction and interactive rendering of trees from photographs, https://doi.org/10.1145/1015706.1015785, fetched 2026-10-26
