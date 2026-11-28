# Ray Tracing-Based Image Warping for Remote Volume Rendering

Also called: Object-space splatting with ray tracing, Sphere-based reprojection

## Overview
This method decouples volume rendering computation from display by transmitting object-space samples (position and color) from a server to a client, where they are rendered as semi-transparent spheres using first-hit ray tracing [S1, unofficial]. It replaces the rasterized point primitives of the base image-warping method to eliminate reprojection artifacts caused by fixed-size point primitives, effectively implementing a distance-aware splatting technique [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume data, camera parameters (origin $o$, direction $\tilde{d}$), and a representative depth heuristic value $t_i$ derived from the server-side ray marcher [S1, unofficial].
*   **Output:** A stream of compacted object-space samples (packed 64-bit positions and radii) and associated volume-rendered colors sent to the client, which produces the final image via ray tracing [S1, unofficial].

**Data Structures:**
*   **Server Output Buffer:** A screen-sized buffer storing object-space samples [S1, unofficial]. Each sample consists of a quantized 64-bit value packing the object-space coordinate $p$ and the footprint radius [S1, unofficial], and an associated volume-rendered color value [S1, unofficial].
*   **Client Data Structure:** A list of semi-transparent spheres defined by center position $p$ and radius $r$, with a constant color attribute [S1, unofficial].

**Steps:**
1.  **Server: Ray Marching and Depth Estimation.** Render the volume using a ray marcher [S1, unofficial]. For each pixel, determine a representative depth value $t_i$ using a heuristic (e.g., from Zellmann et al.) [S1, unofficial].
2.  **Server: Compute Object-Space Coordinate.** Calculate the object-space coordinate $p$ for the representative depth using the formula:
    $$ p = o + \tilde{d} t_i $$
    where $o$ is the ray origin, $\tilde{d}$ is the direction vector, and $t_i$ is the ray parameter associated with the representative depth [S1, unofficial].
3.  **Server: Assign Footprint Radius.** Determine the radius $r$ for the sample: if the ray hit the volume and integrated a color with non-zero opacity, $r = 0.5 \times \text{voxel\_diagonal}$; otherwise, $r = 0$ [S1, unofficial].
4.  **Server: Quantize and Pack.** Quantize the object-space coordinate $p$ and pack it together with the radius $r$ into a single 64-bit value [S1, unofficial].
5.  **Server: Compact Buffer.** Move all samples with a radius of zero to one side of the buffer [S1, unofficial]. Send only the samples with non-zero radius to the client to reduce bandwidth [S1, unofficial].
6.  **Client: Receive Data.** Receive the buffers containing the packed object-space samples and their associated colors [S1, unofficial].
7.  **Client: Build BVH.** Reinterpret the received object-space samples as semi-transparent spheres [S1, unofficial]. Build a Bounding Volume Hierarchy (BVH) over these sphere positions using the LBVH algorithm [S1, unofficial].
8.  **Client: Ray Trace.** Render the point cloud using first-hit ray tracing [S1, unofficial]. Each sphere is colorized according to its designated pixel color with no additional shading [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def server_render(volume, ray_origins, ray_directions, voxel_size):
    samples = []
    for o, d in zip(ray_origins, ray_directions):
        # Step 1: Volume marching heuristic to find t_i, hit, and color [G]
        t_i, hit, color = volume_march(o, d, volume)
        if not hit or color.opacity == 0:
            continue
        # Step 2: Compute object-space coordinate
        p = o + d * t_i
        # Step 3: Assign footprint radius
        voxel_diagonal = voxel_size * math.sqrt(3)
        r = 0.5 * voxel_diagonal if hit and color.opacity > 0 else 0.0
        # Step 4: Quantize and pack into 64-bit value
        packed = pack_64bit(p, r)
        samples.append((packed, color))
    # Step 5: Compact buffer (move zero-radius samples to one side)
    compacted = [(p, c) for p, c in samples if unpack_radius(p) > 0]
    return compacted

def volume_march(o, d, volume):
    # Standard volume ray marching stub returning (t_i, hit, color) [G]
    return 1.0, True, (1.0, 0.5, 0.2, 0.8)

def pack_64bit(p, r):
    # Quantize coordinates and radius, pack into 64-bit integer [G]
    qx, qy, qz = int(p.x * 1000), int(p.y * 1000), int(p.z * 1000)
    qr = int(r * 1000)
    return (qx << 48) | (qy << 32) | (qz << 16) | qr

def unpack_64bit(packed):
    # Extract position and radius from 64-bit value [G]
    qx = (packed >> 48) & 0xFFFF
    qy = (packed >> 32) & 0xFFFF
    qz = (packed >> 16) & 0xFFFF
    qr = packed & 0xFFFF
    return (qx / 1000.0, qy / 1000.0, qz / 1000.0), qr / 1000.0

def client_render(received_samples):
    # Step 6: Receive data (already in received_samples)
    # Step 7: Reinterpret as spheres and build BVH
    spheres = []
    for packed, color in received_samples:
        p, r = unpack_64bit(packed)
        spheres.append({'center': p, 'radius': r, 'color': color})
    # Step 7: Build LBVH from positions [G]
    bvh = build_bvh(spheres)
    # Step 8: Ray trace
    image = []
    for pixel in screen_pixels:
        ray = {'origin': pixel.origin, 'direction': pixel.direction}
        hit = first_hit_query(bvh, ray)
        if hit:
            image.append(hit.color)
        else:
            image.append((0.0, 0.0, 0.0, 0.0))
    return image

def build_bvh(spheres):
    # Standard LBVH construction algorithm [G]
    if not spheres:
        return None
    # Simplified recursive BVH builder for demonstration
    center = sum(s['center'] for s in spheres) / len(spheres)
    left = [s for s in spheres if (s['center'][0] - center[0]) < 0]
    right = [s for s in spheres if (s['center'][0] - center[0]) >= 0]
    return {'left': build_bvh(left), 'right': build_bvh(right), 'spheres': spheres}

def first_hit_query(bvh, ray):
    # Standard first-hit ray tracing query [G]
    if not bvh:
        return None
    closest = None
    min_t = float('inf')
    for s in bvh['spheres']:
        t = ray_sphere_intersect(ray, s)
        if t is not None and t < min_t:
            min_t = t
            closest = s
    if bvh['left']:
        t_l = ray_sphere_intersect(ray, bvh['left'])
        if t_l is not None and t_l < min_t:
            closest = bvh['left']
            min_t = t_l
    if bvh['right']:
        t_r = ray_sphere_intersect(ray, bvh['right'])
        if t_r is not None and t_r < min_t:
            closest = bvh['right']
    return closest

def ray_sphere_intersect(ray, sphere):
    # Standard ray-sphere intersection [G]
    oc = (ray['origin'][0] - sphere['center'][0], ray['origin'][1] - sphere['center'][1], ray['origin'][2] - sphere['center'][2])
    a = ray['direction'][0]**2 + ray['direction'][1]**2 + ray['direction'][2]**2
    b = 2.0 * (oc[0]*ray['direction'][0] + oc[1]*ray['direction'][1] + oc[2]*ray['direction'][2])
    c = oc[0]**2 + oc[1]**2 + oc[2]**2 - sphere['radius']**2
    discriminant = b**2 - 4*a*c
    if discriminant < 0:
        return None
    t = (-b - math.sqrt(discriminant)) / (2*a)
    return t if t > 0 else None

screen_pixels = [{'origin': (0, 0, 0), 'direction': (0, 0, 1)}]
```

**Parameters:**
*   **Footprint Radius:** Set to half the size of a voxel’s diagonal for valid volume hits; zero otherwise [S1, unofficial].
*   **Data Packing:** Object space coordinate and radius are packed into 64 bits [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Rasterization Artifacts:** Fixed-size point primitives cause artifacts because their size does not vary with distance [S1, unofficial]. This is avoided by using solid spheres where the footprint decreases with increasing distance to the viewer [S1, unofficial].
*   **Multi-hit Ray Tracing:** Multi-hit ray tracing was experimented with but yielded mixed image quality results, so first-hit was chosen for simplicity and consistency [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Server-side complexity is dominated by volume ray marching [S1, unofficial]. Client-side complexity includes LBVH construction and ray tracing [S1, unofficial]. The sources report LBVH construction taking 11 ms and compaction taking 23 ms for a point cloud of 623,584 samples [S1, unofficial].
*   **Memory Complexity:** The server sends only non-zero radius samples, reducing bandwidth [S1, unofficial]. The client stores the BVH and sphere data [S1, unofficial].
*   **Measured Performance:** On an NVIDIA RTX 2080 GPU, rendering five views with a viewport of $1024 \times 1024$ pixels resulted in frame rates ranging from 229 FPS to 408 FPS, depending on the view [S1, unofficial].
*   **Strengths:** Eliminates reprojection artifacts associated with rasterized point primitives; reduces bandwidth by compacting zero-radius samples [S1, unofficial].
*   **Limitations:** Requires real-time ray tracing hardware (e.g., NVIDIA RTX); LBVH construction adds latency on the client side [S1, unofficial].
*   **When to use:** Remote volume rendering where visual quality (artifact-free reprojection) is prioritized over minimal client-side latency, and hardware supports real-time ray tracing [S1, unofficial].
*   **When not to use:** Hardware without ray tracing support; scenarios where network bandwidth is extremely constrained and compaction is insufficient [S1, unofficial].

## Background
The method was proposed by Zellmann as an augmentation to the image-warping-based remote volume rendering algorithm by Zellmann et al. [ZAL12] [S1, unofficial]. It shifts from an image-based approach to an object-order approach based on real-time ray tracing [S1, unofficial]. The implementation uses NVIDIA CUDA for rendering and Boost Asio for network communication [S1, unofficial].

## Sources
[S1] Augmenting Image Warping-Based Remote Volume Rendering with Ray Tracing, https://doi.org/10.48550/arxiv.2006.14726, fetched 2026-10-26 (unofficial)
