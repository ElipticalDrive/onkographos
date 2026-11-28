# MLT Path Radiance Computation

Also called: Lbidir, Lpath, MLT Bidirectional Path Radiance Computation, PathL

## Overview
Computes the radiance contribution of a generated camera path and, optionally, a light path in a participating medium by estimating direct lighting at each vertex and connecting vertices between the two paths. It differs from its sibling [Direct Light Sampling](3337-direct-light-sampling.md) by operating on entire path segments (camera and light) rather than single light rays, and from [Light Sampling for NEE](2616-light-sampling-for-nee.md) by supporting bidirectional connections between camera and light path vertices to handle complex light transport in media.

## How it works
The method generates a camera path and, if bidirectional mode is enabled, a light path. It then computes the radiance by summing emitted light, direct lighting estimates at each vertex, and direct connections between camera and light path vertices. The implementation distinguishes between path tracing (camera path only) and bidirectional path tracing (camera and light paths).

### Inputs and Outputs
**Inputs:**
- `scene`: The scene geometry and medium properties.
- `camera`: Camera object providing ray generation.
- `cameraPath`: Array of camera path vertices.
- `cameraLength`: Number of vertices in the camera path.
- `lightPath` (optional): Array of light path vertices.
- `lightLength` (optional): Number of vertices in the light path.
- `arena`: Memory arena for temporary allocations.
- `rng`: Random number generator.
- `time`: Time of the frame.
- `lightDistribution`: Distribution for sampling lights.
- `escapedRay`: Ray that escaped the scene without hitting geometry.
- `escapedAlpha`: Alpha of the escaped ray.
- `bidirectional`: Boolean flag to enable bidirectional path tracing.

**Outputs:**
- `L`: Computed radiance value.

### Data Structures
- **Vertex**: Represents a point on a path. Fields include:
  - `alpha`: Transmittance/weight up to this vertex.
  - `isect`: Intersection data.
  - `wPrev`: Incoming direction.
  - `specularBounce`: Boolean indicating if the previous bounce was specular.
  - `nSpecularComponents`: Number of specular components in the BSDF.
  - `bsdf`: Bidirectional Scattering Distribution Function.
  - `p`: Position.
  - `n`: Normal.
- **nSpecularVertices**: Array of integers, size `cameraPathLength + lightPathLength + 2`. Stores the cumulative count of specular vertices up to each index.

### Steps
1. **Initialization**:
   - Initialize `L = 0`.
   - Initialize `previousSpecular = true`, `allSpecular = true`.
   - Calculate `nVerts = cameraPathLength + lightLength + 2`.
   - Allocate `nSpecularVertices` array of size `nVerts`, initialized to 0.
   - Loop `i` from 0 to `cameraPathLength - 1` and `j` from 0 to `lightLength - 1`:
     - If `cameraPath[i].specularBounce` or `lightPath[j].specularBounce`, increment `nSpecularVertices[i + j + 2]`.

2. **Camera Path Processing**:
   - Loop `i` from 0 to `cameraPathLength - 1`:
     - Get vertex `vc = cameraPath[i]`, position `pc = vc.p`, normal `nc = vc.n`.
     - **Emitted Light**: If `previousSpecular` and (`directLighting == NULL` or `!allSpecular`), add emitted light:
       $$ L += vc.alpha \cdot L_e(vc.w_{prev}) $$
       where $L_e$ is the emitted radiance from the intersection.
     - **Direct Lighting**: If `directLighting == NULL` or `!allSpecular`:
       - Sample light: `lightNum = lightDistribution->SampleDiscrete(ls.lightNum, &lightPdf)`.
       - Call `EstimateDirect` with `BxDFType(BSDF_ALL & ~BSDF_SPECULAR)`.
       - Compute direct lighting:
         $$ L_d = vc.alpha \cdot \frac{\text{EstimateDirect}(\dots)}{\text{lightPdf}} $$
       - Add to `L`:
         $$ L += \frac{L_d}{i + 1 - nSpecularVertices[i+1]} $$
     - **Update Specular Flags**:
       - `previousSpecular = vc.specularBounce`
       - `allSpecular &= previousSpecular`

