# lineBasedCave_Simplex
A variation of procedural cave generation based on line data for the cave paths and using Simplex noise for generating volume data.

LOGS:
16/04/2020
- First commit
- Working basic pipe volume generation. Algorithm due to change
- Added OpenSimplex Noise in Noise.compute

20/04/2020
- Auto-update mesh added
- Fixed pipe volume generation algorithm. Issue still occur at 0th line
- Added option for world-space noise input coordinate. Input coordinate will be automatically changed to line's local coordinate if disabled
- Planned further optimization for auto update
