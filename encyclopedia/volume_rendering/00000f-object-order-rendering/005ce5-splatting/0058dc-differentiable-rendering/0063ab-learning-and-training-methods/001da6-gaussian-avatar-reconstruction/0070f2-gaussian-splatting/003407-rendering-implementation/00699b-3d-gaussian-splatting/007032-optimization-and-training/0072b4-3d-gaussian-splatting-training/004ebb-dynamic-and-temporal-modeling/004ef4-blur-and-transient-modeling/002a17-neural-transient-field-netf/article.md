# Neural Transient Field (NeTF)


## Overview
Neural Transient Field (NeTF) is a neural modeling framework for Non-Line-of-Sight (NLOS) imaging that recovers the 5D transient function in both spatial location and direction, implicitly represented by a Multi-Layer Perceptron (MLP) [S1]. Unlike NeRF, which parametrizes input training data along lines (rays), NeTF parametrizes input training data on spherical wavefronts [S1]. The final reconstruction of the hidden scene corresponds to a 3D density volume implicitly represented by the MLP [S1]. NeTF is designed to handle sparse and unbalanced sampling inherent in NLOS setups, producing sharper edges in semi-occluded scenes and separating distinct mesh parts better than methods like DLCT [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Sparse set of detection spots on a relay wall, capturing transient responses (time-of-flight histograms) for various incident and exit angles. The data is parameterized on spherical wavefronts rather than Cartesian rays [S1].
*   **Output:** A 3D density volume representing the hidden scene, which can be converted to a mesh via thresholding and Marching Cubes [S1].

**Data Structures:**
*   **Neural Network:** A Multi-Layer Perceptron (MLP) [S1]. The specific architecture (number of layers, hidden units, activation functions) is not detailed in the provided sources [G: Standard MLP architecture for implicit fields, e.g., ReLU activations, is assumed].
*   **Sampling Data:** A set of $(x, y, t)$ measurements from SPAD (Single-Photon Avalanche Diode) arrays, where $x, y$ are spatial coordinates on the relay wall and $t$ is time [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize the MLP parameters. [G: Standard random initialization for MLP weights].
2.  **Spherical Parameterization:** For each training sample, map the NLOS measurement geometry to spherical coordinates. Unlike ray-based rendering, the input data is treated as originating from/arriving at spherical wavefronts [S1].
3.  **Forward Pass:** Query the MLP with the spherical coordinates and time information to predict the transient density/color values [S1].
4.  **Loss Calculation:** Compute the loss between the predicted transients and the measured transients at every measuring spot on the relay wall [S1].
5.  **Optimization with MCMC:** Use Markov Chain Monte Carlo (MCMC) to account for sparse and unbalanced sampling in the transient field. MCMC enables more reliable volume density estimation [S1]. [G: Specific MCMC implementation details (e.g., Metropolis-Hastings kernel, proposal distribution) are not provided in the sources].
6.  **Two-Stage Training (Optional but recommended for sparse data):**
    *   **Stage 1:** Train using all initial samples to obtain an initial reconstruction [S1].
    *   **Calculate Loss Map:** Compute the loss function between predicted and measured transients at every measuring spot [S1].
    *   **Normalize to PDF:** Normalize the calculated loss to form a Probability Density Function (PDF) [S1]. Higher loss implies undersampling [S1].
    *   **Resample:** Resample detection spots using the PDF. Higher loss corresponds to higher PDF and is more densely sampled [S1].
    *   **Stage 2:** Build a new training dataset from the resampled spots and retrain the network to refine reconstruction [S1].
7.  **Surface Extraction:** Map the implicit volume to surfaces by thresholding the density values followed by Marching Cubes [S1].

**Formulas:**
*   **Spherical Parameterization:** The input data is parameterized on spherical wavefronts. Specific mathematical formulation of the spherical coordinate transformation is not provided in the sources [S1].
*   **Loss Normalization:** The loss $L_i$ at spot $i$ is normalized to form a PDF $P_i$:
    $$ P_i = \frac{L_i}{\sum_j L_j} $$
    [S1]

**Parameters:**
*   **Sampling Resolution:** Tested on 256x256, 16x16, 8x8, and 4x4 spots on the relay wall [S1].
*   **Thresholding:** Not given in the sources. [G: A standard threshold (e.g., 0.5 or based on density distribution) is typically used for Marching Cubes].

**Edge Cases and Pitfalls:**
*   **Confocal vs. Non-Confocal:** NeTF does not separately treat confocal and non-confocal setups in its current form [S1].
*   **Occlusion:** Sparse sampling can lead to strong reconstruction artifacts if occlusions are not handled properly. NeTF uses MCMC and two-stage training to mitigate this [S1].
*   **Computational Cost:** Learning-based techniques like NeTF are substantially more computationally expensive than previous optimization techniques [S1].
*   **Noise:** While NeTF reduces noise compared to SOTA methods like Phasor Field, brute-force Marching Cubes on the implicit volume may lead to noise on smooth surfaces [S1].

## Subtopics

### Two-stage Training
A training strategy to address sampling bias (undersampling) in NLOS reconstruction by resampling detection spots based on reconstruction loss [S1]. It involves an initial training phase, calculating a loss map, normalizing it to a PDF, and resampling spots to retrain [S1]. It is particularly effective for recovering fine details (e.g., bunny ears) that are missing in one-stage training due to occlusion or sparse viewpoints [S1].

## Operating characteristics
*   **Accuracy:** On the Bunny dataset (confocal), NeTF achieved a Mean Absolute Error (MAE) of 2.65 cm, compared to 2.38 cm for DLCT and 2.43 cm for F-K [S1]. On the Bunny dataset (non-confocal), NeTF achieved an MAE of 7.45 cm, compared to 3.77 cm for FBP and 7.02 cm for BP [S1]. On the Z letter scene (non-confocal), NeTF achieved an MAE of 0.60 cm, compared to 0.46 cm for FBP [S1].
*   **Visual Quality:** NeTF produces sharper edges for both front and back planes in semi-occluded scenes compared to Phasor Field and DLCT/F-K [S1]. NeTF manages to separate different parts of a mesh, whereas DLCT exhibits adhesion between different parts [S1]. NeTF reconstruction incurs much lower noise than SOTA methods, similar to NeRF [S1].
*   **Robustness:** NeTF performs reasonably well even with very sparse sampling spots (e.g., 8x8 or 16x16) [S1].
*   **Performance:** Learning-based techniques like NeTF are substantially more computationally expensive than previous optimization techniques [S1].
*   **Scaling:** Not given in the sources. [G: Typically scales with the number of MLP queries and training iterations].

## Background
NeTF was introduced as a novel neural modeling framework for NLOS imaging, analogous to Neural Radiance Fields (NeRF) but adapted for spherical wavefronts instead of Cartesian rays [S1]. It was tested on synthetic and real NLOS data, including the Bunny, Lucy, Indonesian, Bike, Letter Z, and semi-occluded scenes [S1]. The method leverages MCMC to handle sparse sampling and two-stage training to address sampling bias [S1].

## Sources
[S1] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
