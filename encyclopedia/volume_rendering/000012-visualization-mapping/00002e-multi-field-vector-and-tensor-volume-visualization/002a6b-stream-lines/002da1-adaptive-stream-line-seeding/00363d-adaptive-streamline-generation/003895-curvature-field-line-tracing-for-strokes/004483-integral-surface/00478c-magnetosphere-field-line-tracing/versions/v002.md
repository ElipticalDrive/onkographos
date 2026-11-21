# Magnetosphere Field Line Tracing

Also called: Earth Magnetosphere Dataset Generation Pipeline, Earth Magnetosphere Field Line Tracing, Earth data generation, dipole field line integration

## Overview
This method generates a synthetic dataset of Earth's magnetospheric magnetic field lines by tracing dipole field lines from ionospheric footpoints and magnetosheath draped lines, encoding field intensity to grayscale, and outputting the result as a compact JSON file [S1, unofficial]. It serves as a data preparation step for volumetric visualization, distinct from general stream line tracing by its specific geometric constraints (ionospheric seeding, magnetopause boundaries) and reliance on an analytical dipole model rather than a grid-based vector field [S1, unofficial]. It builds on [Integral Surface](11268-integral-surface.md) by providing the discrete point-cloud input required for surface generation.

## How it works
**Inputs:**
- `r_start`: Initial radial distance (default 3.0 $R_E$) [S1, unofficial].
- `theta0`: Co-latitude of the ionospheric footpoint (radians) [S1, unofficial].
- `phi0`: Azimuth (longitude) of the footpoint (radians) [S1, unofficial].
- `n_points`: Maximum number of sample points per field line [S1, unofficial].
- `step`: Fixed step size for Euler integration [G] (value not given in sources; typical values for magnetospheric scales are $0.01-0.1 R_E$).
- `N_FIELD_LINES`, `N_POINTS_PER_LINE`, `N_SHEATH_LINES`, `N_POINTS_SHEATH`: Global dataset parameters [S1, unofficial].
- `MAX_RECORDS`: Hard cap on total output points (default 5000) [S1, unofficial].
- `RANDOM_SEED`: Seed for reproducibility [S1, unofficial].

**Outputs:**
- A JSON array where the first record is metadata (`source='meta'`) and subsequent records are field line points [S1, unofficial].
- Each point record contains: `x`, `y`, `z` (position in $R_E$), `Bx`, `By`, `Bz` (field vector), `intensity` (field magnitude), `source` ('magnetosphere' or 'magnetosheath'), and `gray` (normalized grayscale intensity) [S1, unofficial].

**Data Structures:**
- `Record`: Dictionary with keys `x`, `y`, `z`, `Bx`, `By`, `Bz`, `intensity`, `source`, `gray`. All floats are rounded to a specified precision (e.g., 3 decimal places) for compression [S1, unofficial].
- `Metadata`: Dictionary with keys `source` ('meta'), `source_label`, `date`, `coord_units`, `field_units`, `planetary_ref`, `domain_scale`, `measured`, `data_type`, `mp_model`, `bs_model`, `B_model`, `notes` [S1, unofficial].

**Algorithm:**

1. **Initialize Global State:**
   - Set random seed: `random.seed(RANDOM_SEED)` [S1, unofficial].
   - Initialize `all_records` list [S1, unofficial].

2. **Trace Magnetospheric Dipole Field Lines:**
   - Define angular steps: `phi_steps = 8`, `theta_steps = N_FIELD_LINES // phi_steps` [S1, unofficial].
   - Loop `i_phi` from 0 to `phi_steps - 1`:
     - Calculate azimuth: `phi = (i_phi / phi_steps) * 2.0 * pi` [S1, unofficial].
     - Loop `i_th` from 0 to `theta_steps - 1`:
       - Calculate co-latitude: `theta = radians(25.0 + i_th * 30.0 / max(theta_steps - 1, 1))` [S1, unofficial].
       - Call `trace_dipole_field_line(theta, phi, N_POINTS_PER_LINE)` [S1, unofficial].
       - Append returned points to `all_records` [S1, unofficial].

3. **Trace Magnetosheath Lines:**
   - Call `generate_magnetosheath_lines(N_SHEATH_LINES, N_POINTS_SHEATH)` [S1, unofficial].
   - Append returned points to `all_records` [S1, unofficial].

4. **Encode Intensity to Grayscale:**
   - Extract all intensities: `intensities = [r['intensity'] for r in all_records]` [S1, unofficial].
   - Calculate range: `i_min = min(intensities)`, `i_max = max(intensities)` [S1, unofficial].
   - For each record `r` in `all_records`:
     - Normalize: `norm = (r['intensity'] - i_min) / max(i_max - i_min, 1e-9)` [S1, unofficial].
     - Apply gamma correction: `norm = max(0.0, min(1.0, norm)) ** 0.7` [S1, unofficial].
     - Compute gray value: `r['gray'] = int(round(norm * 255))` [S1, unofficial].

5. **Enforce Point Cap:**
   - If `len(all_records) > MAX_RECORDS`: truncate to `all_records[:MAX_RECORDS]` [S1, unofficial].

6. **Create Metadata:**
   - Construct `meta` dictionary with standard keys (`source='meta'`, etc.) [S1, unofficial].

