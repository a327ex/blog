`2015-01-09 17:30:45`

This post describes an algorithm for transforming tilemaps into a level's collision geometry. Here's a high level description of it:

```lua
-- grid creation
for each tile
    if should be collsion then set as 1
    else set as 0
  
-- edge creation
for each tile 
    create 4 edges, v0 -> v1, v1 -> v2, v2 -> v3 and v3 -> v0
    where v0 is the bottom left most point 
    where v1 is the top left most point
  
-- edge deletion 1
for each edge e1
    for each edge e2
        if e1 == e2 then delete both e1 and e2
  
-- edge deletion 2
for each edge
    take p1, p2 as the edges start and end points
    find the next edge, the one that starts with p2
    take p3 as the next edges end point
    if p1, p2, p3 are collinear (on the same line)
        delete the p2 -> p3 edge
        make p2 in p1 -> p2 point to p3
  
-- group tagging
set tag number
for each edge
    walk this edge recursively until it cant go on anymore while
    tagging edges you walk through with the current tag number
    increase tag number
  
-- holes
set shapes as polygons defined by the tag numbers
for each shape s1
    for each shape s2
        if s1 is inside s2
            s1 is a hole, so
            set s1 tag number as a hole
  
-- finding zero width points
set holes as shapes that are holes
set all_points as all points from all shapes
for each shape s in holes
    find the two points, one in s and one in all_points
    that have the minimum distance between each other
  
-- make zero width channel
for each pair of points p1, p2 found from the previous step
    set mid_point as the average of p1, p2 
    check the tile value in the position of mid_point
    if the tile value is 1
        create outgoing edge e1 (p1 -> p2)
        create incoming edge e2 (p2 -> p1)
  
-- define vertices from edges 
while #edges > 0
    edge = get an edge from the edges list
    walk this edge recursively until it cant go on anymore while
    adding the points of the edges you go through to a vertices list
  
-- create shapes
using the vertices list, create your shapes
```

<br>

## Grid Creation

Create a grid that represents your tilemap in terms of collision. For every tile that needs to be considered as a solid, set it to `1`, for every tile that shouldn't, set it to `0`. Example:

```lua
grid = {
    {1, 1, 1, 1, 1}
    {1, 0, 1, 1, 1}
    {1, 0, 0, 0, 1}
    {1, 1, 1, 0, 1}
    {1, 1, 1, 1, 1}
}
```

<br>

## Edge Creation

For every tile that is `1`, create the four edges that define it. An edge can be defined as a struct of two points, where the order they appear in defines the edge's direction. For this algorithm, it's important you define the edges in a clockwise manner, starting from `v0` (bottom-left point), going to `v1` (top-left point) and so on. In the end you should have added the following edges to an edges list so that it looks like this: `v0 -> v1`, `v1 -> v2`, `v2 -> v3`, `v3 -> v0`. In other words, define edges clockwisely and add each new edge to the end of the list.

<p align="center">
<img src="https://i.imgur.com/DVChjcj.png"/>
</p>

<br>

## Edge Deletion 1

For every edge in the edges list, check to see if it appears once or more than once. If it appears more than once then delete all instances of it.

<p align="center">
<img src="https://i.imgur.com/ORw4SAC.png"/>
</p>

This image shows how the dotted edges appear more than once and how they're always inner edges instead of outer ones. For purposes of collision we only care about the outer ones, so it makes sense to delete inners. It's important to realize that because of the order in which edges are created, edge equality means that for one edge, its points are `v0 -> v1`, but for the other its points are `v1 -> v0`. If you try to go through the edges list and only look for edges that have the same start and end points (without reversing them), then this procedure will failure.

<br>

## Edge Deletion 2

This is where we get rid of redundant edges. After taking away the inner edges, we still have multiple outer edges that sometime define a single line. Ideally, these multiple edges should be merged into one when possible. The way to do this is to go through each edge, to take its points `p1` and `p2`, find the next edge (the one that starts with `p2`), take its end point `p3`, and then check if `p1`, `p2` and `p3` are collinear (if they are on the same line). If they are collinear, then delete the `p2 -> p3` edge and make the `p1 -> p2` edge point to `p3` instead of `p2`.

<p align="center">
<img src="https://i.imgur.com/K8ZjabE.png"/>
</p>

And then you repeat that until you only have the relevant edges. In this case you'd only have four edges in the end:

<p align="center">
<img src="https://i.imgur.com/GTQl3Mc.png"/>
</p>

Now, the algorithm could end here. For a lot of cases it works fine without any problems. You should have a list of edges and all you'd have to do is to go through them, grab the vertices in the right order (by choosing an edge, then finding the edge that follows from it by checking the end/start points (they're not necessarily ordered like this in the edges list)) and then just pushing those vertices into whatever data structure you use for creating level geometry. I use box2d chain shapes so I just have to push the vertices list with the vertices in an order that makes sense and it works fine.