3. **Bidirectional Connections (if bidirectional)**:
   - Loop `i` from 0 to `cameraPathLength - 1`:
     - Get `vc, pc, nc`.
     - If `!vc.specularBounce`:
       - Loop `j` from 0 to `lightLength - 1`:
         - Get `vl = lightPath[j]`, `pl = vl.p`, `nl = vl.n`.
         - If `!vl.specularBounce`:
           - Compute connection direction: $w = \text{Normalize}(pl - pc)$.
           - Compute BSDF values:
             $$ f_c = v_c.bsdf.f(v_c.w_{prev}, w) \cdot (1 + v_c.nSpecularComponents) $$
             $$ f_l = v_l.bsdf.f(-w, v_l.w_{prev}) \cdot (1 + v_l.nSpecularComponents) $$
           - If $f_c$ or $f_l$ is black, continue.
           - Create test ray $r(pc, pl - pc, 10^{-3}, 0.999, time)$.
           - If `!scene->IntersectP(r)`:
             - Compute path weight:
               $$ pathWt = \frac{1}{i + j + 2 - nSpecularVertices[i+j+2]} $$
             - Compute geometry term:
               $$ G = \frac{|\cos \theta_c| \cdot |\cos \theta_l|}{\|pl - pc\|^2} $$
               where $\cos \theta_c = \text{AbsDot}(nc, w)$ and $\cos \theta_l = \text{AbsDot}(nl, w)$.
             - Add to `L`:
               $$ L += (vc.alpha \cdot f_c \cdot G \cdot f_l \cdot vl.alpha) \cdot pathWt $$

4. **Escaped Ray Contribution**:
   - If `escapedRay` exists (`!eAlpha.IsBlack()`) and `previousSpecular` and (`directLighting == NULL` or `!allSpecular`):
     - Loop over lights:
       $$ L += eAlpha \cdot scene->lights[i]->L_e(eRay) $$

5. **Path Generation (if bidirectional)**:
   - Generate camera ray: `camera->GenerateRayDifferential(sample.cameraSample, &cameraRay)`.
   - Scale differentials: `cameraRay.ScaleDifferentials(1.0 / sqrt(nPixelSamples))`.
   - Generate camera path: `GeneratePath(cameraRay, cameraWt, scene, arena, sample.cameraPathSamples, cameraPath, &escapedRay, &escapedAlpha)`.
   - If `!bidirectional`, return `Lpath(scene, cameraPath, cameraLength, arena, sample.lightingSamples, rng, sample.cameraSample.time, lightDistribution, escapedRay, escapedAlpha)`.
   - If `bidirectional`:
     - Sample light: `lightNum = lightDistribution->SampleDiscrete(sample.lightNumSample, &lightPdf)`.
     - Get light.
     - Sample light ray: `light->Sample_L(..., &lightRay, &Nl, &lightRayPdf)`.
     - Compute light weight: `lightWt *= AbsDot(Normalize(Nl), lightRay.d) / (lightPdf * lightRayPdf)`.
     - Generate light path: `lightLength = GeneratePath(RayDifferential(lightRay), lightWt, scene, arena, sample.lightPathSamples, lightPath, NULL, NULL)`.
     - Return `Lbidir(scene, cameraPath, cameraLength, lightPath, lightLength, arena, sample.lightingSamples, rng, sample.cameraSample.time, lightDistribution, escapedRay, escapedAlpha)`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def MLT_Path_Radiance_Computation(scene, camera, cameraPath, cameraLength, lightPath, lightLength, arena, rng, time, lightDistribution, escapedRay, escapedAlpha, bidirectional):
    L = 0.0
    previousSpecular = True
    allSpecular = True
    
    nVerts = cameraLength + lightLength + 2
    nSpecularVertices = [0] * nVerts
    
    # Count specular vertices
    for i in range(cameraLength):
        for j in range(lightLength):
            if cameraPath[i].specularBounce or lightPath[j].specularBounce:
                nSpecularVertices[i + j + 2] += 1
                
    # Process camera path
    for i in range(cameraLength):
        vc = cameraPath[i]
        pc = vc.p
        nc = vc.n
        
        # Emitted light
        if previousSpecular and (directLighting is None or not allSpecular):
            L += vc.alpha * vc.isect.Le(vc.wPrev)
            
        # Direct lighting
        if directLighting is None or not allSpecular:
            lightNum, lightPdf = lightDistribution.SampleDiscrete(ls.lightNum)
            light = scene.lights[lightNum]
            # EstimateDirect is a helper function that samples the light and computes direct illumination
            # It returns the direct lighting contribution
            Ld = vc.alpha * EstimateDirect(vc, light, lightPdf) / lightPdf
            # Specular correction
            correction = i + 1 - nSpecularVertices[i+1]
            if correction > 0:
                L += Ld / correction
                
        previousSpecular = vc.specularBounce
        allSpecular = allSpecular and previousSpecular
        
    # Bidirectional connections
    if bidirectional:
        for i in range(cameraLength):
            vc = cameraPath[i]
            pc = vc.p
            nc = vc.n
            if not vc.specularBounce:
                for j in range(lightLength):
                    vl = lightPath[j]
                    pl = vl.p
                    nl = vl.n
                    if not vl.specularBounce:
                        w = Normalize(pl - pc)
                        fc = vc.bsdf.f(vc.wPrev, w) * (1 + vc.nSpecularComponents)
                        fl = vl.bsdf.f(-w, vl.wPrev) * (1 + vl.nSpecularComponents)
                        
                        if fc.is_black() or fl.is_black():
                            continue
                            
                        # Shadow ray
                        r = Ray(pc, pl - pc, 1e-3, 0.999, time)
                        if not scene.IntersectP(r):
                            pathWt = 1.0 / (i + j + 2 - nSpecularVertices[i+j+2])
                            G = AbsDot(nc, w) * AbsDot(nl, w) / DistanceSquared(pl, pc)
                            L += (vc.alpha * fc * G * fl * vl.alpha) * pathWt
                            
    # Escaped ray contribution
    if not escapedAlpha.IsBlack() and previousSpecular and (directLighting is None or not allSpecular):
        for i in range(scene.num_lights):
            L += escapedAlpha * scene.lights[i].Le(escapedRay)
            
    return L

