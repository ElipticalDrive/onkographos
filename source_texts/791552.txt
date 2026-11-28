================================================================================
CPSC 314 Assignment 3 - README
================================================================================

Name:           Lucas Lyu
Student Number: [30349559]
CWL Username:   [clyu05]


--------------------------------------------------------------------------------
HOW TO RUN
--------------------------------------------------------------------------------
- Serve the project from a local web server (do not open the HTML file directly).
  Example: python -m http.server 8000, or use VS Code Live Server.
- Part 1 (core): Open part1/A3.html in the browser.
- Part 2 (feature extension): Open part2/A3.html in the browser.


--------------------------------------------------------------------------------
KEYBOARD ACTIONS & USAGE
--------------------------------------------------------------------------------

Part 1 & Part 2 (shared):
  1              - Blinn-Phong Snowman
  2              - Ray Marching (jack-o-lantern)
  3              - Helmet Albedo map
  4              - Helmet MetalRoughness map
  5              - Helmet Emissive map
  6              - Helmet Normal map
  7              - Helmet AO map
  8              - Helmet PBR (full)
  W / S / A / D  - Move the light/sphere on the xz plane
  Q / E          - Move the light up / down
  Mouse          - Click and drag to orbit camera (when applicable)

  Part 2 only, when in Scene 2 (Ray Marching): four buttons appear in the top-right:
  - Sun rotate   Toggle sunset light rotating around the pumpkin (on/off).
  - Candle rotate Toggle candle light moving in a small circle (on/off).
  - View sector   Toggle camera sector path (on/off).
  - Reset         Restart the scene-2 animation (fly-in and orbit time reset).


--------------------------------------------------------------------------------
DIRECTORY STRUCTURE
--------------------------------------------------------------------------------
  part1/   - All parts before the feature extension (Blinn-Phong, Ray Marching,
             PBR helmet textures and materials).
  part2/   - Feature extension (Part d): All helmet material views with custom
             recoloring; and in Ray Marching (Scene 2), pumpkin assembly animation
             (parts fly in and join) plus camera on a sector path (zoom in then
             out) around the pumpkin.


--------------------------------------------------------------------------------
NOTES FOR THE MARKER
--------------------------------------------------------------------------------
- Part 2 (Part d) extends the assignment with:
  1) All material helmets recolor: The six helmet views (Albedo, MetalRoughness,
     Emissive, Normal, AO, and the full PBR material) use modified colors/tints
     so the damaged helmet has a consistent custom look across all maps.
  2) Ray Marching (Scene 2) pumpkin and camera:
     - Pumpkin assembly: When you switch to scene 2, the eyes, nose, mouth (and
       teeth) fly in from around the pumpkin and assemble into place. The
       animation starts from 0 when entering scene 2 (not from app start).
     - Camera sector path: After a short delay, the camera moves along a
       sector-shaped path: it zooms in then zooms out, orbiting in that fan
       shape to film the jack-o-lantern.
     - Scene 2 only: four buttons (Sun rotate, Candle rotate, View sector,
       Reset) appear in the top-right to toggle each effect or reset the
       animation; they are hidden when not in Scene 2.


--------------------------------------------------------------------------------
COLLABORATION & RESOURCES (fill in as required)
--------------------------------------------------------------------------------
People I discussed the assignment with:

Websites / resources I used:

================================================================================
