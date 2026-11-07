# Combined Depth Cueing and Shading (Reversed Order)
Also called: Reversed order shading and depth cueing

## Overview
Combined Depth Cueing and Shading (Reversed Order) is a frequency-domain volume rendering technique that combines directional shading and depth cueing by reversing their standard physical order to enable a single composite filter application [S1, unofficial]. While the physically correct order requires shading first followed by depth cueing, this approach applies depth cueing first using a composite filter $H'$ and then applies shading, correcting for the resulting error [S2, unofficial]. This method is distinct from the standard combined approach because it avoids the two separate convolutions required by the correct order, significantly reducing computational cost, particularly when the light vector is perpendicular to the view vector [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D scalar field $f(x)$, view vector $V$, light vector $L$, depth cueing function $d(x)$, shading coefficients ($C_{cue}, C_{dif}, L_{dif}$) [S1, S2, unofficial].
*   **Output:** Rendered image with combined depth cueing and directional shading [S1, S2, unofficial].

**Data Structures:**
*   **Composite Filter $H'$:** A frequency-domain filter that combines the interpolation filter $H$ and the depth cueing function $d(x)$ into a single operation [S1, unofficial].
*   **1D Depth Cueing Function $d_1(t)$:** The depth cueing function applied along the view direction, where $t = V \cdot x$ [S2, unofficial].

**Steps:**
1.  **Apply Composite Filter:** Apply the composite filter $H'$ to the volume spectrum to perform interpolation and depth cueing simultaneously [S1, unofficial]. This replaces the separate interpolation and depth cueing steps [S2, unofficial].
2.  **Apply Shading:** Compute the directional shading term on the result from Step 1 [S1, unofficial].
3.  **Adjustment (if necessary):** If the light vector $L$ is not perpendicular to the view vector $V$, compute and add an adjustment term to correct for the error introduced by the reversed order [S1, S2, unofficial].

**Formulas:**
The shading equation includes an ambient term, a shading term, and a constant term [S1, unofficial]. The gradient term of the shading equation is the primary component affected by the order of operations [S1, S2, unofficial].

The error term resulting from reversing the order is given by [S2, unofficial]:
$$ f(x) (\nabla [d_1(V \cdot x)] \cdot L) = f(x) d_1'(V \cdot x) (V \cdot L) $$
where $d_1'(t)$ is the derivative of the 1D depth cueing function [S2, unofficial].

If $V \cdot L$ is non-zero, the adjustment term including all coefficients is [S1, S2, unofficial]:
$$ \frac{1}{2} C_{cue} C_{dif} L_{dif} f(x) (V \cdot L) $$
This adjustment term can be computed during slice extraction without additional convolution [S1, unofficial].

