# Beam Optimization

Also called: Beam Optimization for Primary Rays, Ray Batching, Ray Grouping Optimization

## Overview
Beam optimization is a technique to improve the performance of primary rays in volume rendering by grouping adjacent rays into beams and marching them together [S1][S2, unofficial]. It reduces redundant computations for rays far from geometry by treating a group as a single entity until local detail requires splitting [S2, unofficial]. Unlike [Block Culling](5431-block-culling.md) which skips entire blocks, or [Bounding Shape Optimization](7750-bounding-shape-optimization.md) which tightens bounds, beam optimization adapts the granularity of ray evaluation based on proximity to surfaces [S2, unofficial].

## How it works
**Inputs:** A set of primary rays defined by pixel coordinates $(u, v)$ and camera parameters.
**Outputs:** Intersection distances or hit status for each pixel.
**Data Structures:**
- **Beam:** A rectangular group of rays. Defined by top-left pixel $(u_{min}, v_{min})$, bottom-right pixel $(u_{max}, v_{max})$, and the current ray origin $O$ and direction $D$ (typically the center ray of the beam).
- **Distance Field:** A function $SDF(P)$ returning the minimum distance from point $P$ to the scene geometry.

**Parameters:**
- **Beam Size:** Typically $8 \times 8$ pixels [S1].

**Algorithm:**
1. Initialize a queue of beams, initially containing one beam covering the entire image (or a tile).
2. While the queue is not empty:
    a. Pop a beam $B$.
    b. Let $O_B, D_B$ be the origin and direction of the center ray of $B$.
    c. Perform standard Sphere Tracing (marching) along $O_B, D_B$ using $SDF$.
    d. Let $d_{min}$ be the smallest distance to geometry found during the march so far.
    e. Calculate the maximum spatial extent of the beam $W_B$ (e.g., the diagonal length of the beam's pixel footprint at the current step).
    f. If $d_{min} > W_B$, the beam is far from geometry. The entire beam can be advanced by $d_{min}$ without risk of missing intersections for any ray in the group [S2, unofficial].
    g. If $d_{min} \le W_B$, the beam is close to geometry. Split $B$ into four sub-beams (quadrants) and push them to the queue.
    h. If a ray in the beam hits an object (distance $< \epsilon$), mark the corresponding pixels as hit and remove the beam from further processing [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def beam_optimization(rays, sdf, beam_size=8):
    # rays: list of (origin, direction, u, v)
    # Initialize beams
    beams = []
    # Group rays into initial beams
    for i in range(0, len(rays), beam_size * beam_size):
        chunk = rays[i:i + beam_size * beam_size]
        if len(chunk) < beam_size * beam_size:
            continue # Handle edge cases if image not divisible
        beams.append(Beam(chunk, beam_size))
    
    queue = beams
    
    while queue:
        beam = queue.pop(0)
        if beam.is_empty():
            continue
            
        # Center ray of the beam
        center_ray = beam.get_center_ray()
        origin, direction = center_ray
        
        # March along center ray
        t = 0.0
        hit = False
        while t < MAX_DISTANCE:
            dist = sdf(origin + t * direction)
            if dist < EPSILON:
                hit = True
                break
            t += dist
            
            # Check if beam can advance safely
            # W_beam is the max deviation of any ray in beam from center ray at distance t
            w_beam = beam.get_max_deviation(t)
            
            if dist > w_beam:
                # Safe to advance entire beam
                origin += t * direction
                t = 0.0
            else:
                # Too close, split
                sub_beams = beam.split()
                queue.extend(sub_beams)
                break # Stop marching this beam, let children handle it
        
        if hit:
            beam.mark_hit()
```

## Subtopics

### Beam Optimization for Soft Shadows
An extension of beam optimization for primary rays applied to shadow ray computation [S1].
- **Definition:** Shoots a beam of rays from the view-sample position to an area light to find conservative bounds for shadowing [S1].
- **Steps:**
    1. Shoot a beam of four rays per pixel from the view-sample position to the area light [S1].
    2. Find the conservative furthest distance along these rays where no geometry is intersected [S1].
    3. Shorten all shadow rays accordingly based on this conservative furthest distance [S1].
    4. Trace the same beam rays in the reverse direction (from the light source) to find a conservative interval where geometry may be intersected [S1].
- **Pitfall:** Pixels whose entire beams have no intersection with geometry require no additional shadow rays, allowing for significant culling [S1].

## Operating characteristics
- **Time Complexity:** Reduces the number of SDF evaluations for rays far from geometry. Complexity depends on the distribution of geometry; near surfaces, it degrades to per-ray evaluation due to splitting [S2, unofficial].
- **Memory Complexity:** Low overhead, requiring storage for the beam queue and ray states [S1].
- **Strengths:** Effective for sparse scenes or large empty volumes where many rays travel long distances without intersecting [S2, unofficial].
- **Limitations:** In dense geometry, frequent splitting can introduce overhead. Beam size must be tuned; too large causes missed intersections, too small reduces benefit [S2, unofficial].
- **When to use:** Primary ray tracing in sparse voxel DAGs or large open volumes [S1].
- **When not to use:** Dense, complex geometry where rays intersect frequently, or when memory bandwidth is the bottleneck rather than computation [S2, unofficial].

## Background
Beam optimization was introduced by Laine and Karras in the context of high-resolution sparse voxel DAGs [S1]. It was originally described in [2010a] for primary rays and extended in [2010b] for soft shadows and ambient occlusion [S1]. The technique leverages the coherence of adjacent rays in image-order rendering to reduce redundant distance field queries [S1][S2, unofficial].

## Sources
[S1] High resolution sparse voxel DAGs, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S2] Sphere Tracing GPU An evaluation of the Sphere Tracing algorithm and a GPU designed to run it, https://odr.chalmers.se/items/37b0e53f-fc76-4b0e-96d8-8e864573ada9/full, fetched 2026-10-29 (unofficial)
