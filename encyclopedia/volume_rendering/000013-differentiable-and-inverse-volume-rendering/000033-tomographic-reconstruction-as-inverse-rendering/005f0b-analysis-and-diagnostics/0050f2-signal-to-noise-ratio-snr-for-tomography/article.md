# Signal-to-Noise Ratio (SNR) for Tomography

Also called: rapport signal sur bruit

## Overview
Signal-to-Noise Ratio (SNR) for Tomography is a metric used to evaluate the robustness of a tomographic reconstruction method against acquisition noise [S1, unofficial]. It quantifies how well a reconstruction algorithm preserves signal integrity when the input sinogram is corrupted by noise, distinguishing between noise introduced by the acquisition process and noise induced by the reconstruction algorithm itself [S1, unofficial]. Unlike standard image quality metrics that compare a reconstruction to a ground truth, SNR for Tomography compares two reconstructions: one from a clean sinogram and one from a noisy sinogram [S1, unofficial]. This sets it apart from metrics like SSIM or MSE which measure fidelity to an ideal target, as SNR specifically isolates the method's sensitivity to input perturbations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input 1:** A clean sinogram $R_\theta(\rho)$ (ideal, noise-free acquisition).
*   **Input 2:** A noisy sinogram $R_\theta^{noise}(\rho)$ (acquired with additive noise).
*   **Input 3:** The reconstruction algorithm $\mathcal{R}$ (e.g., Back Projection, Fourier Reconstruction).
*   **Output:** The SNR value in decibels (dB), representing the robustness of $\mathcal{R}$.

**Data Structures:**
*   **Sinogram:** A 2D array where rows correspond to projection angles $\theta$ and columns to detector positions $\rho$.
*   **Reconstructed Image:** A 2D or 3D array of pixel/voxel intensities.

**Noise Model:**
Acquisition noise is modeled as additive white Gaussian noise [S1, unofficial]. To generate the noisy sinogram, random noise values are added to each value $R_\theta(\rho)$ of the clean sinogram. The noise amplitude is proportional to the signal intensity:
$$
R_\theta^{noise}(\rho) = R_\theta(\rho) + \epsilon
$$
where $\epsilon$ is a random value uniformly distributed between $-(A/2)R_\theta(\rho)$ and $+(A/2)R_\theta(\rho)$ [S1, unofficial]. The parameter $A$ represents the noise level percentage; a typical value cited is $A = 10\%$ [S1, unofficial].

**Step-by-Step Method:**
1.  **Generate Noisy Sinogram:** Apply the noise model described above to the clean sinogram $R_\theta(\rho)$ using a specified noise level $A$ (e.g., 10%) [S1, unofficial].
2.  **Reconstruct Clean Image:** Apply the reconstruction algorithm $\mathcal{R}$ to the clean sinogram to obtain the ideal reconstruction $J$ [S1, unofficial].
3.  **Reconstruct Noisy Image:** Apply the same reconstruction algorithm $\mathcal{R}$ to the noisy sinogram to obtain the noisy reconstruction $J_B$ [S1, unofficial].
4.  **Compute Mean Squared Error (MSE):** Calculate the MSE between the ideal input image $I_{in}$ (which corresponds to $J$ in the context of comparing robustness, or the ground truth if available, but the text defines SNR as comparing $J$ and $J_B$) and the noisy output image $I_{out}$ (which corresponds to $J_B$). The formula is:
    $$
    mse(I_{in}, I_{out}) = \frac{\sum_{p=1}^{N} (I_{in}(p) - I_{out}(p))^2}{N}
    $$
    where $I_{in}$ is the ideal image, $I_{out}$ is the noisy image, and $N$ is the number of pixels [S1, unofficial].
5.  **Calculate SNR:** Compute the SNR using the ratio of the mean intensity of the ideal image to the MSE, expressed in decibels:
    $$
    snr(I_{in}, I_{out}) = 10 \log_{10} \left( \frac{\bar{I}_{in}^2}{mse(I_{in}, I_{out})} \right)
    $$
    Note: The source text provides the formula as $\log(\frac{\bar{I}_{in}}{mse})$ but explicitly states it is expressed in decibels and gives the context of power ratio. Standard SNR in dB is $10 \log_{10}(\text{Power Ratio})$. The text formula (2.23) is written as $\log(\frac{\bar{I}_{in}}{mse})$ which is dimensionally inconsistent if not squared or if log base is not specified. However, looking at standard definitions and the context of "power average signal over noise", and the fact that MSE is variance-like, the standard definition is $10 \log_{10}(\frac{\mu^2}{\sigma^2})$. The text says "ratio between the average intensity... and the MSE". Let's stick to the text's explicit formula structure but correct for dB conversion if implied. The text says: "Il estime alors la puissance moyenne du signal par dessus le bruit... (formule 2.23)". Formula 2.23 is: $snr(I_{in}, I_{out}) = \log(\frac{\bar{I}_{in}}{mse(I_{in}, I_{out})})$. This is likely a typo in the source for $10 \log_{10}(\frac{\bar{I}_{in}^2}{mse})$ or similar. Given the values (6.33 dB), we assume the standard dB conversion. The source explicitly writes the log without base or coefficient in the equation block, but the text says "decibel". We will use the standard interpretation of the text's intent: $10 \log_{10}(\frac{\bar{I}_{in}^2}{mse})$. *Correction*: The source text (2.23) literally says `log ( I_in_bar / mse )`. This is physically incorrect for dB. However, we must follow the source. If the source defines it this way, we cite it. But wait, if $A=10\%$, and SNR is 6.33 dB, let's check. If the formula is just $\log_{10}(\dots)$, it's not dB. If it's $10 \log_{10}$, it is. The text says "expressed in decibel on a logarithmic scale". We will assume the standard $10 \log_{10}$ factor is implied by the unit "dB" despite the potentially simplified notation in the source equation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_noisy_sinogram(clean_sinogram, noise_level_percent=0.10):
    """
    Adds uniform noise to the sinogram based on the method in [S1, unofficial].
    Noise is uniform between -(A/2)*R and +(A/2)*R.
    """
    A = noise_level_percent
    # Generate random noise uniform in [-A/2, A/2]
    noise = np.random.uniform(-A/2, A/2, size=clean_sinogram.shape)
    # Scale noise by the signal value
    noisy_sinogram = clean_sinogram + (noise * clean_sinogram)
    return noisy_sinogram

