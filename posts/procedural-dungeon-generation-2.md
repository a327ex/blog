`2015-08-30 22:29`

This post explains a technique for generating randomized dungeons that was first described by [TinyKeepDev](http://store.steampowered.com/app/278620/) [here](https://www.reddit.com/r/gamedev/comments/1dlwc4/procedural_dungeon_generation_algorithm_explained/). I'll go over it in a little more detail than the steps in the original post. The general way the algorithm works is this:

<p align="center">
<img src="https://i.imgur.com/RJAVmBj.gif"/>
</p>

<br>

## Generate Rooms

First you wanna generate some rooms with some width and height that are placed randomly inside a circle. TKdev's algorithm used the normal distribution for generating room sizes and I think that this is generally a good idea as it gives you more parameters to play with. Picking different ratios between width/height mean and standard deviation will generally result in different looking dungeons.

One function you might need to do this is `getRandomPointInCircle`:

```lua
function getRandomPointInCircle(radius)
  local t = 2*math.pi*math.random()
  local u = math.random()+math.random()
  local r = nil
  if u > 1 then r = 2-u else r = u end
  return radius*r*math.cos(t), radius*r*math.sin(t)
end
```

You can get more info on how that works exactly [here](http://stackoverflow.com/questions/5837572/generate-a-random-point-within-a-circle-uniformly). And after that you should be able to do something like this:

<p align="center">
<img src="https://i.imgur.com/JLdt85Z.gif"/>
</p>

One very important thing that you have to consider is that since you're (at least conceptually) dealing with a grid of tiles you have to snap everything to that same grid. In the gif above the tile size is 4 pixels, meaning that all room positions and sizes are multiples of 4. To do this I wrap position and width/height assignments in a function that rounds the number to the tile size:

```lua
function roundm(n, m)
  return math.floor(((n + m - 1)/m))*m
end

-- Now we can change the returned value from getRandomPointInCircle to:
function getRandomPointInCircle(radius)
  ...
  return roundm(radius*r*math.cos(t), tile_size), roundm(radius*r*math.sin(t), tile_size)
end
```

<br>

## Separate Rooms

Now we can move on to the separation part. There's a lot of rooms mashed together in one place and they should not be overlapping somehow. TKdev used the separation steering behavior to do this but I found that it's much easier to just use a physics engine. After you've added all rooms, simply add solid physics bodies to match each room's position and then just run the simulation until all bodies go back to sleep. In the gif I'm running the simulation normally but when you're doing this between levels you can advance the physics simulation faster.

<p align="center">
<img src="https://i.imgur.com/25UsRgG.gif"/>
</p>

The physics bodies themselves are not tied to the tile grid in any way, but when setting the room's position you wrap it with the `roundm` call and then you get rooms that are not overlapping with each other and that also respect the tile grid. The gif below shows this in action as the blue outlines are the physics bodies and there's always a slight mismatch between them and the rooms since their position is always being rounded:

<p align="center">
<img src="https://i.imgur.com/WddMHg9.gif"/>
</p>

One issue that might come up is when you want to have rooms that are skewed horizontally or vertically. For instance, consider the game I'm working on:

<p align="center">
<img src="https://user-images.githubusercontent.com/409773/41509734-d4ed413a-722e-11e8-904b-e80b52c973c1.gif"/>
</p>

Combat is very horizontally oriented and so I probably want to have most rooms being bigger in width than they are in height. The problem with this lies in how the physics engine decides to resolve its collisions whenever long rooms are near each other:

<p align="center">
<img src="https://i.imgur.com/JsxowrA.gif"/>
</p>

As you can see, the dungeon becomes very tall, which is not ideal. To fix this we can spawn rooms initially inside a thin strip instead of on a circle. This ensures that the dungeon itself will have a decent width to height ratio:

<p align="center">
<img src="https://i.imgur.com/9DeqvFz.gif"/>
</p>

To spawn randomly inside this strip we can just change the `getRandomPointInCircle` function to spawn points inside an ellipse instead (in the gif above I used `ellipse_width = 400` and `ellipse_height = 20`):

```lua
function getRandomPointInEllipse(ellipse_width, ellipse_height)
  local t = 2*math.pi*math.random()
  local u = math.random()+math.random()
  local r = nil
  if u > 1 then r = 2-u else r = u end
  return roundm(ellipse_width*r*math.cos(t)/2, tile_size), 
         roundm(ellipse_height*r*math.sin(t)/2, tile_size)
end
```

<br>

## Main Rooms

The next step simply determines which rooms are main/hub rooms and which ones aren't. TKdev's approach here is pretty solid: just pick rooms that are above some width/height threshold. For the gif below the threshold I used was `1.25*mean`, meaning, if `width_mean` and `height_mean` are 24 then rooms with width and height bigger than 30 will be selected.

<p align="center">
<img src="https://i.imgur.com/EzhzxAQ.gif"/>
</p>

<br>

## Delaunay Triangulation + Graph

Now we take all the midpoints of the selected rooms and feed that into the Delaunay procedure. You either implement this procedure yourself or find someone else who's done it and shared the source. In my case I got lucky and [Yonaba already implemented it](https://github.com/Yonaba/delaunay). As you can see from that interface, it takes in points and spits out triangles:

<p align="center">
<img src="https://i.imgur.com/qKsmD25.gif"/>
</p>

After you have the triangles you can then generate a graph. This procedure should be fairly simple provided you have a graph data structure/library at hand. In case you weren't doing this already, it's useful that your Room objects/structures have unique ids to them so that you can add those ids to the graph instead of having to copy them around.

<br>

## Minimum Spanning Tree

After this we generate a minimum spanning tree from the graph. Again, either implement this yourself or find someone who's done it in your language of choice.

<p align="center">
<img src="https://i.imgur.com/ZxVDhL0.gif"/>
</p>

The minimum spanning tree will ensure that all main rooms in the dungeon are reachable but also will make it so that they're not all connected as before. This is useful because by default we usually don't want a super connected dungeon but we also don't want unreachable islands. However, we also usually don't want a dungeon that only has one linear path, so what we do now is add a few edges back from the Delaunay graph:

<p align="center">
<img src="https://i.imgur.com/y2IClYR.gif"/>
</p>

This will add a few more paths and loops and will make the dungeon more interesting. TKdev arrived at 15% of edges being added back and I found that around 8-10% was a better value. This may vary depending on how connected you want the dungeon to be in the end.

<br>

## Hallways

For the final part we want to add hallways to the dungeon. To do this we go through each node in the graph and then for each other node that connects to it we create lines between them. If the nodes are horizontally close enough (their `y` position is similar) then we create a horizontal line. If the nodes are vertically close enough then we create a vertical line. If the nodes are not close together either horizontally or vertically then we create 2 lines forming an L shape. 

The test that I used for what ~close enough~ means calculates the midpoint between both nodes' positions and checks to see if that midpoint's `x or y` attributes are inside the node's boundaries. If they are then I create the line from that midpoint's position. If they aren't then I create two lines, both going from the source's midpoint to the target's midpoint but only in one axis.

<p align="center">
<img src="https://i.imgur.com/Rcq6Trp.png"/>
</p>

In the picture above you can see examples of all cases. Nodes 62 and 47 have a horizontal line between them, nodes 60 and 125 have a vertical one and nodes 118 and 119 have an L shape. It's also important to note that those aren't the only lines I'm creating. They are the only ones I'm drawing but I'm also creating 2 additional lines to the side of each one spaced by `tile_size`, since I want my corridors to be at least 3 tiles wide in either width or height.

Anyway, after this we check to see which rooms that aren't main/hub rooms that collide with each of the lines. Colliding rooms are then added to whatever structure you're using to hold all this by now and they will serve as the skeleton for the hallways:

<p align="center">
<img src="https://i.imgur.com/TyIIU8r.png"/>
</p>

Depending on the uniformity and size of the rooms that you initially set you'll get different looking dungeons here. If you want your hallways to be more uniform and less weird looking then you should aim for low standard deviation and you should put some checks in place to keep rooms from being too skinny one way or the other.

For the last step we simply add 1 tile sized grid cells to make up the missing parts. Note that you don't actually need a grid data structure or anything too fancy, you can just go through each line according to the tile size and add grid rounded positions (that will correspond to a 1 tile sized cell) to some list. This is where having 3 (or more) lines happening instead of only 1 matters.

<p align="center">
<img src="https://i.imgur.com/W6Bvuhp.png"/>
</p>


<p align="center">
<img src="https://i.imgur.com/zSh9GUe.gif"/>
</p>

And then after this we're done! :cyclone: 

<br>

## END

The data structures I returned from this entire procedure were: a list of rooms (each room is just a structure with a unique id, x/y positions and width/height); the graph, where each node points to a room id and the edges have the distance between rooms in tiles; and then an actual 2D grid, where each cell can be nothing (meaning its empty), can point to a main/hub room, can point to a hallway room or can be hallway cell. With these 3 structures I think it's possible to get any type of data you could want out of the layout and then you can figure out where to place doors, enemies, items, which rooms should have bosses, and so on.
