# A collection of resources about routing

WARNING: These are just draft notes; this guide is not ready for use yet.

The goal of this guide is to help people working on UATP understand how routing works and various approaches for using it in their projects.

## Shortest-path algorithms

To start, it's helpful to understand how a shortest path algorithm works. The basic form is quite simple, and understanding it (and implementing it yourself) lets you tackle a variety of problems with almost the same code.

### Dijkstra's algorithm and A\*

The most basic algorithm for calculating a single path from A to B is Dijkstra's. Read through the legendary [Red Blob Games](https://www.redblobgames.com/pathfinding/a-star/introduction.html) guide to understand it. It's focused on grids, but the ideas work the same for graphs based on roads and intersections.

That guide also describes A\*, an improvement that can speed things up. The admissible heuristic depends on the edge cost chosen, discussed below. If that cost is just distance, then measuring the straight-line distance from a node to the goal works.

### Contraction hierarchies

There's a technique called [contraction hierarchies](https://en.wikipedia.org/wiki/Contraction_hierarchies) (CHs) that can _dramatically_ speed up shortest-path queries. You first have to perform an expensive precalculation step on your graph. After this, you can calculate shortest paths _much_ faster. Understanding and implementing a CH is quite difficult, but you can use a library like <https://github.com/easbar/fast_paths>. You need to understand the trade-offs involved in using a CH, not how they work.

First, understand that the results of using a CH are **equivalent to Dijkstra's**. The CH "compresses" the graph, creating shortcut edges that let many queries skip searching large chunks of the graph. This process is lossless; you will still provably get the shortest path. If you're seeing differences between a CH and Dijkstra implementation in practice, one possibility is that there are multiple shortest paths for the same query.

Because preparing the CH is an expensive one-time cost, it only makes sense to use them if you're calculating many routes or need very low latency to calculate a long route. The upfront cost needs to be "amortized".

CHs only work for a "static" graph, like a road network. If you're considering routes that combine walking and public transit, they won't work.

#### Preparing the CH

In practice, you'll prepare the CH once for the area you're working in, then load and query it in a web app or in your pipeline/script that makes a bunch of queries. Even if preparing it takes a long time, the cost is amortized 

How long does it take to prepare a CH? It depends on the graph size and structure; if your network has a hierarchical structure to it, where some edges very quickly get you across large distances, then it'll be faster. So preparing a graph for driving, where the edge cost is based on speed limits, will be faster than a graph for walking, if the edge cost is just based on distance. [fast_paths benchmarks](https://github.com/easbar/fast_paths/?tab=readme-ov-file#benchmarks) illustrate this well -- using distance edge costs for the entire USA (57 million edges) takes twice as long as speed.

#### Multiple graphs or changing edge weights

What if you have multiple related graphs, like one for pedestrians and another for cyclists? Or if you're calculating two types of routes for cyclists, using edge costs that either favor fast, scary routes vs indirect, quieter routes? You will need two separate CHs. But you may be able to cheat and speed up preparing the two CHs.

CH preparation is split into two stages, and you may be able to [reuse the first one, "node ordering"](https://github.com/easbar/fast_paths/?tab=readme-ov-file#preparing-the-graph-after-changes). The graphs need to have the same nodes and edges, only with edge costs differing. To make this work for graphs for two different modes of travel, you can "pad" one graph with infinite-cost edges. For example if you're preparing a graph for both cyclists and drivers, they both can use most edges, but some will only be accessible to one mode or the other. In both graphs, still include all the same nodes and edges, but just set an infinite cost edge for one mode. As always, benchmark this for your particular use case -- if the hierarchical structure of the network from the edge weights is very different between the two, it might be slower to reuse the node ordering. It might be better to just prepare two CHs independently.

This trick also works if you're updating a graph's edge weights. For example, if you're modelling a road closure, you'll only end up changing a few edge costs. So reusing node ordering might help.

## Making the graph for routing: walking / cycling / driving

(Not public transit)

- OSM isn't a graph; how to turn it into one
  - Complications with dead-ends, cul-de-sacs, loop edges, multiple edges between the same nodes
- Driving-specific considerations
  - Various types of turn restrictions, and how they complicate the graph
  - Gated neighbourhoods, except for residents -- the big edge cost trick
- Walking
  - Two styles of sidewalk tagging in OSM
  - Disconnected graphs, depending what you include
- Cycling
  - LTS
  - Other factors; see SoTM / od2net talks

### Edge costs

- Driving
  - Tolls
  - Historic / current congestion
  - Timed turn restrictions
- Cycling
  - Opening hours
- Walking
  - Curb cuts, pavement width? <https://www.accessmap.io>
- How to balance between multiple constraints
  - Using Cartesian products in the priority queue
- Elevation

## Handling requests

- Snapping requests to the nearest node
  - Do you need to project to Mercator to use an r-tree?

## Output

Maybe start with this section.

- One A to B path
  - How much detail -- a LineString? Attributes per step?
  - List of OSM IDs and direction -- warning these IDs aren't stable through time
- Route network
  - The overline problem
  - The od2net approach
- Travel time matrix / skims
- Isochrones

### Isochrones

- "As the crow flies" vs using the network
- Notes from <https://a-b-street.github.io/docs/software/fifteen_min.html#implementation>
- Pipeline
  - Just do Dijkstra's, remembering the cost to reach each node
  - Fill out a grid with those costs, maybe smooth
    - <https://github.com/a-b-street/abstreet/pull/1075> and the issue
  - Marching squares
- Flexibility of Dijkstra's -- start from multiple sources
  - Single-source from every house vs multi-source from a few dozens POIs
- Connectivity calculations along the way

## Public transit

- Vastly more complicated
  - GTFS data
  - Ticket prices
  - Time and day matter

...


## Using existing software

- Batch routing and HTTP/JSON APIs vs native bindings
- In a web map

## Misc

perf advice: profile your code. maybe a slow step is calculating edge cost dynamically from its properties.

if you're calculating many routes and you're tempted to try caching the results, profile your approaches. calculating a single path is often very very fast. the overhead and other steps (using the results) might be the dominant factor. a cache isn't free either time or memory-wise, measure measure measure.


multi start / multi end



- TODO: Alternative route / second best route?