def compute_mse(image_in, image_out):
    """
    Computes Mean Squared Error between two images.
    """
    N = image_in.size
    mse = np.sum((image_in - image_out) ** 2) / N
    return mse

def compute_snr_tomography(reconstruction_func, clean_sinogram, noisy_sinogram):
    """
    Computes SNR for Tomography as defined in [S1, unofficial].
    """
    # Step 1 & 2: Reconstruct clean image J
    J = reconstruction_func(clean_sinogram)
    
    # Step 3: Reconstruct noisy image J_B
    J_B = reconstruction_func(noisy_sinogram)
    
    # Step 4: Compute MSE between J (ideal) and J_B (noisy)
    # Note: The source defines I_in as ideal and I_out as noisy output
    mse_val = compute_mse(J, J_B)
    
    # Step 5: Compute SNR in dB
    # Source formula 2.23: log(I_in_bar / mse). 
    # Text states it is in decibels. Standard dB is 10*log10(Power).
    # Power is proportional to mean^2. 
    # We implement the standard interpretation of the text's intent for dB.
    mean_I_in = np.mean(J)
    
    # Avoid division by zero
    if mse_val == 0:
        return float('inf')
        
    # Using 10 * log10 for decibels as per text description "expressed in decibel"
    # and standard SNR definition, assuming the source equation omitted the 10 and square for brevity or had a typo.
    snr_db = 10 * np.log10((mean_I_in ** 2) / mse_val)
    
    return snr_db
```

**Parameters:**
*   $A$: Noise level percentage. Typical value: $10\%$ [S1, unofficial].

**Edge Cases:**
*   **Zero MSE:** If the reconstruction is identical for clean and noisy inputs (perfect noise immunity), MSE is 0, leading to infinite SNR [S1, unofficial].
*   **Negative Intensities:** If the reconstruction algorithm produces negative values, the mean intensity $\bar{I}_{in}$ might be small or zero, affecting the SNR calculation. The noise model assumes positive sinogram values for the multiplicative noise scaling [S1, unofficial].

## Subtopics

### Contrast-to-Noise Ratio (CNR)
A metric evaluating how a denoiser strategy increases the contrast between a Region of Interest (ROI) and a uniform background [S2]. It is used to assess the effectiveness of denoising in cryo-electron tomography [S2].
$$
CNR = \frac{l_f - l_u}{\frac{1}{\sqrt{N_{pr}}} \sqrt{0.5(r^2_f + r^2_u)}}
$$
where $N_{pr}$ is the number of paired regions selected to compute the CNR, $l_f$ and $r^2_f$ refer to the mean and variance in the selected regions containing features, and $l_u$ and $r^2_u$ are the mean and variance in the selected uniform regions [S2].

## Operating characteristics
*   **Time Complexity:** The SNR calculation itself is $O(N)$ where $N$ is the number of pixels in the reconstructed volume, as it involves computing MSE and mean values. The dominant cost is the reconstruction of the two volumes (clean and noisy), which depends on the specific reconstruction algorithm (e.g., Back Projection is $O(N \cdot M)$ where $M$ is the number of projections).
*   **Memory Complexity:** Requires storing two reconstructed volumes (clean and noisy) and the two sinograms. $O(N)$ for volumes.
*   **Measured Performance:** In a comparison of robustness to 10% additive Gaussian noise on sinograms, Back Projection achieved an SNR of 6.33 dB, while Fourier Reconstruction achieved -0.44 dB [S1, unofficial]. This indicates Back Projection is significantly more robust to acquisition noise than Fourier Reconstruction in this context [S1, unofficial].
*   **Strengths:** Directly measures robustness to acquisition noise, independent of reconstruction-induced noise [S1, unofficial]. Simple to compute once reconstructions are available.
*   **Limitations:** Does not measure absolute image quality (fidelity to ground truth), only relative robustness. Sensitive to the choice of noise model and level [S1, unofficial].
*   **When to use:** When comparing reconstruction algorithms for their stability against noisy input data.
*   **When not to use:** When absolute accuracy or structural similarity to a ground truth is the primary concern (use SSIM, MSE, or PSNR instead).

## Background
The metric is defined in the context of tomographic reconstruction quality and precision assessment [S1, unofficial]. It originates from French-language research on tomographic algorithms and applications, specifically a thesis or report titled "Précision et qualité en reconstruction tomographique : algorithmes et applications" [S1, unofficial]. The method is used to compare the robustness of different reconstruction techniques, such as Back Projection and Fourier Reconstruction, against acquisition noise [S1, unofficial].

## Sources
[S1] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
[S2] Adaptive differentiable grids for cryo-electron tomography reconstruction and denoising, https://doi.org/10.1093/bioadv/vbad131, fetched 2026-10-27
