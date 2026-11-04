# Frequency Domain Directional Shading

Also called: Frequency domain shading, Gradient moment shading, Gradient-based frequency shading, Hemispherical light source shading, Nusselt's analog shading

## Overview
Frequency Domain Directional Shading is a local illumination technique for volume rendering that computes diffuse shading entirely in the frequency domain, avoiding spatial domain differentiation [S1, S2, unofficial]. It solves the problem of applying directional lighting cues to volume data without the computational cost or nonlinearity issues of spatial domain methods [S1, S2, unofficial]. Unlike [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md) and [Lambert Shading with Shadow Rays](1133-lambert-shading-with-shadow-rays.md), which operate in the spatial domain or require shadow rays, this method leverages the property that spatial differentiation corresponds to multiplication by a linear ramp in the frequency domain [S2, unofficial]. It differs from [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md) by using a hemispherical light source model to maintain linearity, enabling efficient frequency domain computation [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Premultiplied volume data $f(x) m(x)$, where $f(x)$ is the density and $m(x)$ is the opacity mask [S1, S2, unofficial].
- Input: Light direction vector $L$ (unit vector) [S2, unofficial].
- Input: Interpolation filter $H(s)$ [S1, unofficial].
- Output: Shaded projection slice $I(x)$ [S1, unofficial].

**Data Structures:**
- Volume data: 3D array of complex numbers representing the Fourier transform of the premultiplied volume [S2, unofficial].
- Gradient magnitude spectrum: Fourier transform of $|\nabla f(x)| m(x)$ [S1, S2, unofficial].
- Frequency coordinates: 3D grid of frequency vectors $s = [s_x, s_y, s_z]$ [S1, unofficial].

**Steps:**
1. **Compute Gradient Magnitude Spectrum:**
   - Compute the gradient magnitude $|\nabla f(x)|$ in the spatial domain from the volume data $f(x)$ [S1, S2, unofficial].
   - Multiply by the opacity mask $m(x)$ to get $|\nabla f(x)| m(x)$ [S1, S2, unofficial].
   - Compute the 3D Fourier Transform to obtain $F\{|\nabla f(x)| m(x)\}$ [S1, S2, unofficial].
   - Note: There is no simple frequency domain representation for gradient magnitude, so this requires a separate volume extraction [S1, S2, S3, unofficial].

2. **Reconstruct Pure Spectrum:**
   - Compute the 3D Fourier Transform of the premultiplied volume $f(x) m(x)$ to get $F\{f(x) m(x)\}$ [S1, S2, unofficial].
   - Apply the interpolation filter $H(s)$ to reconstruct the pure spectrum of $f(x)$ [S1, S2, unofficial]. This step compensates for the filter response during slice extraction [S1, S3, unofficial].

3. **Compute First Moment for Directional Shading:**
   - The directional shading term involves the gradient $\nabla f(x)$ [S3, unofficial].
   - By the derivative theorem, the gradient in the spatial domain corresponds to the first moment in the frequency domain [S3, unofficial].
   - Multiply the pure spectrum $F\{f(x)\}$ by $i \pi s$ (where $s$ is the frequency vector) to obtain the spectrum of the gradient $\nabla f(x)$ [S1, S2, S3, unofficial].
   - Specifically, the term $\nabla f(x) \cdot L$ is computed by multiplying the spectrum by $i \pi (s \cdot L)$ [S1, S2, unofficial].

4. **Combine Shading Terms:**
   - The hemispherical light source model provides a linear diffuse term: $E_i = L_{dif} \frac{1}{2} (1 + N \cdot L)$ [S2, S3, unofficial].
   - For volumes, the normal $N$ is approximated by the normalized gradient direction, but the model uses the gradient magnitude $|\nabla f(x)|$ and the gradient vector $\nabla f(x)$ directly [S1, S2, S3, unofficial].
   - The shaded contribution is computed as:
     $$ C_{dif} L_{dif} \frac{1}{2} \left( |\nabla f(x)| + (\nabla f(x) \cdot L) \right) $$
   - In the frequency domain, this corresponds to combining the spectrum of $|\nabla f(x)| m(x)$ and the moment term of $F\{f(x)\}$ [S1, S2, S3, unofficial].
   - The ambient term $C_{amb} L_{amb} f(x)$ is added separately [S2, S3, unofficial].

5. **Slice Extraction:**
   - Perform slice extraction (inverse Fourier transform) on the combined frequency domain shading terms to obtain the spatial slice [S1, S2, unofficial].
   - The method operates entirely in the frequency domain and requires computations only on the plane of the slice being extracted [S1, S2, unofficial].

**Formulas:**
- Hemispherical irradiance: $E_i = L_{dif} \frac{1}{2} (1 + N \cdot L)$ [S2, S3, unofficial].
- Volume diffuse term: $C_{dif} L_{dif} \frac{1}{2} (|\nabla f(x)| + (\nabla f(x) \cdot L))$ [S1, S2, S3, unofficial].
- Frequency domain differentiation: Multiplication by $i \pi s$ in frequency domain equals differentiation in spatial domain [S2, unofficial].

**Parameters:**
- Light direction $L$: Usually set perpendicular to the view vector $V$ to maximize shading effect and eliminate the adjustment term in combined depth cueing/shading [S2, unofficial].
- $C_{dif}$: Diffuse color constant [S2, unofficial].
- $L_{dif}$: Directional light intensity constant [S2, unofficial].
- $C_{amb}$: Ambient color constant [S2, unofficial].
- $L_{amb}$: Ambient light intensity constant [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Gradient Magnitude Computation:** The gradient magnitude $|\nabla f(x)|$ cannot be computed simply in the frequency domain, requiring a second spectrum $F\{|\nabla f(x)| m(x)\}$ and thus two slice extractions from two volumes [S1, S2, S3, unofficial].
- **Negative Values:** A linear shading equation such as $C_{amb} L_{amb} f(x) + C_{dif} L_{dif} \nabla f(x)$ can be derived but requires restricting the upper bound of $C_{dif}$ to avoid negative values, which restricts the shading effect [S1, S2, S3, unofficial].
- **Sine Wave Depth Cueing:** If depth cueing is implemented as a sine wave, the filter $H'$ requires shifting $H$ and adding three copies with complex weights, increasing filter kernel size [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frequency_domain_directional_shading(volume_data, light_direction, view_direction, interpolation_filter):
    """
    Compute directional shading in the frequency domain.
    
    Args:
        volume_data: 3D array of complex numbers representing F{f(x) * m(x)} [S1, S2, unofficial]
        light_direction: Unit vector L [S2, unofficial]
        view_direction: Unit vector V [S2, unofficial]
        interpolation_filter: Function H(s) returning filter value at frequency s [S1, unofficial]
        
    Returns:
        shaded_slice: 2D array of real numbers for the extracted slice
    """
    # Step 1: Compute gradient magnitude spectrum
    # Gradient magnitude cannot be computed simply in frequency domain, requiring a second spectrum [S1, S2, S3, unofficial]
    spatial_volume = inverse_fourier_transform(volume_data)
    gradient_magnitude_spatial = compute_gradient_magnitude(spatial_volume)
    gradient_magnitude_spectrum = fourier_transform(gradient_magnitude_spatial)

    # Step 2: Reconstruct pure spectrum
    # Apply interpolation filter H(s) to reconstruct pure spectrum of f(x) [S1, S2, unofficial]
    freq_grid = generate_frequency_grid(volume_data.shape)
    pure_spectrum = volume_data * interpolation_filter(freq_grid)

    # Step 3: Compute first moment for directional shading
    # Multiply spectrum by i * pi * s to apply directional shading [S1, S2, S3, unofficial]
    # The term (grad f . L) corresponds to multiplying the spectrum by i * pi * (s . L)
    directional_component_spectrum = pure_spectrum * (1j * np.pi * np.dot(freq_grid, light_direction))

    # Step 4: Combine shading terms
    # Combine gradient magnitude spectrum and directional component spectrum [S1, S2, S3, unofficial]
    shaded_spectrum = gradient_magnitude_spectrum + directional_component_spectrum

    # Step 5: Slice extraction
    # Perform slice extraction (inverse Fourier transform) on the combined frequency domain terms [S1, S2, unofficial]
    shaded_spatial = inverse_fourier_transform(shaded_spectrum)
    
    # Add ambient term (simplified, assuming ambient is just f(x))
    ambient_spatial = inverse_fourier_transform(volume_data)
    final_slice = ambient_spatial + shaded_spatial
    
    return final_slice

# Helper definitions (standard operations)
def fourier_transform(data):
    """Compute 3D discrete Fourier transform."""
    return np.fft.fftn(data)

def inverse_fourier_transform(data):
    """Compute 3D inverse discrete Fourier transform."""
    return np.fft.ifftn(data).real

def compute_gradient_magnitude(spatial_data):
    """Compute gradient magnitude |∇f(x)| in spatial domain."""
    # Compute partial derivatives along each axis
    dx = np.gradient(spatial_data, axis=0)
    dy = np.gradient(spatial_data, axis=1)
    dz = np.gradient(spatial_data, axis=2)
    # Return magnitude
    return np.sqrt(dx**2 + dy**2 + dz**2)

def generate_frequency_grid(shape):
    """Generate 3D frequency coordinate grid s = [sx, sy, sz]."""
    sx = np.fft.fftfreq(shape[0])
    sy = np.fft.fftfreq(shape[1])
    sz = np.fft.fftfreq(shape[2])
    return np.meshgrid(sx, sy, sz, indexing='ij')
```

## Subtopics

### [Hemisphere Illumination](1384-hemisphere-illumination.md)
A local shading model that simulates diffuse reflection from a virtual hemisphere light source to provide directional illumination cues. It enhances shape perception by weighting volume points based on orientation relative to a light direction. Unlike standard volume rendering, it uses a hemisphere light source for directional illumination, though it remains a local approximation. It builds on Frequency Domain Directional Shading by computing directional shading in the frequency domain using an orientation field.

## Operating characteristics
- **Time Complexity:** The method operates entirely in the frequency domain and requires computations only on the plane of the slice being extracted [S1, S2, unofficial]. Computing a moment incurs a few additional floating-point operations per sample on the slice, which is small relative to the convolution operations [S1, S2, unofficial].
- **Memory Complexity:** Requires a pair of floating-point numbers for each voxel due to the wide dynamic range and complex arithmetic of Fourier transforms [S2, unofficial]. Additionally, it requires a second spectrum for the gradient magnitude, effectively doubling the memory requirement for the volume data during processing [S1, S2, S3, unofficial].
- **Measured Performance:** The additional expense of computing the moment is small relative to the convolution operations [S1, S2, unofficial]. However, the need for two slice extractions (one for the volume and one for the gradient magnitude) increases the overall computational cost [S1, S2, S3, unofficial].
- **Strengths:** Avoids the nonlinearity of the MAX function used in Lambertian reflection, allowing computation entirely in the frequency domain [S1, S2, unofficial]. Efficient for large datasets as it leverages FFTs [S2, unofficial].
- **Limitations:** Requires two slice extractions due to the lack of a simple frequency domain representation for gradient magnitude [S1, S2, S3, unofficial]. The shading effect is restricted if a linear shading equation is used to avoid negative values [S1, S2, S3, unofficial].
- **When to Use:** Suitable for volume rendering where directional shading cues are important and computational efficiency in the frequency domain is desired.
- **When Not to Use:** When memory is constrained or when the overhead of two slice extractions is prohibitive.

## Background
- **Origin:** Developed by Takashi Totsuka and Marc Levoy [S1, S2, S3, unofficial].
- **History:** Presented in the paper "Frequency Domain Volume Rendering" at SIGGRAPH 1993 [S1, S2, S3, unofficial].
- **Development:** The method builds on previous work in frequency domain volume rendering, extending it to include directional shading using a hemispherical light source model [S1, S2, S3, unofficial].
- **Usage:** Used in volume rendering applications where efficient directional shading is required, particularly in medical imaging and scientific visualization [S1, S2, S3, unofficial].

## Sources
[S1] PDF Frequency Domain Volume Rendering - Stanford University, https://graphics.stanford.edu/papers/fvr/totsuka-fvr-sig93.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF FREQUENCY DOMAIN VOLUME RENDERING Takashi Totsuka Marc Levoy ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/93/570/CSL-TR-93-570.pdf, fetched 2026-10-27 (unofficial)
[S3] Frequency Domain Volume Rendering, https://graphics.stanford.edu/papers/fvr/fvr.pdf, fetched 2026-10-27 (unofficial)