def EstimateDirect(vc, light, lightPdf):
    # This is a placeholder for the actual EstimateDirect implementation
    # which involves sampling the light and computing the direct illumination
    # contribution based on the BSDF and geometry.
    # It returns the direct lighting contribution.
    pass
```

### Parameters
- `directLighting`: Not given in the sources.
- `ls.lightNum`: Not given in the sources.
- `nPixelSamples`: Not given in the sources.

### Edge Cases and Pitfalls
- **Specular Vertices**: The specular correction factor `i + 1 - nSpecularVertices[i+1]` must be positive. If it is zero or negative, the division should be skipped or handled to avoid division by zero.
- **Shadow Rays**: The test ray for connections must be created with appropriate `mint` and `maxt` values to avoid self-intersections and to ensure it reaches the target vertex.
- **Black BSDF Values**: If $f_c$ or $f_l$ is black, the connection should be skipped to avoid adding zero or invalid contributions.

## Subtopics

### Lpath
Path tracing variant that only uses the camera path. Choose when bidirectional connections are not needed or when the scene is simple. Link: [Lpath](52-lpath.md).

### Lbidir
Bidirectional path tracing variant that connects camera and light paths. Choose for complex scenes with participating media where bidirectional connections improve convergence. Link: [Lbidir](52-lbidir.md).

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Not given in the sources.
- **Strengths**: Handles complex light transport in participating media by combining path tracing and bidirectional path tracing.
- **Limitations**: Computationally expensive due to the need to generate and process both camera and light paths.
- **When to Use**: Complex scenes with participating media where bidirectional connections are beneficial.
- **When Not to Use**: Simple scenes or when computational resources are limited.

## Background
The method is part of the MLT (Metropolis Light Transport) framework for rendering participating media. It was introduced in the pbrt-photon-beams repository, an unofficial extension of the pbrt renderer. The method builds on [Direct Light Sampling](3337-direct-light-sampling.md) by extending it to handle path segments and bidirectional connections.

## Sources
[S1] brianrli/pbrt-photon-beams, https://github.com/brianrli/pbrt-photon-beams, fetched 2026-10-27 (unofficial)
