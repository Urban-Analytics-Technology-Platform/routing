# A collection of resources about routing

"The road not taken but still searched: practical advice from years working on routing-related software"

WARNING: These are just draft notes; this guide is not ready for use yet.

The goal of this guide is to help people working on UATP understand how routing works and various approaches for using it in their projects.

## Shortest-path algorithms

To start, it's helpful to understand how a shortest path algorithm works. The basic form is quite simple, and understanding it (and implementing it yourself) lets you tackle a variety of problems with almost the same code.

### Dijkstra's algorithm and A\*

The most basic algorithm for calculating a single path from A to B is Dijkstra's. Read through the legendary [Red Blob Games](https://www.redblobgames.com/pathfinding/a-star/introduction.html) guide to understand it. It's focused on grids, but the ideas work the same for graphs based on roads and intersections.

That guide also describes A\*, an improvement that can speed things up. The admissible heuristic depends on the edge cost chosen, discussed below. If that cost is just distance, then measuring the straight-line distance from a node to the goal works.

TODO: python pseudocode

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

The pathfinding algorithms described above are generic -- they work on abstract graphs with nodes and edges, with some numeric cost. The interesting part of routing is all about how you build graphs to represent realistic movement through a transportation system. This section is about how you do that. OpenStreetMap (OSM) is described as the primary data source, but many things would also apply to other datasets, like Ordnance Survey.

### Preliminaries: OSM isn't a graph

TODO: a wiki place explaining the data model

If you're starting from OSM, you need to understand its data model -- even if you're just using a library that takes care of it for you. OSM data about roads is not a graph. It has _nodes_ (don't confuse these with a graph's nodes), _ways_, and _relations_. Nodes consist of points with free-form string key/value tags. Ways are an ordered list of nodes, also with their own tags. And relations are lists of nodes, ways, and other relations. OSM has data about many things besides roads, but for our purposes, OSM nodes represent physical points. Ways represent roads, but critically, they usually do not represent edges in a graph. A single way could be a road that stretches over many intersections. A long way will get split in OSM when the tags along it change somewhere. There are no consistent rules about how ways are split; sometimes they're split for no reason.

So a first step is to understand how to transform raw OSM data into a graph structure:

1) Scan through OSM elements (which're always ordered by all nodes, then all ways, then all relations in valid inputs)
2) Build a mapping from OSM node ID to the point and tags
3) Collect a list of OSM ways representing roads you want in your network (you might filter based on the `highway` tag upfront if you only care about some modes)
4) Go through the list of nodes in all of your ways. Count the number of times each node is referenced. When a node only appears once in any way and it's not the first or last node in a way, then it's just there to define the shape of that way. Otherwise, we'll treat it as an intersection between two ways (roads) or an endpoint.
5) Now we'll assemble the graph. Iterate through all of the ways again.
  5a) Walk through the way's nodes in order
  5b) ... TODO maybe just pseudocode, or an illustration

TODO: diagram with both types of nodes

TODO: python psuedocode

For routing purposes, we could make every single OSM node a node in the graph. A node of degree 2 won't have an effect on the outcome of a route, but it will slightly impact performance to store more nodes and edges than necessary.

There are some edge cases to consider:

- Dead-ends: some ways start or end with a node only referenced once, but they should still wind up as nodes in a graph.
- Loop edges: start and end at the same node. They won't ever be part of a shortest path, but you should decide if you need them if you're calculating metrics for every street in an area, or for matching points to the nearest road.
- Cul-de-sacs: like loop edges with a normal road attached at one end
- Multi edges: you might have two roads going between the same pair of nodes

### Driving

This section goes into detail about building a graph for driving.

The simplest approach to edge costs is to use units of time, and simply assume edge cost is its length divided by speed limit. Noe `maxspeed` is often missing in OSM; TODO tobias' osm-legal-defaults as a backup. Of course, this is wildly optimistic. If you have a source of real edge-level congestion or delays, you can use that instead (and please share your source!).

A second question is how to treat turns through intersections. Without much difficulty, you can distinguish some cases and apply a fixed penalty that you estimate:

- Crossing a signalized junction
  - In some areas, you might know typical timings for left/right turn phases

- turning from a main road to a smaller road, without vehicular conflicts
  - Probably very cheap in car-centric areas, but maybe penalized in pedestrian-dense areas, whether or not there's a formal pedestrian crossing
- turning from 




** TODO: Edges for turns and roads?
- lanes


  - Various types of turn restrictions, and how they complicate the graph
- turn costing

  - Tolls
  - Timed turn restrictions

### Tricks: private/gated communities

Suppose you have a zone where 

  - Gated neighbourhoods, except for residents -- the big edge cost trick



(Not public transit)

- Walking
  - Two styles of sidewalk tagging in OSM
  - Disconnected graphs, depending what you include
- Cycling
  - LTS
  - Other factors; see SoTM / od2net talks

### Edge costs

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
  - You could use the final polygon, then intersect and look for points inside
  - Or output the cost to each road, then match up roads to things on them
  - But measure measure measure! Depending, it might be much slower. Preprocess and match POIs to roads, then directly output the cost to each POI instead. This is why being comfortable with the core loop of Dijkstra is important. Using a generic library might have perf impacts.

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

- reference projects/code

- example custom edge cost, prioritize closeness to some POIs
- discuss what the edge cost can be / what value can be tracked during search
  - can be a full vector. at the end of the day, needs to support sum() and min()
- testing / how to tune edge costs, if you have known counts on some edges?
- maybe just have a simple web frontend where people can write custom python for edge costs
