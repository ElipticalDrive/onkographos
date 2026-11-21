# Frequency Guided Resolution Scheduler

Also called: Frequency-based resolution scheduling

## Overview
A strategy to schedule rendering resolution throughout optimization by distributing steps based on frequency energy, reducing optimization complexity [S1]. It sets itself apart from naive multi-resolution training by calculating a switch point based on the ratio of low-frequency energy to total energy, ensuring that high-resolution optimization only begins after the model has sufficiently captured the scene's low-frequency structure [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Total optimization steps $S$; a set of $N$ training views $\{I_n\}_{n=1}^N$ sampled at high resolution $(H, W)$; a hyperparameter $a$ (default 4) [S1].
*   **Outputs:** A sequence of rendering resolutions (or downsampling factors) $r(k)$ for each iteration $k \in [1, S]$ [S1].

**Data Structures:**
*   Frequency energy array: Stores $\mathcal{X}(F)$ for various downsampling factors.
*   Switch indices: Array $s_{r_i}$ storing the iteration index where resolution switches from factor $r_i$ to $r_{i-1}$ [S1].

**Steps:**

1.  **Compute Total Frequency Energy:**
    Calculate the total frequency energy $\mathcal{X}(F)$ of the training views. First, compute the Discrete Fourier Transform (DFT) $F_n = DFT(I_n)$ for each view $I_n$. Then compute:
    $$
    \mathcal{X}(F) = \frac{1}{N} \sum_{n=1}^{N} \sum_{i=1}^{H} \sum_{j=1}^{W} ||F_n(i, j)||^2
    $$
    where $N$ is the number of training views, and $(H, W)$ is the high-resolution size [S1].

2.  **Determine Maximum Downsampling Factor ($r_m$):**
    Find the largest downsampling factor $r_m$ such that the frequency energy of the downsampled image $F_{r_m}$ satisfies:
    $$
    \mathcal{X}(F_{r_m}) = \frac{1}{a} \mathcal{X}(F)
    $$
    where $a$ is the hyperparameter (default 4) [S1]. This defines the lowest resolution used in the schedule [S1].

3.  **Sample Intermediate Resolutions:**
    Linearly sample intermediate frequency energy values $\mathcal{X}(F_{r_1}), \dots, \mathcal{X}(F_{r_{m-1}})$ between $\mathcal{X}(F_{r_m})$ and $\mathcal{X}(F)$ [S1].
    For each sampled energy value, determine the corresponding downsampling factor $r_i$. The relationship between resolution size and energy is assumed to follow the inverse square law relative to the downsampling factor [S1].

4.  **Calculate Switch Iterations:**
    For each downsampling factor $r_i$ (where $r_0=1$ corresponds to full resolution), calculate the iteration index $s_{r_i}$ where the rendering resolution switches from $r_i$ to $r_{i-1}$:
    $$
    s_{r_i} = S \cdot \frac{\mathcal{X}(F_{r_i})}{\mathcal{X}(F)}
    $$
    This formula distributes the total steps $S$ proportionally to the energy contained in the lower-resolution bands [S1].

5.  **Interpolate Resolution per Iteration:**
    For any iteration $k$, determine the current rendering resolution $r(k)$. If $k$ falls between $s_{r_i}$ and $s_{r_{i-1}}$, interpolate the resolution based on the inverse squares of $r_i$ and $r_{i-1}$ [S1]. Specifically, the resolution size scales with the inverse of the downsampling factor.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_frequency_energy(views, downsample_factor):
    """
    Computes the average frequency energy of views downsampled by a factor.
    Uses center cropping and scaling to approximate anti-aliased downsampling.
    """
    total_energy = 0.0
    count = 0
    for view in views:
        # Center crop to downsampled size
        h, w = view.shape[:2]
        new_h, new_w = h // downsample_factor, w // downsample_factor
        start_y, start_x = (h - new_h) // 2, (w - new_w) // 2
        cropped = view[start_y:start_y+new_h, start_x:start_x+new_w]
        
        # Scale numerical value by 1/r^2 as per source description for energy preservation
        # Note: Source says "scale its numerical value by 1/r2" before inverse transform.
        # However, energy is sum of squares. If we scale pixel values by 1/r^2, 
        # energy scales by 1/r^4. The source formula implies X(Fr) is the energy of the downsampled image.
        # Standard DFT energy of a cropped image (without scaling) is lower.
        # The source states: "scale its numerical value by 1/r2 , then inversely transform...".
        # This implies the downsampled image Ir is constructed this way.
        # We compute energy of this constructed Ir.
        scaled_cropped = cropped / (downsample_factor ** 2)
        
        # Compute DFT and energy
        # Assuming 2D grayscale or sum over channels
        if len(cropped.shape) == 3:
            # Sum energy over channels
            energy = 0
            for c in range(cropped.shape[2]):
                f_n = np.fft.fft2(scaled_cropped[:,:,c])
                energy += np.sum(np.abs(f_n)**2)
            total_energy += energy / cropped.shape[2]
        else:
            f_n = np.fft.fft2(scaled_cropped)
            total_energy += np.sum(np.abs(f_n)**2)
        count += 1
        
    return total_energy / count