For linear depth cueing, if $L$ is perpendicular to $V$ (side illumination), the difference term becomes zero, and no adjustment is necessary [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_reversed_order(volume_spectrum, V, L, d_1D_func, C_cue, C_dif, L_dif):
    # Step 1: Apply composite filter H' for interpolation and depth cueing
    # H' combines interpolation filter H and depth cueing d(x) in frequency domain
    cued_spectrum = volume_spectrum * composite_filter(V, d_1D_func)
    
    # Step 2: Transform to spatial domain and apply shading
    cued_volume = inverse_fft(cued_spectrum)
    
    # Compute gradient of the spatial volume using central differences
    grad_f = compute_gradient(cued_volume)
    
    # Compute depth cueing value at each voxel: d_1(V . x)
    depth_values = d_1D_func(dot(V, coordinates(cued_volume)))
    
    # Shading term: (gradient of f . L) * depth_cueing
    shaded_volume = (grad_f * L) * depth_values
    
    # Step 3: Adjustment if V . L != 0
    if dot(V, L) != 0:
        # Adjustment term: 0.5 * C_cue * C_dif * L_dif * f(x) * (V . L)
        adjustment = 0.5 * C_cue * C_dif * L_dif * cued_volume * dot(V, L)
        shaded_volume += adjustment
    return shaded_volume

def composite_filter(V, d_1D_func):
    # Constructs the composite frequency-domain filter H'
    # Combines interpolation filter H and depth cueing d(x)
    return interpolation_filter(V) * depth_cueing_filter(V, d_1D_func)

def inverse_fft(spectrum):
    # Standard inverse discrete Fourier transform
    return numpy.fft.ifftn(spectrum).real

def compute_gradient(volume):
    # Computes spatial gradient using central differences
    return numpy.gradient(volume)

def dot(a, b):
    # Standard dot product
    return sum(x * y for x, y in zip(a, b))

def coordinates(volume):
    # Returns meshgrid of voxel coordinates
    return numpy.meshgrid(*[range(d) for d in volume.shape], indexing='ij')
```

**Parameters:**
*   $C_{cue}$: Depth cueing coefficient [S1, unofficial].
*   $C_{dif}$: Diffuse shading coefficient [S1, unofficial].
*   $L_{dif}$: Diffuse light intensity [S1, unofficial].
*   Typical values are not specified in the sources [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   If $L$ is perpendicular to $V$, the adjustment term is zero, and no correction is needed [S1, S2, unofficial].
*   For complex depth cueing functions, a convolution is necessary for the adjustment term [S2, unofficial].
*   The correct physical order (shading first, then depth cueing) requires two convolutions, making depth cueing expensive because the composite filter $H'$ cannot be used [S2, unofficial].

## Subtopics

### Combined Depth Cueing and Shading
The parent method that applies shading first, then depth cueing, requiring two convolutions. [Combined Depth Cueing and Shading](1170-combined-depth-cueing-and-shading.md)

### Composite Depth Cueing and Directional Shading
A related method that combines depth cueing and shading using a composite filter. [Composite Depth Cueing and Directional Shading](1169-composite-depth-cueing-and-shading.md)

### Linear Depth Cueing via Fourier Projection-Slice Theorem
The foundational method for applying linear depth cueing in the frequency domain. [Linear Depth Cueing via Fourier Projection-Slice Theorem](1168-linear-depth-cueing-via-fourier-projection-slice-theorem.md)

## Operating characteristics
*   **Time Complexity:** The interpolation step in the frequency domain is $O(n^2)$, which consumes most of the running time [S2, unofficial]. Using a smaller filter (e.g., $1 \times 1 \times 1$) reduces the number of input samples contributing to each output sample, saving time, but may introduce strong ghosts in the spatial domain [S2, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, S2, unofficial].
*   **Measured Performance:** Not given in the sources [S1, S2, unofficial].
*   **Scaling:** The cost of interpolation is determined by the size of the filter [S2, unofficial].
*   **Strengths:** Avoids two convolutions by using a single composite filter $H'$, reducing computational cost [S1, S2, unofficial]. For side illumination ($L \perp V$), no adjustment is needed, further simplifying the process [S1, S2, unofficial].
*   **Limitations:** Requires an adjustment term if $L$ is not perpendicular to $V$, which may involve convolution for complex depth cueing functions [S1, S2, unofficial]. The correct physical order is more expensive [S2, unofficial].
*   **When to use:** When computational efficiency is critical and side illumination is used, or when the adjustment term can be computed efficiently [S1, S2, unofficial].
*   **When not to use:** When the highest physical accuracy is required and computational cost is not a concern, or when complex depth cueing functions require expensive convolution for the adjustment term [S2, unofficial].

## Background
This technique was developed by Takashi Totsuka and Marc Levoy as part of their work on frequency domain volume rendering [S1, unofficial]. It addresses the computational cost of combining depth cueing and directional shading in the frequency domain [S1, S2, unofficial]. The method is described in the technical report "Frequency Domain Volume Rendering" [S1, unofficial].

## Sources
[S1] PDF FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27 (unofficial)
[S2] Frequency Domain Volume Rendering - studylib.net, https://studylib.net/doc/18162906/frequency-domain-volume-rendering, fetched 2026-10-27 (unofficial)


# Combined Depth Cueing and Shading (Reversed Order)
Also called: Reversed order shading and depth cueing

## Overview
Combined Depth Cueing and Shading (Reversed Order) is a frequency-domain volume rendering technique that combines directional shading and depth cueing by reversing their standard physical order to enable a single composite filter application [S1, unofficial]. While the physically correct order requires shading first followed by depth cueing, this approach applies depth cueing first using a composite filter $H'$ and then applies shading, correcting for the resulting error [S2, unofficial]. This method is distinct from the standard combined approach because it avoids the two separate convolutions required by the correct order, significantly reducing computational cost, particularly when the light vector is perpendicular to the view vector [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D scalar field $f(x)$, view vector $V$, light vector $L$, depth cueing function $d(x)$, shading coefficients ($C_{cue}, C_{dif}, L_{dif}$) [S1, S2, unofficial].
*   **Output:** Rendered image with combined depth cueing and directional shading [S1, S2, unofficial].

**Data Structures:**
*   **Composite Filter $H'$:** A frequency-domain filter that combines the interpolation filter $H$ and the depth cueing function $d(x)$ into a single operation [S1, unofficial].
*   **1D Depth Cueing Function $d_1(t)$:** The depth cueing function applied along the view direction, where $t = V \cdot x$ [S2, unofficial].

**Steps:**
1.  **Apply Composite Filter:** Apply the composite filter $H'$ to the volume spectrum to perform interpolation and depth cueing simultaneously [S1, unofficial]. This replaces the separate interpolation and depth cueing steps [S2, unofficial].
2.  **Apply Shading:** Compute the directional shading term on the result from Step 1 [S1, unofficial].
3.  **Adjustment (if necessary):** If the light vector $L$ is not perpendicular to the view vector $V$, compute and add an adjustment term to correct for the error introduced by the reversed order [S1, S2, unofficial].

**Formulas:**
The shading equation includes an ambient term, a shading term, and a constant term [S1, unofficial]. The gradient term of the shading equation is the primary component affected by the order of operations [S1, S2, unofficial].

The error term resulting from reversing the order is given by [S2, unofficial]:
$$ f(x) (\nabla [d_1(V \cdot x)] \cdot L) = f(x) d_1'(V \cdot x) (V \cdot L) $$
where $d_1'(t)$ is the derivative of the 1D depth cueing function [S2, unofficial].

If $V \cdot L$ is non-zero, the adjustment term including all coefficients is [S1, S2, unofficial]:
$$ \frac{1}{2} C_{cue} C_{dif} L_{dif} f(x) (V \cdot L) $$
This adjustment term can be computed during slice extraction without additional convolution [S1, unofficial].

For linear depth cueing, if $L$ is perpendicular to $V$ (side illumination), the difference term becomes zero, and no adjustment is necessary [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_reversed_order(volume_spectrum, V, L, d_1D_func, C_cue, C_dif, L_dif):
    # Step 1: Apply composite filter H' for interpolation and depth cueing
    # H' combines interpolation filter H and depth cueing d(x) in frequency domain
    cued_spectrum = volume_spectrum * composite_filter(V, d_1D_func)
    
    # Step 2: Transform to spatial domain and apply shading
    cued_volume = inverse_fft(cued_spectrum)
    
    # Compute gradient of the spatial volume using central differences
    grad_f = compute_gradient(cued_volume)
    
    # Compute depth cueing value at each voxel: d_1(V . x)
    depth_values = d_1D_func(dot(V, coordinates(cued_volume)))
    
    # Shading term: (gradient of f . L) * depth_cueing
    shaded_volume = (grad_f * L) * depth_values
    
    # Step 3: Adjustment if V . L != 0
    if dot(V, L) != 0:
        # Adjustment term: 0.5 * C_cue * C_dif * L_dif * f(x) * (V . L)
        adjustment = 0.5 * C_cue * C_dif * L_dif * cued_volume * dot(V, L)
        shaded_volume += adjustment
    return shaded_volume

def composite_filter(V, d_1D_func):
    # Constructs the composite frequency-domain filter H'
    # Combines interpolation filter H and depth cueing d(x)
    return interpolation_filter(V) * depth_cueing_filter(V, d_1D_func)

def inverse_fft(spectrum):
    # Standard inverse discrete Fourier transform
    return numpy.fft.ifftn(spectrum).real

def compute_gradient(volume):
    # Computes spatial gradient using central differences
    return numpy.gradient(volume)

def dot(a, b):
    # Standard dot product
    return sum(x * y for x, y in zip(a, b))

def coordinates(volume):
    # Returns meshgrid of voxel coordinates
    return numpy.meshgrid(*[range(d) for d in volume.shape], indexing='ij')
```

**Parameters:**
*   $C_{cue}$: Depth cueing coefficient [S1, unofficial].
*   $C_{dif}$: Diffuse shading coefficient [S1, unofficial].
*   $L_{dif}$: Diffuse light intensity [S1, unofficial].
*   Typical values are not specified in the sources [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
*   If $L$ is perpendicular to $V$, the adjustment term is zero, and no correction is needed [S1, S2, unofficial].
*   For complex depth cueing functions, a convolution is necessary for the adjustment term [S2, unofficial].
*   The correct physical order (shading first, then depth cueing) requires two convolutions, making depth cueing expensive because the composite filter $H'$ cannot be used [S2, unofficial].

## Subtopics

### Combined Depth Cueing and Shading
The parent method that applies shading first, then depth cueing, requiring two convolutions. [Combined Depth Cueing and Shading](1170-combined-depth-cueing-and-shading.md)

### Composite Depth Cueing and Directional Shading
A related method that combines depth cueing and shading using a composite filter. [Composite Depth Cueing and Directional Shading](1169-composite-depth-cueing-and-shading.md)

### Linear Depth Cueing via Fourier Projection-Slice Theorem
The foundational method for applying linear depth cueing in the frequency domain. [Linear Depth Cueing via Fourier Projection-Slice Theorem](1168-linear-depth-cueing-via-fourier-projection-slice-theorem.md)

## Operating characteristics
*   **Time Complexity:** The interpolation step in the frequency domain is $O(n^2)$, which consumes most of the running time [S2, unofficial]. Using a smaller filter (e.g., $1 \times 1 \times 1$) reduces the number of input samples contributing to each output sample, saving time, but may introduce strong ghosts in the spatial domain [S2, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, S2, unofficial].
*   **Measured Performance:** Not given in the sources [S1, S2, unofficial].
*   **Scaling:** The cost of interpolation is determined by the size of the filter [S2, unofficial].
*   **Strengths:** Avoids two convolutions by using a single composite filter $H'$, reducing computational cost [S1, S2, unofficial]. For side illumination ($L \perp V$), no adjustment is needed, further simplifying the process [S1, S2, unofficial].
*   **Limitations:** Requires an adjustment term if $L$ is not perpendicular to $V$, which may involve convolution for complex depth cueing functions [S1, S2, unofficial]. The correct physical order is more expensive [S2, unofficial].
*   **When to use:** When computational efficiency is critical and side illumination is used, or when the adjustment term can be computed efficiently [S1, S2, unofficial].
*   **When not to use:** When the highest physical accuracy is required and computational cost is not a concern, or when complex depth cueing functions require expensive convolution for the adjustment term [S2, unofficial].

## Background
This technique was developed by Takashi Totsuka and Marc Levoy as part of their work on frequency domain volume rendering [S1, unofficial]. It addresses the computational cost of combining depth cueing and directional shading in the frequency domain [S1, S2, unofficial]. The method is described in the technical report "Frequency Domain Volume Rendering" [S1, unofficial].

## Sources
[S1, unofficial] PDF FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27 (unofficial)
[S2, unofficial] Frequency Domain Volume Rendering - studylib.net, https://studylib.net/doc/18162906/frequency-domain-volume-rendering, fetched 2026-10-27 (unofficial)
