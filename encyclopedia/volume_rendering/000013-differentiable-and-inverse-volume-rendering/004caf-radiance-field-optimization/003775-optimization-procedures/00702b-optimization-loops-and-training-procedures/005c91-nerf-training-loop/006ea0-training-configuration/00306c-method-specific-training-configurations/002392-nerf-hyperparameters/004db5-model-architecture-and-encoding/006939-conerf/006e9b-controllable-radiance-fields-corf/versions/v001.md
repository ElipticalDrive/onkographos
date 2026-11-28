# Controllable Radiance Fields (CoRF)
## Overview
CoRF is a method for generalizable, 3D-aware, and motion-controllable synthesis of face dynamics, generating never-before-seen faces with controlled dynamic motions and viewpoints [S1]. It extends the NeRF paradigm by integrating a style-based generator that decouples identity and motion, allowing for intuitive control over facial expressions while maintaining 3D consistency [S1]. Unlike standard NeRFs that reconstruct a single static scene, CoRF learns a latent space for motion representations that can be interpolated or sampled to synthesize novel dynamic behaviors [S1].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Noise vector $z$, motion representation $m$, camera pose $\xi$, and 3D position $x$.
*   **Outputs:** Rendered RGB image $I_0$.

**Data Structures:**
*   **Motion Representation ($m$):** A vector representing facial expression or dynamic motion state.
*   **Noise Vector ($z$):** Latent vector sampled from a prior distribution to control identity and fine details.
*   **Style Vector ($w$):** Output of the noise mapping module, $w = f_z(z)$.
*   **Camera Pose ($\xi$):** Condition for the discriminator.

**Components:**
The generator $G$ is composed of three modules: $G = \{f_z, f_m, G_s\}$ [S1].
1.  **Noise Mapping ($f_z$):** Maps noise $z$ to style vector $w$.
2.  **Motion Mapping ($f_m$):** Embeds motion representation $m$ into a layered latent motion space.
3.  **Synthesis Module ($G_s$):** Renders the image using the combined style and motion features.

**Training Components:**
*   **Discriminator ($D$):** Ensures photo-realism.
*   **Regressor ($R$):** Extracts motion features from images to supervise motion control.
*   **Identity Encoder ($E$):** Extracts identity features to ensure consistency.

**Numbered Steps:**
1.  **Motion Embedding:** The motion representation $m$ is processed by the mapping network $f_m$ to produce a motion embedding [S1].
2.  **Style Generation:** The noise vector $z$ is processed by $f_z$ to produce the style vector $w$ [S1].
3.  **Feature Combination:** The motion embedding and style vector $w$ are combined via summation ($\oplus$) [S1].
4.  **Rendering:** The combined features are passed to the synthesis module $G_s$ along with position $x$ and camera pose $\xi$ to render the image $I_0$ [S1].
5.  **Motion Supervision:** The regressor $R$ estimates motion from the generated image. A motion reconstruction loss $\mathcal{L}_{motion}$ minimizes the distance between the estimated motion and the conditioned motion $m$ [S1].
6.  **Consistency Supervision:** To preserve consistency over views and poses, the regressor $R$ and identity encoder $E$ extract features from paired images (same identity, different motions). A consistency loss $\mathcal{L}_{consist}$ and identity loss $\mathcal{L}_{id}$ encourage these features to be similar [S1].
7.  **Adversarial Training:** The discriminator $D$ compares real and generated images, conditioned on camera pose $\xi$ and motion representation $m$, using an adversarial loss $\mathcal{L}_{adv}$ [S1].

**Formulas:**
Generator composition:
$$ G = \{f_z, f_m, G_s\} $$
where $f_z$ maps noise to style, $f_m$ maps motion, and $G_s$ is the synthesis module [S1].

Style vector definition:
$$ w = f_z(z) $$
where $w$ is combined with the motion embedding via summation [S1].

**Parameters:**
*   **Motion representation $m$:** Input to the generator and discriminator, representing facial expression/motion [S1].
*   **Noise vector $z$:** Input to the generator, processed by $f_z$ to produce style vector $w$ [S1].
*   **Style vector $w$:** Output of $f_z(z)$, combined with motion embedding [S1].
*   **Camera pose $\xi$:** Condition for the discriminator $D$ [S1].

**Edge Cases and Pitfalls:**
*   **Motion Control:** Effective motion control requires the embedding of $m$ into a layered latent space via $f_m$ [S1].
*   **Consistency:** Spatio-temporal consistency is ensured by the regressor $R$ and encoder $E$ [S1].
## Operating characteristics
Not given in the sources.
## Background
CoRF was proposed to address the limitations of existing 3D-aware generative models in synthesizing dynamic face content. While prior works like GRAF, $\pi$-GAN, StyleNeRF, CIPS-3D, and LolNeRF focus on static objects or random camera poses, CoRF specifically targets the generalizable, 3D-aware, and motion-controllable synthesis of face dynamics [S1]. It enables the generation of never-before-seen faces with controlled dynamic motions and viewpoints [S1].
## Sources
[S1] Controllable Radiance Fields for Dynamic Face Synthesis, https://doi.org/10.1109/3dv57658.2022.00075, fetched 2026-10-29