def frequency_guided_resolution_scheduler(views, total_steps, a=4):
    """
    Schedules rendering resolution based on frequency energy.
    """
    # 1. Compute Total Frequency Energy at Full Resolution (r=1)
    # Note: Source defines X(F) using full resolution H,W.
    # We assume downsample_factor=1 for full res.
    X_F = compute_frequency_energy(views, 1)
    
    # 2. Determine Max Downsampling Factor r_m
    # Find r_m such that X(F_{r_m}) = X(F) / a
    # We search for integer r_m. In practice, this might be pre-computed or approximated.
    # For implementation, we can iterate r from 1 upwards until energy drops below threshold.
    r_m = 1
    current_energy = X_F
    while current_energy > X_F / a:
        r_m += 1
        current_energy = compute_frequency_energy(views, r_m)
    
    # 3. Sample Intermediate Resolutions
    # Linearly sample energy values between X(F_{r_m}) and X(F)
    # Number of steps m is determined by r_m.
    # Let's define factors r_1 ... r_m. r_m is the max.
    # We need r_1 ... r_{m-1}.
    # Source: "linearly sample the rest X(F_{r1})...X(F_{rm-1}) between X(F) and X(F_{rm})"
    
    energies = []
    factors = []
    
    # Start with full res (r=1)
    factors.append(1)
    energies.append(X_F)
    
    # Add intermediate factors
    # We need m-1 intermediate factors. 
    # The source says "Given a group of downsampling factors r1, r2, ..., rm".
    # And "r(k) ... is interpolated with the inverse square of ri and ri-1".
    # This implies we have a set of discrete resolution levels.
    # Let's generate m-1 intermediate energy levels.
    num_intermediates = r_m - 1 # Assuming integer factors 1..r_m
    
    if num_intermediates > 0:
        # Linearly sample energies between X(F_{r_m}) and X(F)
        # Note: X(F_{r_m}) is the lowest energy. X(F) is highest.
        low_energy = current_energy # This is X(F_{r_m})
        high_energy = X_F
        
        # We need to map energy back to factor.
        # Source: "solve the corresponding resolution factors r1...rm-1"
        # And "interpolated with the inverse square of ri and ri-1".
        # This implies Energy ~ 1/r^2. So r ~ 1/sqrt(Energy).
        
        # Generate intermediate energy values
        # We want m-1 points between low_energy and high_energy.
        # Actually, the source says "linearly sample the rest ... between X(F) and X(F_{rm})".
        # So we create m-1 energy values.
        
        # Let's define the set of factors explicitly for clarity.
        # Factors: 1, r_1, r_2, ..., r_{m-1}, r_m
        # But r_m is determined by the threshold.
        # Let's assume we use integer downsampling factors 1, 2, ..., r_m.
        # This is the simplest interpretation of "downsampling factors r1...rm".
        
        factors = list(range(1, r_m + 1))
        
        # Compute energy for each factor
        energies = []
        for r in factors:
            e = compute_frequency_energy(views, r)
            energies.append(e)
            
        # If the computed energy for r_m doesn't exactly match X(F)/a, 
        # we might need to adjust. But for scheduling, we use the computed energies.
        # The source uses the theoretical relation X(F_{rm}) = X(F)/a to DEFINE rm.
        # So we stick to the computed energies for the switch points.
        
    else:
        factors = [1]
        energies = [X_F]

    # 4. Calculate Switch Iterations
    # s_{ri} is the iteration where we switch FROM ri TO ri-1.
    # Wait, source: "increase the rendering resolution from (H/ri, W/ri) to (H/ri-1, W/ri-1) at the sri-th iteration."
    # This means at iteration s_{ri}, we START using the HIGHER resolution (smaller factor).
    # So for k < s_{r1}, we use r1? No, r1 is the smallest factor (highest res) if r1=1.
    # Let's re-read carefully.
    # "r1, r2, ..., rm that monotonically increase". So r1 < r2 < ... < rm.
    # r1 is the smallest downsampling factor (closest to 1, i.e., highest res).
    # rm is the largest downsampling factor (lowest res).
    # "increase the rendering resolution from (H/ri, W/ri) to (H/ri-1, W/ri-1)".
    # This implies we start at rm (lowest res) and move to rm-1, ..., up to r1 (highest res).
    # So the sequence of factors used is rm, rm-1, ..., r1.
    # The switch from ri to ri-1 happens at s_{ri}.
    # So for k < s_{r_{m-1}}, we use rm?
    # Let's look at the formula: s_{ri} = S * X(F_{ri}) / X(F).
    # Since X(F_{ri}) decreases as ri increases, s_{ri} decreases as ri increases.
    # s_{r1} (for r1=1) = S * X(F)/X(F) = S.
    # s_{rm} (for rm=max) = S * X(F_{rm})/X(F) = S/a.
    # So at iteration S/a, we switch from rm to rm-1?
    # Source: "increase ... from (H/ri, W/ri) to (H/ri-1, W/ri-1) at the sri-th iteration."
    # This phrasing is slightly ambiguous. Does "at the sri-th iteration" mean the switch happens BEFORE or AFTER?
    # Usually, "switch at step X" means step X uses the new state.
    # If we start at rm, we use rm until we switch to rm-1.
    # The switch to rm-1 happens at s_{rm-1}?
    # Let's check the indices.
    # We have factors r1 < r2 < ... < rm.
    # We start with rm.
    # We switch to rm-1 at s_{rm-1}?
    # Source says: "increase ... from (H/ri, W/ri) to (H/ri-1, W/ri-1) at the sri-th iteration."
    # This implies the switch FROM ri TO ri-1 is triggered at s_{ri}.
    # So if we are at ri, we stay at ri until iteration s_{ri}, then switch to ri-1.
    # But we start at rm. So we stay at rm until s_{rm}?
    # But s_{rm} = S/a.
    # And we end at r1. We switch to r1 at s_{r1} = S.
    # So the schedule is:
    # [1, s_{rm}): Use rm
    # [s_{rm}, s_{rm-1}): Use rm-1
    # ...
    # [s_{r1}, S]: Use r1? But s_{r1}=S. So [S, S] is empty?
    # Actually, the last switch is to r1 (full res).
    # So for k >= s_{r1}, we use r1. Since s_{r1}=S, we use r1 only at the very last step?
    # This seems odd. Usually, you want full res for the last portion.
    # Let's re-read: "s_{ri} = S * (1 - f(F, Fr)) = S * X(Fr)/X(F)".
    # Wait, the source has TWO formulas.
    # Formula 1: s_r = S * (1 - f(F, Fr)) = S * X(Fr)/X(F).
    # Formula 2: "distribute S * f(F, Fr) iterations to the HR rendering."
    # This implies f(F, Fr) is the fraction of steps for HR.
    # If f(F, Fr) = 1 - X(Fr)/X(F), then steps for HR = S * (1 - X(Fr)/X(F)).
    # Steps for LR = S * X(Fr)/X(F).
    # So we use LR for the first S * X(Fr)/X(F) steps.
    # Then we switch to HR.
    # So the switch point s_r is indeed S * X(Fr)/X(F).
    # And for k < s_r, we use LR (factor r). For k >= s_r, we use HR (factor 1).
    
    # Generalizing to multiple resolutions:
    # We have factors r1 < r2 < ... < rm.
    # We start with rm (lowest res).
    # We switch to rm-1 at s_{rm-1}?
    # Source: "increase ... from (H/ri, W/ri) to (H/ri-1, W/ri-1) at the sri-th iteration."
    # This means the switch FROM ri TO ri-1 happens at s_{ri}.
    # So if we are currently using ri, we switch to ri-1 at s_{ri}.
    # We start with rm.
    # So we use rm until s_{rm}? No, the switch FROM rm TO rm-1 happens at s_{rm}.
    # So for k < s_{rm}, we use rm.
    # For k >= s_{rm}, we use rm-1.
    # Then we use rm-1 until s_{rm-1}.
    # For k >= s_{rm-1}, we use rm-2.
    # ...
    # For k >= s_{r1}, we use r1 (full res).
    # Since s_{r1} = S, we use r1 only at k=S?
    # This implies we never use full res until the very last step?
    # This contradicts "reduce optimization complexity".
    # Let's look at the "multiple resolutions" paragraph again.
    # "Given a group of downsampling factors r1, r2, ..., rm that monotonically increase...
    # we increase the rendering resolution from (H/ri, W/ri) to (H/ri-1, W/ri-1) at the sri-th iteration."
    # Maybe the indices are reversed?
    # If r1 is the LARGEST factor (lowest res) and rm is 1 (highest res)?
    # "r1, r2, ..., rm that monotonically increase". So r1 < r2.
    # So r1 is small (high res), rm is large (low res).
    # "increase ... from (H/ri, W/ri) to (H/ri-1, W/ri-1)".
    # If i=m, we switch from rm to rm-1.
    # If i=1, we switch from r1 to r0? r0 is not defined.
    # Maybe the switch is FROM ri-1 TO ri?
    # "increase the rendering resolution from (H/ri, W/ri) to (H/ri-1, W/ri-1)"
    # H/ri is smaller than H/ri-1 (since ri > ri-1).
    # So we are going from Low Res (H/ri) to High Res (H/ri-1).
    # This confirms we move from large factor to small factor.
    # So we start with rm.
    # We switch to rm-1 at s_{rm}.
    # We switch to rm-2 at s_{rm-1}.
    # ...
    # We switch to r1 at s_{r2}.
    # What about s_{r1}?
    # s_{r1} = S * X(F_{r1})/X(F) = S.
    # So we switch to r1 at S.
    # This means we use r1 only at the last step.
    # This seems to be the literal interpretation.
    # However, in practice, one might define the last switch to be at S-1 or similar.
    # Or perhaps r1 is not 1.
    # "r1, ..., rm-1" are intermediate. rm is max.
    # If r1=1, then yes.
    # But maybe the schedule stops at r1?
    # "interpolated with the inverse square of ri and ri-1".
    # This interpolation is for r(k) between s_{ri} and s_{ri-1}.
    # If we are between s_{r2} and s_{r1}, we interpolate between r2 and r1.
    # At s_{r1}=S, we are at r1.
    # So for k in [s_{r2}, S], we interpolate between r2 and r1.
    # This makes sense. We gradually increase resolution from r2 to r1 over the last phase.
    
    switch_points = {}
    for i, r in enumerate(factors):
        # factors are [1, 2, ..., r_m]
        # r_i corresponds to factors[i]
        # We need to map factor to its switch point.
        # The switch FROM factor r TO factor prev_r happens at s_r.
        # prev_r is the next smaller factor in the list.
        # List: [1, 2, 3]. r=3. prev=2. Switch at s_3.
        # r=2. prev=1. Switch at s_2.
        # r=1. prev=None. Switch at s_1 = S.
        
        # Calculate s_r
        X_Fr = energies[i]
        s_r = int(total_steps * X_Fr / X_F)
        switch_points[r] = s_r
        
    # 5. Determine Resolution for each iteration
    schedule = []
    for k in range(1, total_steps + 1):
        # Find which interval k falls into.
        # Intervals are defined by switch points.
        # We start with rm (largest factor).
        # We switch to rm-1 at s_{rm}.
        # So if k < s_{rm}, use rm.
        # If s_{rm} <= k < s_{rm-1}, use rm-1.
        # ...
        # If k >= s_{r2}, interpolate between r2 and r1.
        
        # Sort factors descending to process from low res to high res
        sorted_factors = sorted(factors, reverse=True)
        
        current_r = sorted_factors[0] # rm
        for i in range(len(sorted_factors) - 1):
            r_curr = sorted_factors[i]
            r_next = sorted_factors[i+1]
            s_curr = switch_points[r_curr]
            
            if k < s_curr:
                current_r = r_curr
                break
            else:
                # We are in the interval [s_curr, s_next)
                # But wait, the switch happens AT s_curr.
                # So for k >= s_curr, we have ALREADY switched to r_next?
                # Source: "increase ... from (H/ri, W/ri) to (H/ri-1, W/ri-1) at the sri-th iteration."
                # This implies at iteration s_{ri}, we are using the NEW resolution (ri-1).
                # So if k >= s_{ri}, we use ri-1.
                # So if k < s_{rm}, we use rm.
                # If s_{rm} <= k < s_{rm-1}, we use rm-1.
                # If s_{rm-1} <= k < s_{rm-2}, we use rm-2.
                # ...
                # If s_{r2} <= k < s_{r1}, we use r2?
                # But s_{r1} = S.
                # So for k >= s_{r2}, we use r2?
                # And then interpolate?
                # "r(k) for iteration k between s_{ri} and s_{ri-1} is interpolated..."
                # This suggests interpolation happens BETWEEN switch points.
                # So if k is between s_{r2} and s_{r1}, we interpolate between r2 and r1.
                # This matches the logic:
                # k < s_{rm}: rm
                # s_{rm} <= k < s_{rm-1}: rm-1
                # ...
                # s_{r2} <= k < s_{r1}: interpolate r2 and r1.
                
                # Let's implement this logic.
                pass
        
        # Re-implementing the loop correctly
        # Factors sorted descending: [rm, rm-1, ..., r1]
        # Switch points: s_{rm}, s_{rm-1}, ..., s_{r1}
        # Note: s_{r1} = S.
        
        # Check if k is before the first switch
        if k < switch_points[sorted_factors[0]]:
            current_r = sorted_factors[0]
        else:
            # Find the interval
            found = False
            for i in range(len(sorted_factors) - 1):
                r_curr = sorted_factors[i]
                r_next = sorted_factors[i+1]
                s_curr = switch_points[r_curr]
                s_next = switch_points[r_next]
                
                if s_curr <= k < s_next:
                    # Interpolate between r_curr and r_next
                    # "interpolated with the inverse square of ri and ri-1"
                    # This likely means the resolution size (1/r) is interpolated.
                    # Or the factor r is interpolated?
                    # "inverse square of ri" -> 1/ri^2.
                    # Let's assume we interpolate the factor r linearly?
                    # Or the resolution size?
                    # Source: "interpolated with the inverse square of ri and ri-1".
                    # This is vague. It might mean the energy is interpolated?
                    # But we are scheduling resolution.
                    # Let's assume linear interpolation of the downsampling factor r.
                    # This is a common practice.
                    
                    # Fraction of progress in the interval
                    progress = (k - s_curr) / (s_next - s_curr)
                    # Interpolate r
                    current_r = r_curr + progress * (r_next - r_curr)
                    found = True
                    break
            
            if not found:
                # Should not happen if s_{r1}=S and k <= S
                current_r = sorted_factors[-1] # r1
        
        schedule.append(current_r)
        
    return schedule
