# Wavefront Path Tracer with Spatial Hashing

Also called: Spatial Hash Path Tracing

## Overview
A path tracing method that uses spatial hashing to accumulate radiance estimates before final display [S1, unofficial]. It differs from standard path tracing by decoupling the accumulation buffer from the pixel grid, enabling spatial averaging of radiance estimates across hash buckets [S1, unofficial].

## How it works
The method consists of four distinct phases: Generate, Extend, Accumulate, and Display [S1, unofficial]. It relies on Multiple Importance Sampling (MIS) to combine light source and BRDF samples [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:** G-buffer raster (containing pixel position, normal, and albedo), voxel grid (sparse representation of participating media or surfaces), sun disk direction, and camera parameters [S1, unofficial][G].
*   **Outputs:** Final image with surface details retained [S1, unofficial].

**Data Structures:**
*   **Ray Queue:** A dynamic list of rays to be processed in the Extend phase [S1, unofficial].
*   **Spatial Hash Map:** A hash table mapping 3D spatial coordinates (bucket indices) to accumulated albedo-demodulated radiance values [S1, unofficial].
*   **Hash Function:** Maps a 3D position to a bucket index by integer division of coordinates by the cell size [G].

**MIS Weight Formula:**
The importance sampling weight $F$ for a direction sample $X$ is calculated as [S1, unofficial]:
$$
F = \frac{f(X)}{(1/2)p_1(X) + (1/2)p_2(X)}
$$
Where:
*   $F$ is the MIS weight [S1, unofficial].
*   $f(X)$ is the PDF of the combined distribution [S1, unofficial].
*   $p_1(X)$ is the probability of sampling from the cosine-weighted normal distribution [S1, unofficial].
*   $p_2(X)$ is the probability of sampling from the sun disk distribution [S1, unofficial].
*   Both distributions are sampled with equal probability $1/2$ [S1, unofficial].

**Steps:**
1.  **Generate:** For each pixel in a gbuffer raster, generate a path and scatter rays using MIS. Place the rays into a ray queue [S1, unofficial].
2.  **Extend:** Intersect each ray with the voxel grid using sparse ray marching. Shade each intersection by updating the path’s throughput. If scattering occurs, place the scattered ray into the output ray queue [S1, unofficial].
3.  **Accumulate:** For each path, output the albedo-demodulated radiance into the spatial hash using the path’s radiance [S1, unofficial].
4.  **Display:** Multiply the shading from the spatial hash by the albedo again to retain surface details lost during spatial averaging in the hash buckets [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random
from collections import namedtuple

Ray = namedtuple('Ray', ['origin', 'direction', 'throughput'])

def wavefront_path_tracer(gbuffer, voxel_grid, sun_direction, cell_size, num_samples):
    # Data structures
    ray_queue = []
    spatial_hash = {}  # key: (bx, by, bz), value: [sum_demodulated_radiance, count]

    # Helper: hash bucket index [G]
    def get_bucket(pos):
        return (int(pos.x / cell_size), int(pos.y / cell_size), int(pos.z / cell_size))

    # Helper: cosine-weighted hemisphere sampling [G]
    def sample_cosine_weighted(normal):
        u1, u2 = random.random(), random.random()
        z = 1.0 - u1
        r = math.sqrt(max(0.0, 1.0 - z * z))
        phi = 2.0 * math.pi * u2
        tangent = math.cross(normal, (1.0, 0.0, 0.0))
        if math.dot(tangent, tangent) < 1e-6:
            tangent = math.cross(normal, (0.0, 1.0, 0.0))
        tangent = math.normalize(tangent)
        bitangent = math.cross(normal, tangent)
        return math.normalize(tangent * (r * math.cos(phi)) + bitangent * (r * math.sin(phi)) + normal * z)

    def pdf_cosine(direction, normal):
        return max(0.0, math.dot(direction, normal) / math.pi)

    # Helper: sun disk sampling [G]
    def sample_sun_disk(sun_dir):
        return sun_dir  # Simplified uniform direction near sun [G]

    def pdf_sun(direction, sun_dir):
        return 1.0 / (4.0 * math.pi)  # Simplified uniform sphere PDF [G]

    # Helper: sparse ray marching [G]
    def sparse_ray_march(ray, grid):
        # Standard sparse grid traversal stepping through cells until hit or exit [G]
        t = 0.0
        step = cell_size
        while t < 1000.0:  # Max march distance [G]
            pos = ray.origin + ray.direction * t
            bucket = get_bucket(pos)
            if bucket in grid:
                return grid[bucket]  # Return hit info [G]
            t += step
        return None

    # Generate Phase
    for pixel in gbuffer:
        for _ in range(num_samples):
            if random.random() < 0.5:
                ray_direction = sample_cosine_weighted(pixel.normal)
                p1 = pdf_cosine(ray_direction, pixel.normal)
                p2 = pdf_sun(ray_direction, sun_direction)
            else:
                ray_direction = sample_sun_disk(sun_direction)
                p1 = pdf_cosine(ray_direction, pixel.normal)
                p2 = pdf_sun(ray_direction, sun_direction)
            
            # MIS weight calculation matching source formula [S1, unofficial]
            mis_weight = p1 if random.random() < 0.5 else p2
            mis_weight = mis_weight / (0.5 * p1 + 0.5 * p2)
            throughput = pixel.albedo * mis_weight
            ray_queue.append(Ray(pixel.position, ray_direction, throughput))

    # Extend Phase
    while ray_queue:
        ray = ray_queue.pop(0)
        hit = sparse_ray_march(ray, voxel_grid)
        if hit:
            ray.throughput *= hit.albedo
            if random.random() < 0.5:
                new_dir = sample_cosine_weighted(hit.normal)
                p1 = pdf_cosine(new_dir, hit.normal)
                p2 = pdf_sun(new_dir, sun_direction)
            else:
                new_dir = sample_sun_disk(sun_direction)
                p1 = pdf_cosine(new_dir, hit.normal)
                p2 = pdf_sun(new_dir, sun_direction)
            
            mis_weight = p1 if random.random() < 0.5 else p2
            mis_weight = mis_weight / (0.5 * p1 + 0.5 * p2)
            ray_queue.append(Ray(hit.point, new_dir, ray.throughput * mis_weight))
            
            radiance = ray.throughput
            demodulated = radiance / hit.albedo if hit.albedo > 1e-6 else 0.0
            bucket = get_bucket(hit.point)
            if bucket not in spatial_hash:
                spatial_hash[bucket] = [0.0, 0]
            spatial_hash[bucket][0] += demodulated
            spatial_hash[bucket][1] += 1

    # Display Phase
    final_image = {}
    for pixel in gbuffer:
        bucket = get_bucket(pixel.position)
        if bucket in spatial_hash:
            avg_demod = spatial_hash[bucket][0] / spatial_hash[bucket][1]
            final_image[pixel.id] = avg_demod * pixel.albedo
        else:
            final_image[pixel.id] = 0.0
    return final_image
```

**Parameters:**
*   **Spatial Hash Cell Size:** Determines the size of the spatial buckets. Large cell sizes result in a blocky appearance [S1, unofficial].
*   **MIS Probabilities:** Both cosine-weighted normal and sun disk distributions are sampled with probability $1/2$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Large Cell Sizes:** Using large cell sizes in the spatial hash results in a blocky appearance [S1, unofficial].
*   **No Spatial Filtering:** The implementation does not perform spatial filtering on the hash map itself, which may lead to artifacts if the hash buckets are too large or too small [S1, unofficial].
*   **Division by Zero:** When demodulating radiance, ensure albedo is not zero to avoid division by zero errors [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Strengths:** Allows for spatial averaging of radiance estimates. Retains surface details by remodulating with albedo [S1, unofficial].
*   **Limitations:** Large cell sizes cause blocky artifacts. No spatial filtering is applied to the hash map [S1, unofficial].
*   **When to use:** Not given in the sources. [G]
*   **When not to use:** Not given in the sources. [G]

## Background
The method is loosely based on Jacco Bikker’s blog entry [S1, unofficial]. It was implemented as part of a voxel raytracer development log [S1, unofficial]. The use of MIS for combining light source and BRDF samples is a common technique in path tracing, but the specific application of spatial hashing for radiance accumulation in this context is notable [S1, unofficial].

## Sources
[S1] Voxel Raytracer Devlog // My thought repository, https://nelari.us/post/voxel-ray-tracing/, fetched 2026-10-24 (unofficial)
