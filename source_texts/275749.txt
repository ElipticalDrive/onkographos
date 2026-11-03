# Ray-sphere intersection detection in five lines of LÖVR

<p float="left" align="middle">
  <img align="top" src="/screenshot-1.jpg" width="400" />
  <img align="top" src="/screenshot-2.jpg" width="400" /> 
</p>

The gist:

```lua
local translation = vec3(sphere.center):sub(ray.origin)
local distance = translation:length()

local adjacentLength = translation:dot(ray.direction)
local oppositeLength = math.sqrt(distance ^ 2 - adjacentLength ^ 2)
local areIntersecting = oppositeLength <= sphere.radius
```

Run with:

```bash
./run.sh
```
