# Porcine Eye Model Construction

Also called: Ex vivo porcine eye geometric model, Porcine eye optical model

## Overview
The Porcine Eye Model Construction method defines the precise geometric and optical parameters of an ex vivo porcine eye to enable accurate ray tracing for distortion correction in Optical Coherence Tomography (OCT) systems. It differs from human eye models by accounting for specific differences in corneal curvature, lens thickness, and gradient refractive index properties inherent to porcine tissue. This model serves as the physical substrate for the Thick Lens (Snell's Law Refraction) method, providing the surface curvatures, thicknesses, and refractive indices required to compute ray paths through the eye's optical components.

## How it works
The method constructs a multi-surface optical model by defining the geometry of the cornea and lens, and assigning refractive indices to the intervening media. The geometry is derived from published mean refractive powers and physical measurements, while the refractive indices are determined based on the operating wavelength (1050 nm).

### Inputs and Outputs
- **Inputs**: Refractive power of anterior and posterior corneal surfaces in diopters ($P$); refractive indices of air, cornea, aqueous humor, and vitreous humor at the operating wavelength; physical measurements of corneal thickness, anterior chamber depth, and lens dimensions.
- **Outputs**: A structured representation of the eye's optical surfaces, including radius of curvature and thickness for each interface, and the refractive indices of the media.

### Data Structures
The model is represented as a sequence of optical surfaces. Each surface entry contains:
- `Surface`: Identifier (e.g., "Anterior Cornea").
- `Radius`: Radius of curvature in millimeters (mm). Positive values indicate convex surfaces facing the incident light; negative values indicate concave surfaces. $\infty$ indicates a planar surface.
- `Thickness`: Axial distance to the next surface in millimeters (mm).

The specific layout for the porcine eye model is defined as follows:

| Surface | Radius (mm) | Thickness (mm) |
| :--- | :--- | :--- |
| Anterior Cornea | 8.68 | 0.833 |
| Posterior Cornea | 8.35 | 1.770 |
| Anterior Lens | 7.08 | 3.034 |
| Middle Lens | $\infty$ | 4.366 |
| Posterior Lens | -5.08 | 0.000 |

### Formulas and Calculations
The radius of curvature for each corneal surface is calculated from its refractive power using the following formula [S1]:

$$
R = \frac{(n_2 - n_1) \cdot 1000}{P}
$$

Where:
- $R$ is the radius of curvature in millimeters.
- $P$ is the refractive power of the surface in diopters (D).
- $n_1$ is the refractive index of the medium preceding the surface.
- $n_2$ is the refractive index of the medium following the surface.

**Anterior Cornea Calculation**:
- $P = 43.27$ D [S1].
- $n_1 = 1.0003$ (air at 1050 nm) [S1].
- $n_2 = 1.376$ (cornea) [S1].
- $R_{ca} = \frac{(1.376 - 1.0003) \cdot 1000}{43.27} \approx 8.68$ mm [S1].

**Posterior Cornea Calculation**:
- $P = -5.15$ D [S1].
- $n_1 = 1.376$ (cornea) [S1].
- $n_2 = 1.333$ (aqueous humor) [S1].
- $R_{cp} = \frac{(1.333 - 1.376) \cdot 1000}{-5.15} \approx 8.35$ mm [S1].

**Lens Geometry**:
- Anterior lens radius: 7.08 mm [S1].
- Posterior lens radius: -5.08 mm [S1].
- Total lens thickness: 7.40 mm, composed of an anterior region (3.034 mm) and a middle region (4.366 mm) [S1].
- The middle lens surface is planar ($\infty$ radius), separating the anterior and posterior lens regions [S1].

**Refractive Indices**:
- The approximate refractive index of the eye tissue ($n_{eye}$) is 1.344 at 1050 nm [S1].
- Cornea: 1.376 [S1].
- Aqueous humor: 1.333 [S1].
- Vitreous humor: Computed by optical design software based on 1050 nm properties [S1].

### Parameters
- **Operating Wavelength**: 1050 nm [S1].
- **Anterior Corneal Radius ($R_{ca}$)**: 8.68 mm [S1].
- **Posterior Corneal Radius ($R_{cp}$)**: 8.35 mm [S1].
- **Corneal Thickness**: 0.833 mm [S1].
- **Anterior Chamber Depth**: 1.77 mm [S1].
- **Lens Anterior Radius**: 7.08 mm [S1].
- **Lens Posterior Radius**: -5.08 mm [S1].
- **Lens Total Thickness**: 7.40 mm [S1].
- **Eye Refractive Index ($n_{eye}$)**: 1.344 [S1].

### Edge Cases and Pitfalls
- **Gradient Refractive Index (GRIN)**: The anterior and middle lens surfaces are designated as "Gradient 3" type in optical design software to model the gradient refractive index of the lens material, rather than treating the lens as a homogeneous medium [S1]. This requires specific handling in ray tracing algorithms that support GRIN media.
- **Sign Convention**: The posterior lens radius is negative (-5.08 mm), indicating a concave surface relative to the incident light direction from the anterior chamber. Implementations must adhere to the sign convention used in the source (standard optical design convention) to ensure correct refraction calculations.
- **Ex Vivo Conditions**: The model is specific to ex vivo porcine eyes. In vivo conditions or human eyes will have different parameters, requiring separate model construction [S1].

## Operating characteristics
- **Accuracy**: Provides high geometric accuracy for ex vivo porcine eye ray tracing at 1050 nm, enabling distortion correction in OCT images [S1].
- **Complexity**: The model involves 5 surfaces and GRIN lens handling, increasing computational complexity compared to simple homogeneous eye models.
- **Applicability**: Specifically designed for indirect viewing systems using ex vivo porcine eyes. Not directly applicable to human eyes or in vivo porcine eyes without parameter adjustment.

## Background
The Porcine Eye Model Construction method was developed to address the significant differences in curvature and thickness between human and porcine eyes, which contribute to distortion in OCT images [S1]. Since experiments were conducted on ex vivo porcine eyes, a specific geometric optical model was required to achieve accurate ray tracing for distortion correction [S1]. The parameters were derived from mean refractive powers reported in literature [25], physical measurements [25, 26], and lens dimensions [27]. The model was implemented in Zemax OpticStudio for optical design and simulation [S1].

## Sources
[S1] 3D fast ray-traced distortion correction of indirectly viewed intraoperative retinal OCT for surgical tool path reconstruction, https://europepmc.org/article/PMC/PMC13671697, fetched 2026-10-26
