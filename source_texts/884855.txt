# ep-premc-mincmorph

[![Version](https://img.shields.io/docker/v/fnndsc/ep-premc-mincmorph?sort=semver)](https://hub.docker.com/r/fnndsc/ep-premc-mincmorph)
[![MIT License](https://img.shields.io/github/license/fnndsc/ep-premc-mincmorph)](https://github.com/FNNDSC/ep-premc-mincmorph/blob/main/LICENSE)
[![Build](https://github.com/FNNDSC/ep-premc-mincmorph/actions/workflows/ci.yml/badge.svg)](https://github.com/FNNDSC/ep-premc-mincmorph/actions)

`ep-premc-mincmorph` smoothens masks using `mincmorph`.
Typically, this is done as a preprocessing step to create
a volume suitable for marching-cubes.

This program does the same thing as:

https://github.com/aces/surface-extraction/blob/7c9c5987a2f8f5fdeb8d3fd15f2f9b636401d9a1/scripts/marching_cubes.pl.in#L166-L184