7. **Output:**
   - Combine: `output = [meta] + all_records` [S1, unofficial].
   - Write to `OUTPUT_FILE` as JSON with compact separators (no whitespace) [S1, unofficial].

**`trace_dipole_field_line(theta0, phi0, n_points)` Function:**

1. **Initialize Position:**
   - Calculate starting position at `r_start = 3.0`:
     - `x = r_start * sin(theta0) * cos(phi0)`
     - `y = r_start * sin(theta0) * sin(phi0)`
     - `z = r_start * cos(theta0)` [S1, unofficial].

2. **Iterate for `n_points`:**
   - Calculate radial distance: `r = sqrt(x^2 + y^2 + z^2)` [S1, unofficial].
   - **Check Ionosphere Boundary:** If `r < 1.1`, break loop [S1, unofficial].
   - **Check Magnetopause Boundary:**
     - Calculate polar angle from sunward X axis: `theta_sun = atan2(sqrt(y^2 + z^2), max(x, -40.0))` [S1, unofficial].
     - Calculate magnetopause radius: `mp_r = magnetopause_radius(theta_sun)` [G] (function not defined in sources; must be implemented based on a chosen magnetopause model, e.g., Shue et al.) [S1, unofficial].
     - If `r > mp_r`, break loop [S1, unofficial].
   - **Evaluate Dipole Field:**
     - Calculate field components: `Bx, By, Bz = dipole_field(x, y, z)` [S1, unofficial].
     - Calculate magnitude: `mag = sqrt(Bx^2 + By^2 + Bz^2)` [S1, unofficial].
     - **Check Zero Field:** If `mag < 1e-9`, break loop [S1, unofficial].
   - **Record Point:**
     - Append `{'x': x, 'y': y, 'z': z, 'Bx': Bx, 'By': By, 'Bz': Bz, 'intensity': mag, 'source': 'magnetosphere'}` to local list [S1, unofficial].
   - **Euler Step:**
     - Update position:
       - `x += step * Bx / mag`
       - `y += step * By / mag`
       - `z += step * Bz / mag` [S1, unofficial].

3. **Return:** Local list of points [S1, unofficial].

**`dipole_field(x, y, z)` Function:**
- The magnetic field of a centered dipole aligned with the Z-axis is given by:
  $$ \mathbf{B} = \frac{\mu_0 m}{4\pi r^5} \left( 3z x, 3z y, 3z^2 - r^2 \right) $$
  where $r = \sqrt{x^2 + y^2 + z^2}$ and $m$ is the dipole moment [G] (standard physics formula; constant factors can be absorbed into `step` or ignored if only direction matters for tracing, but `intensity` requires magnitude) [S1, unofficial].
- In normalized units ($R_E$, arbitrary field units):
  $$ B_x = \frac{3xz}{r^5}, \quad B_y = \frac{3yz}{r^5}, \quad B_z = \frac{3z^2 - r^2}{r^5} $$ [G]

**`magnetopause_radius(theta_sun)` Function:**
- Not given in the sources. Must be implemented using a standard model (e.g., Shue et al. 1998) [G].

## Subtopics

### [Magnetosheath Draped Field Line Generation](12013-magnetosheath-draped-field-line-generation.md)
Generates draped interplanetary magnetic field (IMF) line samples within the magnetosheath region, bounded by the bow shock and the magnetopause. Unlike the parent method which traces single lines from ionospheric footpoints, this method seeds lines randomly in the dayside magnetosheath to capture the draped topology of the solar wind magnetic field. It is distinct from general stream line tracing by its specific geometric constraints (dayside-only seeding) and reliance on external magnetosheath field models.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of field lines and points per line ($O(N_{lines} \times N_{points})$) [S1, unofficial].
- **Memory Complexity:** Linear with respect to the total number of output points, capped at `MAX_RECORDS` [S1, unofficial].
- **Measured Performance:** At precision=3, a reduction of 8–15% in file size is observed relative to default 4–5 decimal place output, with negligible visual impact for scene-scale data [S1, unofficial].
- **Scaling:** The method is designed for offline dataset generation. The visualizer enforces a secondary cap of 5000 records if the generated dataset exceeds `MAX_RECORDS` (default 5000) [S1, unofficial].
- **Strengths:** Simple, fast generation of synthetic magnetospheric data; compact JSON output; handles boundary conditions (ionosphere, magnetopause) explicitly [S1, unofficial].
- **Limitations:** Relies on a simplified dipole model; does not account for time-varying solar wind conditions or realistic magnetohydrodynamic effects; requires external implementation of magnetopause and magnetosheath models [S1, unofficial].
- **When to Use:** When a synthetic, reproducible dataset of magnetospheric field lines is needed for testing visualization pipelines or educational purposes [S1, unofficial].
- **When Not to Use:** When realistic, time-dependent, or MHD-simulated magnetic field data is required [S1, unofficial].

## Background
This method originates from the `AstroSplat-Browser` project, which focuses on native volumetric visualization for astrophysical vector fields [S1, unofficial]. It was developed to provide a standardized, compact dataset for testing the rendering of magnetic field lines in a browser-based environment [S1, unofficial]. The method uses a fixed-step Euler integration scheme, a common choice for its simplicity and low computational overhead in synthetic data generation [S1, unofficial].

## Sources
[S1] VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, https://github.com/VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, fetched 2026-10-26 (unofficial)