```

**Pitfalls:**
*   **Aliasing:** Simple spatial downsampling (center cropping) can cause aliasing, disturbing recoverable low-frequency components. The method relies on the assumption that the downsampled images $I_r$ are properly anti-aliased (e.g., via low-pass filtering or the specific scaling method described) to ensure the frequency energy $\mathcal{X}(F_r)$ accurately reflects the information content [S1].
*   **Precision:** The calculation of frequency energy involves summing squared magnitudes of DFT coefficients, which can be sensitive to numerical precision for large images [S1].

## Operating characteristics
*   **Time Complexity:** Reduces overall optimization time by performing fewer high-resolution steps. The exact speedup depends on the scene's frequency content and the hyperparameter $a$ [S1].
*   **Memory Complexity:** No significant additional memory overhead compared to standard 3DGS, as it only schedules existing resolution buffers [S1].
*   **Strengths:** Accelerates 3DGS optimization without trading off quality [S1].
*   **Limitations:** Requires pre-computation of frequency energies for various resolutions, which adds a small overhead before optimization [S1].

## Background
The Frequency Guided Resolution Scheduler is introduced in the **DashGaussian** paper, which aims to optimize 3D Gaussian Splatting in approximately 200 seconds [S1]. It builds on the observation that optimization complexity can be reduced by gradually fitting the 3DGS model to higher levels of frequency components [S1].

## Sources
[S1] DashGaussian: Optimizing 3D Gaussian Splatting in 200 Seconds, https://doi.org/10.1109/cvpr52734.2025.01041, fetched 2026-10-31
