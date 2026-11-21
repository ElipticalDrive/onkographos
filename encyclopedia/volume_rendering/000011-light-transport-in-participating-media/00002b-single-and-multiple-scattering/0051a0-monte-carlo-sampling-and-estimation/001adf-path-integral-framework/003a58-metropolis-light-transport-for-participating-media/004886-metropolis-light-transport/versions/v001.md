# Metropolis Light Transport

Also called: Metropolis, MetropolisRenderer

## Overview
Metropolis Light Transport (MLT) is a global illumination rendering method that uses Markov Chain Monte Carlo (MCMC) to sample light paths $\bar{x}$ according to a target distribution $\pi(\bar{x})$ proportional to the image contribution of the paths [S1]. It differs from standard path tracing by exploring the space of light transport paths via small and large mutations, allowing it to find complex light transport paths (like those involving multiple bounces) that standard path tracing might miss [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene geometry, materials, lights, camera parameters, and MLT control parameters (e.g., `nBootstrap`, `largeStepProbability`, `maxDepth`).
*   **Output:** A rendered image (film) with accumulated radiance values.

**Data Structures:**
The implementation relies on specific sample structures to store the state of light transport paths [S2, unofficial]:

1.  **`PathSample`**: Stores samples for path vertices.
    *   `BSDFSample bsdfSample`: Contains `uComponent`, `uDir[0]`, `uDir[1]`.
    *   `float rrSample`: Russian roulette sample.
2.  **`LightingSample`**: Stores samples for light interactions.
    *   `BSDFSample bsdfSample`: Contains `uComponent`, `uDir[0]`, `uDir[1]`.
    *   `float lightNum`: Index of the selected light source.
    *   `LightSample lightSample`: Contains `uComponent`, `uPos[0]`, `uPos[1]`.
3.  **`MLTSample`**: The primary state container for a light path.
    *   `CameraSample cameraSample`: Contains `imageX`, `imageY`, `time`, `lensU`, `lensV`.
    *   `float lightNumSample`: Sample for light selection.
    *   `float lightRaySamples[5]`: Samples for light ray generation.
    *   `vector<PathSample> cameraPathSamples`: Samples for camera path vertices.
    *   `vector<PathSample> lightPathSamples`: Samples for light path vertices.
    *   `vector<LightingSample> lightingSamples`: Samples for lighting interactions.
    *   The vectors are initialized with `maxLength` (typically `maxDepth`) in the constructor [S2, unofficial].
    *   The sample also contains `maxDepth` `PathVertex` objects for camera and light paths to store the geometric state [S2, unofficial].
4.  **`MLTTask`**: Stores parallel task state.
    *   Fields: `progress reporter`, `pixel frequency`, `task number`, `pixel bounds` (`x0`, `x1`, `y0`, `y1`), `time bounds` (`t0`, `t1`), `mean intensity b`, `initial sample`, `scene`, `camera`, `renderer`, `mutex`, and `light distribution` [S2, unofficial].

**Algorithm Steps:**

1.  **Direct Lighting Phase (Optional):**
    If `doDirectSeparately` is true, render direct lighting separately using a `DirectLightingIntegrator`. Generate `nDirectPixelSamples` per pixel using a stratified sampler and render to the film [S2, unofficial].

2.  **Bootstrapping:**
    Generate `nBootstrap` random samples (seed paths). For each sample, compute the path radiance $L$ using a path tracer (e.g., `PathL()`). Compute the intensity $I = L.y()$ (luminance). Sum all intensities into `sumI`. Compute the mean intensity $b = \text{sumI} / \text{nBootstrap}$ [S2, unofficial].

3.  **Initial Sample Selection:**
    Generate a random offset `contribOffset = rng.RandomFloat() * sumI`. Iterate through the bootstrap samples, accumulating their intensities. Select the first sample where the accumulated intensity exceeds `contribOffset`. This becomes the initial sample for the Markov chain [S2, unofficial].

4.  **Task Setup:**
    Calculate `largeStepsPerPixel` as $\max(1, \lceil \text{largeStepProbability} \times \text{nPixelSamples} \rceil)$. If this value is $\ge \text{nPixelSamples}$, divide by 2. Ensure it is at least 1 and less than `nPixelSamples`. Calculate `nTasks = largeStepsPerPixel`. Calculate `largeStepRate = nPixelSamples / largeStepsPerPixel`. Create `MLTTask` objects for each task, passing the initial sample, mean $b$, and scene data [S2, unofficial].

5.  **MLT Loop (Per Task):**
    For each task, iterate $s$ from $0$ to `nTaskSamples` - 1, where `nTaskSamples = nPixels * largeStepRate`.
    
    *   **Mutation Step:**
        If $s \pmod{\text{largeStepRate}} == 0$, perform a **LargeStep** (resampling) using a shuffled pixel index. Otherwise, perform a **SmallStep** (mutation) on the current sample [S2, unofficial].
        
        *   **SmallStep:** Mutate `cameraSample.imageX` in $[x_0, x_1]$, `imageY` in $[y_0, y_1]$, `time` in $[t_0, t_1]$, `lensU`, `lensV$. For $i$ from $0$ to `maxDepth`-1, mutate `cameraPathSamples[i].bsdfSample` components and `rrSample`. Mutate `lightingSamples[i].bsdfSample` components, `lightNum`, and `lightSample` components. If bidirectional, mutate `lightNumSample`, `lightRaySamples[0..4]`, and `lightPathSamples[i]` components [S2, unofficial].
        
        *   **LargeStep:** Mutate `cameraSample.imageX` to $x$, `imageY` to $y$, `time` to $\text{Lerp}(\text{rng.RandomFloat()}, t_0, t_1)$, `lensU` and `lensV` to random floats. For $i$ from $0$ to `maxDepth`-1, mutate `cameraPathSamples[i].bsdfSample` components and `rrSample`. Mutate `lightingSamples[i].bsdfSample` components, `lightNum`, and `lightSample` components. If bidirectional, mutate `lightNumSample`, `lightRaySamples[0..4]`, and `lightPathSamples[i]` components [S2, unofficial].

    *   **Mutation Delta Calculation:**
        When mutating a value $v$ with bounds $[\text{min}, \text{max}]$:
        If $\text{min} == \text{max}$, set $v = \text{min}$.
        Else, calculate delta:
        $$ \delta = (\text{max} - \text{min}) \cdot b \cdot \exp(\log\text{Ratio} \cdot \text{rng.RandomFloat}()) $$
        where $a = 1/1024$, $b = 1/64$, $\log\text{Ratio} = -\log(b/a)$ [S2, unofficial].
        
        Update $v$:
        If $\text{rng.RandomFloat}() < 0.5$, $v += \delta$; if $v \ge \text{max}$, $v = \text{min} + (v - \text{max})$.
        Else, $v -= \delta$; if $v < \text{min}$, $v = \text{max} - (\text{min} - v)$.
        If $v < \text{min}$ or $v \ge \text{max}$, set $v = \text{min}$ [S2, unofficial].

    *   **Evaluate Proposed Sample:**
        Compute radiance $L_{\text{proposed}}$ and intensity $I_{\text{proposed}}$ for the mutated sample [S2, unofficial].

    *   **Acceptance Probability:**
        Calculate acceptance probability $a$:
        $$ a = \min\left(1, \frac{I_{\text{proposed}}}{I_{\text{current}}}\right) $$
        where $I$ is the intensity (luminance) of the path [S2, unofficial].

    *   **Splatting:**
        Before splatting, check if $I_{\text{current}}$ or $I_{\text{proposed}}$ is zero or if $1/I$ is infinite. Skip splatting if so [S2, unofficial].
        
        If $I_{\text{current}} > 0$ and finite, splat $\frac{b}{\text{nPixelSamples}} \cdot \frac{L_{\text{current}}}{I_{\text{current}}}$ to the film at the camera sample position with weight $(1 - a)$.
        If $I_{\text{proposed}} > 0$ and finite, splat $\frac{b}{\text{nPixelSamples}} \cdot \frac{L_{\text{proposed}}}{I_{\text{proposed}}}$ with weight $a$ [S2, unofficial].

**Parameters:**
*   `largeStepProbability`: Probability of performing a large step vs. a small step. Default: 0.25 [S2, unofficial].
*   `perPixelSamples`: Total number of samples per pixel. Default: 100 [S2, unofficial].
*   `nBootstrap`: Number of random samples generated during bootstrapping. Default: 100,000 [S2, unofficial].
*   `nDirectPixelSamples`: Number of direct lighting samples per pixel. Default: 4 [S2, unofficial].
*   `maxConsecutiveRejects`: Maximum number of consecutive rejected mutations before forcing an acceptance or reset. Default: 512 [S2, unofficial].
*   `maxDepth`: Maximum depth of light paths. Default: 7 [S2, unofficial].

**Pitfalls:**
*   **Heterogeneous Media:** The Metropolis-Hastings acceptance probability cannot be evaluated in closed form if the target distribution $\pi(\bar{x})$ cannot be evaluated exactly. This occurs in scenes with heterogeneous participating media where transmittance is analytically intractable [S1].
*   **Division by Zero:** Check for zero or infinite intensity before splatting to avoid NaNs or infinities in the image [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Metropolis Light Transport was introduced by Veach and Guibas in 1997 [S1]. It is based on using Markov Chain Monte Carlo (MCMC) to sample light paths $\{\bar{x}_i\}_{i \ge 1}$ according to a target distribution $\pi(\bar{x})$ proportional to the image contribution of the paths [S1]. The path integral is estimated by the average $\hat{I}_j = \frac{1}{N} \sum_{i=1}^N W_j f(\bar{x}_i)$, where $Z = \int_\Omega L(\bar{x}) d\mu(\bar{x})$ is an unknown normalizing constant [S1]. The scalar contribution function $L(\bar{x})$ is usually constructed to be similar to the luminance of the path contribution function $f(\bar{x})$ [S1].

## Sources
[S1] Pseudo-marginal metropolis light transport, https://doi.org/10.1145/2820903.2820922, fetched 2026-10-30
[S2] brianrli/pbrt-photon-beams, https://github.com/brianrli/pbrt-photon-beams, fetched 2026-10-27 (unofficial)
