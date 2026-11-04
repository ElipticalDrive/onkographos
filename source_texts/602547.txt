Tongzhi, if you are faced with the computational impossibility of either pre- or on-the-fly calculating a grid or navmesh for your SDF terrain pathfinding,
I have precisely the extremely ersatz solution for you. This implementation of a weighted A* algorithm swaps an explicit grid for implicit SDF evaluations at each step,
and while I can't vouch for it on any theoretical or comparative level it has been rigorously empirically tested by a trusted retinue of supernomadic mountain goats
and eNA2RATA Sentinels, and seems to work reasonably well. The rough flow is:

1. sphere trace from origin
2. step safe distance towards player
3. store this node, generate radial spokes
4. sphere trace from offset spoke positions, step safe distance, store node
5. repeat using lowest cost node according to conventional A* pathfinding

with some additional refinements and workarounds.

counterMax sets the number of steps before returning a pathfinding failure;

radialSpokes sets the number of orthogonally offset positions to evaluate at each step;

radialDist sets the offset distance;

cellSize sets the size of the pseudogrid, this must be smaller than the radialDist

heuristicWeight adjusts the weighting of the target distance, 1 results in the optimal path, higher results in less optimal paths but significantly fewer evaluations