There's one big problem, though, this algorithm doesn't work for shapes with holes in them. To the edges list, holes are just another shape, since you have the main outer shape, and then you have these other inner edges that don't connect with anyone but that define a shape anyway, since they were never deleted. If your physics library allows you to carve polygons into other polygons and get a single carved polygon out of that then that's what you should do here. If somehow you can also easily triangulate polygons then I think you can avoid the rest of the work and just create multiple triangles instead of one single polygon. Otherwise, read further.

<br>

## Group Tagging

The first thing to do is identifying how many shapes there are in this tilemap, where a shape is defined by an actually usable outer shape, or a shape inside another shape (either a hole or not). To do this, simply set a starting tag number. After this, for each edge, walk recursively to the edges that are connected with each other and not yet tagged until you can't walk anymore (usually when you reach the initial edge), all the while tagging those edges with the current tag number. After this procedure ends, increase the tag number and continue doing it until all edges are tagged.

<p align="center">
<img src="https://i.imgur.com/vEQ0Nzx.png"/>
</p>

The image above shows how four shapes are tagged. Since none of the edges in each shape connect to the others, they're all considered different by the algorithm and so they're numbered differently. This is what the grid for the shapes above looks like, in case it's confusing:

```lua
grid = {
    {1, 1, 1, 1, 1, 1, 1, 1, 1, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 1},
    {1, 0, 1, 1, 1, 1, 1, 1, 0, 1},
    {1, 0, 1, 0, 0, 0, 0, 1, 0, 1},
    {1, 0, 1, 1, 1, 1, 1, 1, 0, 1},
    {1, 0, 0, 0, 0, 0, 0, 0, 0, 1},
    {1, 1, 1, 1, 1, 1, 1, 1, 1, 1},
}
```

<br>

## Holes

Now we need to figure out which shapes are holes. For the example above we have the space between `2` and `3` being holes, and the space inside `4` being holes. So it makes sense that we should come to the conclusion that `2` and `4` are holes. However, to simplify the algorithm we'll just say that a shape is a hole if it is contained inside another shape. With this in mind, the only shape that isn't a hole in the example above would be the one tagged with `1`.

So, to do this, we first define each shape as a list of vertices. You can do this by walking through the edges of the shape until it reaches the starting one and just adding all the points to another list. Then, for each shape against each shape (double for loop), check to see if one shape is inside the other (polygon inside polygon check), if it is, then set that tag number as a hole.

<br>

## Zero Width Points

The reason we're identifying holes and not holes is because ultimately we want to make a holed polygon into an actual polygon with a hole in it. But you can't really define a polygon with a hole (at least as far as I know), so to do this we need create "zero-width channels" (as explained in Real Time Collision Detection, 12.5.1.1 Triangulating Polygons with Holes, page 499):

> To be able to triangulate polygons with holes, a reasonable representation must first be established. A common representation is as a counterclockwise outer polygon boundary plus one or more clockwise inner hole boundaries. The drawing on the left in Figure 12.19 illustrates a polygon with one hole.

> To be able to triangulate the polygon, the outer and inner boundaries must be merged into a single continuous chain of edges. This can be done by cutting a zero-width "channel" from the outer boundary to each inner boundary. Cutting this channel consists of breaking both the inner and outer boundary loops at a vertex each and then connecting the two vertices with two coincident edges in opposite directions such that the outer boundary and the hole boundary form a single counterclockwise continuous polygon boundary. Figure 12.19 shows how a hole is effectively resolved by this procedure. Multiple holes are handled by repeating the cutting procedure for each hole.

Where you just change counterclockwise for clockwise and where Figure 12.19 is the one below (edge orientation already changed):

<p align="center">
<img src="https://i.imgur.com/6AsLC3L.png"/>
</p>

Now, to find the points that we'll use for the zero width channel we do the following: hold all shapes that are holes inside one array, hold all points from all shapes (holes or not) in another. Then for each shape in the holes array we find the minimum distance between one of its points with all the other points (except its own points). So now for each hole shape, you have that hole's zero-width point and another point in an outer shape, which is the destination of the zero-width channel.

<br>

## Zero Width Channel

To create the channel, for each hole shape we first must check if the point between the two zero-width points we found in the previous step is occupied as a tile or not. This is because there is no way of knowing which hole is solid or isn't. So, in this situation:

<p align="center">
<img src="https://i.imgur.com/vEQ0Nzx.png"/>
</p>

Something like this would happen:

<p align="center">
<img src="https://i.imgur.com/HKIsd52.png"/>
</p>

The channels between `1, 2` and `3, 4` are okay because they're supposed to exist, but the channel between `2, 3` shouldn't. Since the algorithm doesn't know that what's between `2` and `3` is not supposed to be solid, we need to do a midpoint tile value check (to see if the tile is solid or not). After doing this check, add the new edges to the edges list.

<br>

## END

After this it's pretty much over. All that's left to do is to define all vertices from the edges list. This list is not ordered in any way, so you have to walk the edges (like before) from one to the next. If your tilemap has multiple isolated shapes then you need to make sure you do this walk multiple times to cover all those shapes, but between each shape, a single walk should be fine, since even if it has holes it's all connected because of the zero-width channels. After you have all vertices you can just go through those and create your level geometry.
