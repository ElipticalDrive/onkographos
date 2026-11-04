# Lambertian Shading with Packet Architecture

Also called: Batch diffuse shading, Batch diffuse shading with packet architecture, Packet Ray Tracing with Lambertian Shading, Packet Shading

## Overview
This method performs fast, shadow-less Lambertian shading for isosurfaces extracted from thresholded volumes by batching ray generation and shading operations using a packet architecture [S1, S2][S3, unofficial]. It is distinguished from sibling methods by explicitly avoiding deferred normal computation to prevent prohibitive memory and computational costs associated with per-ray cell storage or repeated neighbor-finding [S1, S2][S3, unofficial]. While the underlying ray traversal remains single-ray based, the shading phase aggregates rays into packets to exploit parallelism [S1, S2][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of rays intersecting the isosurface, derived from the parent [Shading and Frame Buffer Filling](1808-shading-and-frame-buffer-filling.md) stage [G].
*   **Output:** Shaded color values written to the frame buffer [S1, S2][S3, unofficial].

**Data Structures:**
*   **Ray Packet:** A container holding a group of rays, where each ray stores origin, direction, traversal state, hit point, and computed normal [G]. The exact internal layout is not specified in the sources [S1, S2][S3, unofficial].
*   **Normal Vector:** Computed immediately upon intersection, not deferred [S1, S2][S3, unofficial].

**Steps:**
1.  **Ray Generation:** Generate rays for the image plane and group them into packets [S1, S2][S3, unofficial].
2.  **Traversal:** Perform ray traversal against the volume data structure (e.g., octree) using single-ray processing, as traversal does not employ packets [S1, S2][S3, unofficial].
3.  **Intersection Detection:** Identify when a ray intersects the isosurface during traversal [G].
4.  **Normal Computation:** Compute the surface normal at the intersection point immediately [S1, S2][S3, unofficial]. Deferring this computation is prohibited due to the prohibitive cost of storing each cell per ray or repeating the neighbor-finding process [S1, S2][S3, unofficial].
5.  **Batch Shading:** Perform diffuse shading in batch for all rays in the current packet [S1, S2][S3, unofficial].

**Formula:**
$$C = k_d \max(0, N \cdot L) + k_a$$
*   $C$: Final color vector
*   $k_d$: Diffuse reflection coefficient (scalar or vector)
*   $N$: Surface normal at the intersection point
*   $L$: Light direction vector
*   $k_a$: Ambient reflection coefficient (scalar or vector)
*   *Note:* Exact coefficients and light direction are not specified in the sources [S1, S2][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize(v):
    length = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    return [v[0]/length, v[1]/length, v[2]/length] if length > 0 else [0, 0, 1]

def compute_normal(point, volume_data):
    # [G] Central finite difference gradient of the scalar field
    eps = 1e-4
    gx = volume_data(point + [eps, 0, 0]) - volume_data(point - [eps, 0, 0])
    gy = volume_data(point + [0, eps, 0]) - volume_data(point - [0, eps, 0])
    gz = volume_data(point + [0, 0, eps]) - volume_data(point - [0, 0, eps])
    return [gx, gy, gz]

def batch_lambertian_shading(rays, kd, ka, L):
    # Group rays into packets for batch processing
    packet_size = 16  # [G] Typical packet size for parallel execution
    packets = [rays[i:i+packet_size] for i in range(0, len(rays), packet_size)]
    
    for packet in packets:
        # Traverse each ray individually; traversal does not use packets [S1, S2][S3, unofficial]
        for ray in packet:
            # [G] Ray casting against the octree volume is performed here
            # Intersection detection and hit point calculation occur during traversal
            if ray.hit:
                # Compute normal immediately at the intersection point
                ray.normal = compute_normal(ray.hit_point, ray.volume_data)
                ray.normal = normalize(ray.normal)
        
        # Perform diffuse shading in batch for the packet [S1, S2][S3, unofficial]
        for ray in packet:
            if ray.hit:
                # Simple Lambertian shading with no shadows [S1, S2][S3, unofficial]
                diff = max(0.0, ray.normal[0]*L[0] + ray.normal[1]*L[1] + ray.normal[2]*L[2])
                ray.color = [kd[i]*diff + ka[i] for i in range(3)]
```

**Parameters:**
*   **Shading Model:** Simple Lambertian, no shadows [S1, S2][S3, unofficial].
*   **Packet Size:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Deferred Normal Computation:** Do not defer normal computation. Storing each cell per ray or repeating neighbor-finding is prohibitively costly [S1, S2][S3, unofficial].
*   **Traversal Architecture:** Traversal does not use packets. Using packet traversal is not part of this method [S1, S2][S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The packet architecture for shading likely delivers a speedup over a more conventional single-ray tracer [S1, S2][S3, unofficial].
*   **Strengths:** Fast ray casting of isosurfaces; batch processing of shading operations [S1, S2][S3, unofficial].
*   **Limitations:** No shadows; normal computation must be done immediately, limiting flexibility in shading pipelines [S1, S2][S3, unofficial].
*   **When to use:** When fast, shadow-less Lambertian shading of isosurfaces is required [S1, S2][S3, unofficial].
*   **When not to use:** When shadow rendering is required [S1, S2][S3, unofficial].

## Background
This method is described in the context of interactive isosurface ray tracing of large octree volumes [S1, S2][S3, unofficial]. It is part of a pipeline that includes data compression, octree construction, and ray traversal [S1, S2][S3, unofficial]. The approach emphasizes performance for interactive applications by simplifying the shading model and optimizing the shading phase through batching [S1, S2][S3, unofficial].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S3] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
