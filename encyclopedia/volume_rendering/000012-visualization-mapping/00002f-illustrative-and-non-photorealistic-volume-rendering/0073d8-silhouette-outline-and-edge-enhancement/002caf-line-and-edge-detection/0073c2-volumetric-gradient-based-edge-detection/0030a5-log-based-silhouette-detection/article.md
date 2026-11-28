# LOG-based Silhouette Detection

Also called: Laplacian of Gaussian edge detection

## Overview
LOG-based Silhouette Detection is a volumetric edge detection technique used to identify silhouette voxels for rendering sketch-like curves in non-photorealistic volume rendering [S1]. It solves the performance bottleneck of exhaustive spatial searches by replacing them with a precomputed Laplacian of Gaussian (LOG) filter and simple thresholding [S1].

## How it works
**Inputs:**
- A scalar volume dataset $V$ [S1].
- Camera view vector $\vec{E}$ (eye vector) [S1].
- Three user-controllable thresholds: $T_{hLOG}$, $T_{heye}$, and $T_{hgrad}$ [S1].

**Outputs:**
- A set of line segments representing silhouette curves, drawn through the centers of detected silhouette voxels [S1].

**Data Structures:**
- Precomputed LOG volume: A 3D array of the same dimensions as $V$, storing the LOG value for each voxel [S1].

**Preprocessing:**
1. Compute the Laplacian of Gaussian (LOG) value for each voxel in the volume dataset [S1].

**Rendering Steps:**
For each voxel $i$ in the volume:
1. Calculate the LOG value product: $P_{LOG} = V_i \times \text{LOG}(V_i)$ [S1].
2. Calculate the gradient $\nabla \vec{V_i}$ and its magnitude: $M_{grad} = \|\nabla \vec{V_i}\|$ [S1].
3. Calculate the dot product with the eye vector: $D_{eye} = \vec{E} \cdot \nabla \vec{V_i}$ [S1].
4. Check if the voxel satisfies all three criteria simultaneously [S1]:
   - $V_i \times \text{LOG}(V_i) < T_{hLOG}$ [S1]
   - $\vec{E} \cdot \nabla \vec{V_i} < T_{heye}$ [S1]
   - $\|\nabla \vec{V_i}\| > T_{hgrad}$ [S1]
5. If all criteria are met, draw a line segment through the center of the voxel in the direction of $\nabla \vec{V_i} \times \vec{E}$ [S1].

**Parameters:**
- $T_{hLOG}$: Threshold for the LOG value product [S1].
- $T_{heye}$: Threshold for the dot product of the eye vector and gradient [S1].
- $T_{hgrad}$: Threshold for the gradient magnitude [S1].

**Edge Cases and Pitfalls:**
- The method relies on precomputed LOG values; if the volume changes, preprocessing must be repeated [S1].
- The cross product direction becomes zero if the gradient is parallel to the eye vector, requiring a fallback direction or segment omission [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volume(volume):
    # [G] Standard 3x3x3 Laplacian of Gaussian kernel convolution.
    # Not given in the sources.
    log_volume = apply_log_filter(volume)
    return log_volume

def detect_and_render_silhouettes(volume, log_volume, eye_vector, 
                                  t_hlog, t_hey, t_hgrad):
    segments = []
    dimensions = volume.shape
    
    for i in range(dimensions[0]):
        for j in range(dimensions[1]):
            for k in range(dimensions[2]):
                v_i = volume[i, j, k]
                log_v_i = log_volume[i, j, k]
                
                # Compute gradient using central differences [G]
                grad_v_i = compute_gradients(volume, i, j, k)
                
                # Criterion 1: LOG value product
                criterion_1 = (v_i * log_v_i) < t_hlog
                
                # Criterion 2: Dot product with eye vector
                dot_product = dot(eye_vector, grad_v_i)
                criterion_2 = dot_product < t_hey
                
                # Criterion 3: Gradient magnitude
                grad_magnitude = magnitude(grad_v_i)
                criterion_3 = grad_magnitude > t_hgrad
                
                # Check all criteria
                if criterion_1 and criterion_2 and criterion_3:
                    # Direction of the line segment
                    direction = cross(grad_v_i, eye_vector)
                    
                    # Fallback for degenerate cross product [G]
                    if magnitude(direction) < 1e-6:
                        direction = (0.0, 0.0, 1.0)
                        
                    center = (i, j, k)
                    segments.append((center, direction))
                    
    return segments

def apply_log_filter(volume):
    # [G] Standard 3x3x3 Laplacian of Gaussian kernel convolution.
    # Not given in the sources.
    pass

def compute_gradients(volume, i, j, k):
    # [G] Central differences for gradient computation.
    # Not given in the sources.
    pass

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def magnitude(v):
    return (v[0]**2 + v[1]**2 + v[2]**2)**0.5

def cross(a, b):
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])
```

## Subtopics

### [Silhouette Detection via Laplacian Filter](10558-silhouette-detection-via-laplacian-filter.md)
This method detects object silhouettes in image space to prevent advection particles from flowing across geometric boundaries, which would otherwise render the surface's geometry and topology unrecognizable. It replaces exhaustive 3x3x3 subspace searches used in parent methods by applying a discrete Laplacian filter to depth values followed by thresholding.

## Operating characteristics
- **Performance:** The LOG technique provides virtually identical results to the exhaustive 3x3x3 subspace search but is much faster to calculate per frame because it simplifies boundary computation [S1].
- **Frame Rates:** Interactive rendering is possible, with frame rates of 20 to 30 frames per second on modern PCs (Intel Xeon 2.00 GHz, Geforce 3 Ti 500) [S1].
- **Data Volume:** The total number of point primitives in a typical dataset ranges from 5,000 to 2,000,000, and silhouette curves range from 1,000 to 300,000 [S1].
- **Preprocessing Time:** Varies from seconds to a minute depending on dataset size [S1].
- **Strengths:** Significantly improves image quality by highlighting structures, especially bones in medical datasets [S1].
- **Limitations:** Performance can be improved by further reducing cache exchange and floating-point operations [S1].

## Background
- **Origin:** The method was introduced in the paper "Non-Photorealistic Volume Rendering Using Stippling Techniques" [S1].
- **Purpose:** It was developed to address the performance bottleneck of exhaustive spatial searches in non-photorealistic volume rendering, specifically for sketch-like silhouette curves [S1].
- **Application:** It is used in stipple rendering systems to enhance the visual clarity of volumetric data, such as bonsai trees and aneurysm volumes [S1].

## Sources
[S1] Non-Photorealistic Volume Rendering Using Stippling Techniques, https://www.csee.umbc.edu/~rheingan/pubs/vis_stipple.pdf, fetched 2026-10-27
