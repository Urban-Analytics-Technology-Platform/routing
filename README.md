# A collection of resources about routing

WARNING: These are just draft notes; this guide is not ready for use yet.

The goal of this guide is to help people working on UATP understand how routing works and various approaches for using it in their projects.

## Shortest-path algorithms

- To understand Dijkstra's and A\*, check out the [Red Blob Games](https://www.redblobgames.com/pathfinding/a-star/introduction.html) guide. It's focused on grids, but the ideas work the same for graphs based on roads and intersections.
- Contraction hierarchies
  - The results are the same as Dijkstra's; the contraction operation is not lossy
  - When are they good to use? Many queries, amortizing the setup time
  - Adjusting edge weights vs rebuilding the CH from scratch
  - Examples of using <https://github.com/easbar/fast_paths/>
    - Node maps
    - The thread-local path calculator
- Alternative route / second best route?

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

## Input

- Out of scope

## Output

- One A to B path
  - How much detail -- a LineString? Attributes per step?
  - List of OSM IDs and direction -- warning these IDs aren't stable through time
- Route network
  - The overline problem
  - The od2net approach
- Travel time matrix / skims
- Isochrones

## Isochrones

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
