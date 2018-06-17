`2013-06-30 23:43`

This post briefly explains a technique for generating randomized dungeons that is loosely based on the one explained here:

<p align="center">
<a href="https://www.youtube.com/watch?v=GcM9Ynfzll0#t=05m20s"><img src="https://i.imgur.com/kFphDBo.jpg"></a>
</p>

<br>

## Grid Generation

Generate a grid:

<p align="center">
<img src="https://i.imgur.com/X2kuluX.png"/>
</p>

<br>

## Grid Difficulty Coloring 

Color the grid with `x` red rooms (hard), `y` blue rooms (medium) and `z` green rooms (easy) such that `x+y+z = n` rooms in the grid. `x`, `y` and `z` will be used as controls for difficulty.

1. First color the grid with `x` red rooms such that no red room has another red neighbor;
2. Then for each red room color one neighbor blue and one neighbor green;
3. Color the rest of the rooms randomly with the remaining number of rooms for each color.

<p align="center">
<img src="https://i.imgur.com/pjuoq3z.png"/>
</p>

<br>

## Dungeon Path Creation

Choose two nodes that are far apart enough and then find a path between them while mostly avoiding red rooms. If you choose a proper `x`, since red rooms can't be neighbors to themselves and the pathfinding algorithm doesn't go for diagonals, it should create a not so direct path from one node to the other.

<p align="center">
<img src="https://i.imgur.com/vDz7DPA.png"/>
</p>

For all nodes in the path, add their red, blue or green neighbors. This should add the possibility of side paths and overall complexity/difficulty in the dungeon.

<p align="center">
<img src="https://i.imgur.com/UmL8H30.png"/>
</p>

<br>

## Room Creation and Connection

Join smaller grids into bigger ones according to predefined room sizes. Use a higher chances for smaller widths/heights and lower chances for bigger widths/heights.

<p align="center">
<img src="https://i.imgur.com/AgqEvzu.png"/>
</p>

Generate all possible connections between rooms.

<p align="center">
<img src="https://i.imgur.com/nwFbl01.png"/>
</p>

Randomly remove connections until a certain number of connections per room is met. If `n = total rooms`, then set `n*a` rooms with `>=4` connections, `n*b` with 3, `n*c` with 2 and `n*d` with 1, such that `a+b+c+d=1`. Controlling `a`, `b`, `c` and `d` lets you control how mazy the dungeon gets. If `c` or `d` are considerably higher than `a` or `b` then there won't be many hub rooms that connect multiple different paths, so it's gonna have lots of different thin paths with dead ends. If `a` or `b` are higher then the dungeon will be super connected and therefore easier.

<p align="center">
<img src="https://i.imgur.com/FzAjMVe.png"/>
</p>

Reconnect isolated islands. Since the last step is completely random, there's a big chance that rooms or groups of rooms will become unreachable. 

<p align="center">
<img src="https://i.imgur.com/akHr0SR.png"/>
</p>

To fix that:

1. Flood fill to figure out how many isolated groups exist;
2. Pick a group at random and go through its rooms. For each room check if it neighbors a room belonging to another group, if it does then connect them and end the search;
3. Repeat the previous steps until there's only one group left (everyone's connected).

<p align="center">
<img src="https://i.imgur.com/GZkKeim.png"/>
</p>

<br>

## END

And that's it, I guess. There's more stuff like adding special rooms, but that's specific to each game, so whatever. But basically I have a few parameters I can control when creating a new dungeon: dungeon width/height; percentage of hard, medium and easy rooms; what color neighbors get added to the original path and the percentage of rooms with >=4, 3, 2 or 1 connections. I think that's enough to have control of how easy/hard the dungeon will be...
