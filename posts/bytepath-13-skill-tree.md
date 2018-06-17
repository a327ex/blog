## Introduction

In this article we'll focus on the creation of the skill tree. [This is what the skill tree looks like right now](https://streamable.com/9bj59). We'll not place each node hand by hand or anything like that (that will be left as an exercise), but we will go over everything needed to make the skill tree happen and work as one would expect.

First we'll focus on how each node will be defined, then on how we can read those definitions, create the necessary objects and apply the appropriate passives to the player. Then we'll move on to the main objects (Nodes and Links), and after that we'll go over saving and loading the tree. And finally the last thing we'll do is implement the functionality needed so that the player can spend his skill points on it.

<br>

## Skill Tree

There are many different ways we can go about defining a skill tree, each with their advantages and disadvantages. There are roughly three options we can go for:

*   Create a skill tree editor to place, link and define the stats for each node visually;
*   Create a skill tree editor to place and link nodes visually, but define stats for each node in a text file;
*   Define everything in a text file.

I'm someone who likes to keep the implementation of things simple and who has no problem with doing lots of manual and boring work, which means that I'll solve problems in this way generally. When it comes to the options above it means I'll pick the third one.

The first two options require us to build a visual skill tree editor. To understand what this entails exactly we should try to list the high level features that a visual skill tree editor would have:

*   Placing new nodes
*   Linking nodes together
*   Deleting nodes
*   Moving nodes
*   Text input for defining each node's stats

These are pretty much the only high level features I can think of initially, and they imply a few more things:

*   Nodes will probably have to be aligned in relation to each other in some way, which means we'll need some sort of alignment system in place. Maybe nodes can only be placed according to some sort of grid system.
*   Linking, deleting and moving nodes around implies that we need an ability to select certain nodes to which we want to apply each of those actions. This means node selection is another feature we'd have to implement.
*   If we go for the option where we also define stats visually, then text input is necessary. There are many ways we can get a proper TextInput element working in LÖVE for little work (https://github.com/keharriso/love-nuklear), so we just need to add the logic for when a text input element appears, and how we read information from it once its been written to.

As you can see, adding a skill tree editor doesn't seem like a lot of work compared to what we've done so far. So if you want to go for that option it's totally viable and may make the process of building the skill tree better for you. But like I said, I generally have no problem with doing lots of manual and boring work, which means that I have no problem with defining everything in a text file. So for this article we will not do any of those skill tree editor things and we will define the entirety of the skill tree in a text file.

<br>

## Tree Definition

So to get started with the tree's definition we need to think about what kinds of things make up a node:

*   Passive's text:
    *   Name
    *   Stats it changes (6% Increased HP, +10 Max Ammo, etc)
*   Position
*   Linked nodes
*   Type of node (normal, medium or big)

So, for instance, the "4% Increased HP" node shown in the gif below:

<p align="center">
<img src="https://user-images.githubusercontent.com/409773/41511105-eb6d8aec-7246-11e8-9153-fa14e5f79e98.gif">
</p>

Could have a definition like this:

```lua
tree[10] = {
    name = 'HP', 
    stats = {
        {'4% Increased HP', 'hp_multiplier' = 0.04}
    }
    x = 150, y = 150,
    links = {4, 6, 8},
    type = 'Small',
}
```

We're assuming that `(150, 150)` is a reasonable position, and that the position on the `tree` table of the nodes linked to it are 4, 6 and 8 (its own position is 10, since its being defined in `tree[10]`). In this way, we can easily define all the hundreds of nodes in the tree, pass this huge table to some function which will read all this, create Node objects and link those accordingly, and then we can apply whatever logic we want to the tree from there.

<br>

## Nodes and Camera

Now that we have an idea of what the tree file will look like we can start building from it. The first thing we have to do is create a new `SkillTree` room and then use `gotoRoom` to go to it at the start of the game (since that's where we'll be working for now). The basics of this room should be exactly the same as the Stage room, so I'll assume you're capable of doing that with no guidance.

We'll define two nodes in the `tree.lua` file but we'll do it only by their position for now. Our goal will be to read those nodes from that file and create them in the SkillTree room. We could define them like this:

```lua
tree = {}
tree[1] = {x = 0, y = 0}
tree[2] = {x = 32, y = 0}
```

And we could read them like this:

```lua
function SkillTree:new()
    ...

    self.nodes = {}
    for _, node in ipairs(tree) do table.insert(self.nodes, Node(node.x, node.y)) end
end
```

Here we assume that all objects for our SkillTree will not be inside an Area, which means we don't have to use `addGameObject` to add a new game object to the environment, and it also means we need to keep track of existing objects ourselves. In this case we're doing that in the `nodes` table. The `Node` object could look like this:

```lua
Node = Object:extend()

function Node:new(x, y)
    self.x, self.y = x, y
end

function Node:update(dt)
    
end

function Node:draw()
    love.graphics.setColor(default_color)
    love.graphics.circle('line', self.x, self.y, 12)
end
```

So it's a simple object that doesn't extend from GameObject at all. And for now we'll just draw it at its position as a circle. If we go through the `nodes` list and call update/draw on each node we have in it, assuming we're locking the camera at position `0, 0` (unlike in Stage where we locked it at `gw/2, gh/2`) then it should look like this:

<p align="center">
<img src="https://vgy.me/ch8EEm.png">
</p>

And as expected, both the nodes we defined in the tree file are shown here.

<br>

### Camera

To make the skill tree work properly we have to change the way the camera works a bit. Right now we should have the same behavior we have from the Stage room, which means that the camera is simply locked to a position but doesn't do anything interesting. But on the SkillTree we want the camera to be able to be moved around with the mouse and for the player to be able to zoom out (and also back in) so he can see more of the tree at once.

To move it around, we want to make it so that whenever the player is holding down the left mouse button and dragging the screen around, it moves in the opposite direction. So if the player is holding the button and moves the mouse up, then we want to move the camera down. The basic way to achieve this is to keep track of the mouse's position on the previous frame as well as on this frame, and then move the camera in the opposite direction of the `current_frame_position - previous_frame_position` vector. All that looks like this:

```lua
function SkillTree:update(dt)
    ...
  
    if input:down('left_click') then
        local mx, my = camera:getMousePosition(sx, sy, 0, 0, sx*gw, sy*gh)
        local dx, dy = mx - self.previous_mx, my - self.previous_my
        camera:move(-dx, -dy)
    end
    self.previous_mx, self.previous_my = camera:getMousePosition(sx, sy, 0, 0, sx*gw, sy*gh)
end
```

And if you try this out it should behave as expected. Note that the `camera:getMousePosition` has been slightly changed from [the default](http://hump.readthedocs.io/en/latest/camera.html#camera:mousePosition) because of the way we're handling our canvases, which is different than what the library expected. I changed this a long long time ago so I don't remember why it is like this exactly, so I'll just go with it. But if you're curious you should look into this more clearly and examine if it needs to be this way, or if there's a way to use the default camera module without any changes that I just didn't figure it out properly.

As for the zooming in/out, we can simply change the camera's `scale` properly whenever the user presses wheel up/down:

```lua
function SKillTree:update(dt)
    ...
  	
    if input:pressed('zoom_in') then 
        self.timer:tween('zoom', 0.2, camera, {scale = camera.scale + 0.4}, 'in-out-cubic') 
    end
    if input:pressed('zoom_out') then 
        self.timer:tween('zoom', 0.2, camera, {scale = camera.scale - 0.4}, 'in-out-cubic') 
    end
end
```

We're using a timer here so that the zooms are a bit gentle and look better. We're also sharing both timers under the same `'zoom'` id, since we want the other tween to stop whenever we start another one. The only thing left to do in this piece of code is to add limits to how low or high the scale can go, since we don't want it to go below 0, for instance.

<br>

## Links and Stats

With the previous code we should be able to add nodes and move around the tree. Now we'll focus on linking nodes together and displaying their stats. 

To link nodes together we'll create a `Line` object, and this Line object will receive in its constructors the `id` of two nodes that it's linking together. The `id` represents the index of a certain node on the `tree` object. So the node created from `tree[2]` will have `id = 2`. We can change the Node object like this:

```lua
function Node:new(id, x, y)
    self.id = id
    self.x, self.y = x, y
end
```

And we can create the Line object like this:

```lua
Line = Object:extend()

function Line:new(node_1_id, node_2_id)
    self.node_1_id, self.node_2_id = node_1_id, node_2_id
    self.node_1, self.node_2 = tree[node_1_id], tree[node_2_id]
end

function Line:update(dt)
    
end

function Line:draw()
    love.graphics.setColor(default_color)
    love.graphics.line(self.node_1.x, self.node_1.y, self.node_2.x, self.node_2.y)
end
```

Here we use our passed in ids to get the relevant nodes and store then in `node_1` and `node_2`. Then we simply draw a line between the position of those nodes.

Back in the SkillTree room, we need to now create our Line objects based on the `links` table of each node in the tree. Suppose we now have a tree that looks like this:

```lua
tree = {}
tree[1] = {x = 0, y = 0, links = {2}}
tree[2] = {x = 32, y = 0, links = {1, 3}}
tree[3] = {x = 32, y = 32, links = {2}}
```

We want node 1 to be linked to node 2, node 2 to be linked to 1 and 3, and node 3 to be linked to node 2. Implementation wise we want to over each node and over each of its links and then create Line objects based on those links.

```lua
function SkillTree:new()
    ...
  
    self.nodes = {}
    self.lines = {}
    for id, node in ipairs(tree) do table.insert(self.nodes, Node(id, node.x, node.y)) end
    for id, node in ipairs(tree) do 
        for _, linked_node_id in ipairs(node.links) do
            table.insert(self.lines, Line(id, linked_node_id))
        end
    end
end
```

One last thing we can do is draw the nodes using the `'fill'` mode, otherwise our lines will go over them and it will look off:

```lua
function Node:draw()
    love.graphics.setColor(background_color)
    love.graphics.circle('fill', self.x, self.y, self.r)
    love.graphics.setColor(default_color)
    love.graphics.circle('line', self.x, self.y, self.r)
end
```

And after doing all that it should look like this:

<p align="center">
<img src="https://vgy.me/YLZvKs.png">
</p>

---

As for the stats, supposing we have a tree like this:

```lua
tree[1] = {
    x = 0, y = 0, stats = {
    '4% Increased HP', 'hp_multiplier', 0.04, 
    '4% Increased Ammo', 'ammo_multiplier', 0.04
    }, links = {2}
}
tree[2] = {x = 32, y = 0, stats = {'6% Increased HP', 'hp_multiplier', 0.04}, links = {1, 3}}
tree[3] = {x = 32, y = 32, stats = {'4% Increased HP', 'hp_multiplier', 0.04}, links = {2}}
```

We want to achieve this:

<p align="center">
<img src="https://vgy.me/sK4Dxx.gif">
</p>

No matter how zoomed in or zoomed out, whenever the user mouses over a node we want to display its stats in a small rectangle.

The first thing we can focus on is figuring out if the player is hovering over a node or not. The simplest way to do this is to just check is the mouse's position is inside the rectangle that defines each node:

```lua
function Node:update(dt)
    local mx, my = camera:getMousePosition(sx*camera.scale, sy*camera.scale, 0, 0, sx*gw, sy*gh)
    if mx >= self.x - self.w/2 and mx <= self.x + self.w/2 and 
       my >= self.y - self.h/2 and my <= self.y + self.h/2 then 
      	self.hot = true
    else self.hot = false end
end
```

We have a width and height defined for each node and then we check if the mouse position `mx, my` is inside the rectangle defined by this width and height. If it is, then we set `hot` to true, otherwise it will be set to false. `hot` then is just a boolean that tells us if the node is being hovered over or not.

Now for drawing the rectangle. We want to draw the rectangle above everything else on the screen, so doing this inside the Node class doesn't work, since each node is drawn sequentially, which means that our rectangle would end up behind one or another node sometimes. So we'll do it directly in the SkillTree room. And perhaps even more importantly, we'll do it outside the `camera:attach` and `camera:detach` block, since we want the size of this rectangle to remain the same no matter how zoomed in or out we are.

The basics of it looks like this:

```lua
function SkillTree:draw()
    love.graphics.setCanvas(self.main_canvas)
    love.graphics.clear()
        ...
        camera:detach()

        -- Stats rectangle
        local font = fonts.m5x7_16
        love.graphics.setFont(font)
        for _, node in ipairs(self.nodes) do
            if node.hot then
                -- Draw rectangle and stats here
            end
        end
        love.graphics.setColor(default_color)
    love.graphics.setCanvas()
    ...
end
```

Before drawing the rectangle we need to figure out its width and height. The width is based on the size of the longest stat, since the rectangle has to be bigger than it by definition. To do that we can try something like this:

```lua
function SkillTree:draw()
    ...
        for _, node in ipairs(self.nodes) do
            if node.hot then
                local stats = tree[node.id].stats
                -- Figure out max_text_width to be able to set the proper rectangle width
                local max_text_width = 0
                for i = 1, #stats, 3 do
                    if font:getWidth(stats[i]) > max_text_width then
                        max_text_width = font:getWidth(stats[i])
                    end
                end
            end
        end
    ...
end
```

The `stats` variable will hold the list of stats for the current node. So if we're going through the node `tree[2]`, `stats` would be `{'4% Increased HP', 'hp_multiplier', 0.04, '4% Increased Ammo', 'ammo_multiplier', 0.04}`. The stats table is divided in 3 elements always. First there's the visual description of the stat, then what variable it will change on the Player object, and then the amount of that effect. We want the visual description only, which means that we should go over this table in increments of 3, which is what we're doing in the for loop above.

Once we do that we want to find the width of that string given the font we're using, and for that we'll use [`font:getWidth`](https://love2d.org/wiki/Font:getWidth). The maximum width of all our stats will be stored in the `max_text_width` variable and then we can start drawing our rectangle from there:

```lua
function SkillTree:draw()
    ...
        for _, node in ipairs(self.nodes) do
            if node.hot then
                ...
                -- Draw rectangle
                local mx, my = love.mouse.getPosition() 
                mx, my = mx/sx, my/sy
                love.graphics.setColor(0, 0, 0, 222)
                love.graphics.rectangle('fill', mx, my, 16 + max_text_width, 
        		font:getHeight() + (#stats/3)*font:getHeight())  
            end
        end
    ...
end
```

We want to draw the rectangle at the mouse position, except that now we don't have to use `camera:getMousePosition` because we're not drawing with the camera transformations. However, we can't simply use `love.mouse.getPosition` directly either because our canvas is being scaled by `sx, sy`, which means that the mouse position as returned by LÖVE's function isn't correct once we change the game's scale from 1. So we have to divide that position by the scale to get the appropriate value.

After we have the proper position we can draw the rectangle with width `16 + max_text_width`, which gives us about 8 pixels on each side as a border, and then with height `font:getHeight() + (#stats/3)*font:getHeight()`. The first element of this calculation (`font:getHeight()`) serves the same purpose as 16 in the width calculation, which is to be just some value for a border. In this case the rectangle will have `font:getHeight()/2` as a top and bottom border. The second part of is simply the amount of height each stat line takes. Since stats are grouped in threes, it makes sense to count each stat as `#stats/3` and then multiply that by the line height.

Finally, the last thing to do is to draw the text. We know that the x position of all texts will be `8 + mx`, because we decided we wanted 8 pixels of border on each side. And we also know that the y position of the first text will be `my + font:getHeight()/2`, because we decided we want `font:getHeight()/2` as border on top and bottom. The only thing left to figure out is how to draw multiple lines, but we also already know this since we decided that the height of the rectangle would be `(#stats/3)*font:getHeight()`. This means that each line is drawn `1*font:getHeight()`, `2*font:getHeight()`, and so on. All that looks like this:

```lua
function SkillTree:draw()
    ...
        for _, node in ipairs(self.nodes) do
            if node.hot then
                ...
                -- Draw text
                love.graphics.setColor(default_color)
                for i = 1, #stats, 3 do
                    love.graphics.print(stats[i], math.floor(mx + 8), 
          			math.floor(my + font:getHeight()/2 + math.floor(i/3)*font:getHeight()))
                end
            end
        end
    ...
end
```

And this should get us the result we want. As a small note on this, if you look at this code as a whole it looks like this:

```lua
function SkillTree:draw()
    love.graphics.setCanvas(self.main_canvas)
    love.graphics.clear()
        ...
  
        -- Stats rectangle
        local font = fonts.m5x7_16
        love.graphics.setFont(font)
        for _, node in ipairs(self.nodes) do
            if node.hot then
                local stats = tree[node.id].stats
                -- Figure out max_text_width to be able to set the proper rectangle width
                local max_text_width = 0
                for i = 1, #stats, 3 do
                    if font:getWidth(stats[i]) > max_text_width then
                        max_text_width = font:getWidth(stats[i])
                    end
                end
                -- Draw rectangle
                local mx, my = love.mouse.getPosition() 
                mx, my = mx/sx, my/sy
                love.graphics.setColor(0, 0, 0, 222)
                love.graphics.rectangle('fill', mx, my, 
        		16 + max_text_width, font:getHeight() + (#stats/3)*font:getHeight())
                -- Draw text
                love.graphics.setColor(default_color)
                for i = 1, #stats, 3 do
                    love.graphics.print(stats[i], math.floor(mx + 8), 
          			math.floor(my + font:getHeight()/2 + math.floor(i/3)*font:getHeight()))
                end
            end
        end
        love.graphics.setColor(default_color)
    love.graphics.setCanvas()
  
    ...
end
```

And I know that if I looked at code like this a few years ago I'd be really bothered by it. It looks ugly and unorganized and perhaps confusing, but in my experience this is the stereotypical game development drawing code. Lots of small and seemingly random numbers everywhere, pixel adjustments, lots of different concerns instead of the whole thing feeling cohesive, and so on. I'm very used to this type of code by now so it doesn't bother me anymore, and I'd advise you to get used to it too because trying to make it "cleaner", in my experience, only leads to things that are even more confusing and less intuitive to work with.

<br>

## Gameplay

Now that we can place nodes and link them together we have to code in the logic behind buying nodes. The tree will have one or multiple "entry points" from which the player can start buying nodes, and then from there he can only buy nodes that adjacent to one he already bought. For instance, in the way I set my own tree up, there's a central starting node that provides no bonuses and then from it 4 additional ones connect out to start the tree:

<p align="center">
<img src="https://vgy.me/BcKeAi.png">
</p>

Suppose now that we have a tree that looks like this initially:

```lua
tree = {}
tree[1] = {x = 0, y = 0, links = {2}}
tree[2] = {x = 48, y = 0, stats = {'4% Increased HP', 'hp_multiplier', 0.04}, links = {3}}
tree[3] = {x = 96, y = 0, stats = {'6% Increased HP', 'hp_multiplier', 0.06}, links = {4}}
tree[4] = {x = 144, y = 0, stats = {'4% Increased HP', 'hp_multiplier', 0.04}}
```

<p align="center">
<img src="https://vgy.me/tzvg1u.png">
</p>

The first thing we wanna do is make it so that node #1 is already activated while the others are not. What I mean by a node being activated is that it has been bought by the player and so its effects will be applied in gameplay. Since node #1 has no effects, in this way we can create an "initial node" from where the tree will expand.

The way we'll do this is through a global table called `bought_node_indexes`, which will just contain a bunch of numbers pointing to which nodes of the tree have already been bought. In this case we can just add `1` to it, which means that `tree[1]` will be active. We also need to change the nodes and links visually a bit so we can more easily see which ones are active or not. For now we'll simply show locked nodes as grey (with alpha = 32 instead of 255) instead of white:

```lua
function Node:update(dt)
    ...

    if fn.any(bought_node_indexes, self.id) then self.bought = true
    else self.bought = false end
end

function Node:draw()
    local r, g, b = unpack(default_color)
    love.graphics.setColor(background_color)
    love.graphics.circle('fill', self.x, self.y, self.w)
    if self.bought then love.graphics.setColor(r, g, b, 255)
    else love.graphics.setColor(r, g, b, 32) end
    love.graphics.circle('line', self.x, self.y, self.w)
    love.graphics.setColor(r, g, b, 255)
end
```

And for the links:

```lua
function Line:update(dt)
    if fn.any(bought_node_indexes, self.node_1_id) and 
       fn.any(bought_node_indexes, self.node_2_id) then 
      	self.active = true 
    else self.active = false end
end

function Line:draw()
    local r, g, b = unpack(default_color)
    if self.active then love.graphics.setColor(r, g, b, 255)
    else love.graphics.setColor(r, g, b, 32) end
    love.graphics.line(self.node_1.x, self.node_1.y, self.node_2.x, self.node_2.y)
    love.graphics.setColor(r, g, b, 255)
end
```

We only activate a line if both of its nodes have been bought, which makes sense. If we say that `bought_node_indexes = {1}` in the SkillTree room constructor, now we'd get something like this:

<p align="center">
<img src="https://vgy.me/813KZ0.png">
</p>

And if we say that `bought_node_indexes = {1, 2}`, then we'd get this:

<p align="center">
<img src="https://vgy.me/VDZs0t.png">
</p>

And this is working as we expected. Now what we want to do is add the logic necessary so that whenever we click on a node it will be bought if its connected to another node that has been bought. Figuring out if we have enough skill points to buy a certain node, or to add a confirmation step before fully committing to buying the node will be left as an exercise.

Before we make it so that only nodes connected to other bought nodes can be bought, we first must fix a small problem with the way we're defining our tree. This is the definition we have now:

```lua
tree = {}
tree[1] = {x = 0, y = 0, links = {2}}
tree[2] = {x = 48, y = 0, stats = {'4% Increased HP', 'hp_multiplier', 0.04}, links = {3}}
tree[3] = {x = 96, y = 0, stats = {'6% Increased HP', 'hp_multiplier', 0.06}, links = {4}}
tree[4] = {x = 144, y = 0, stats = {'4% Increased HP', 'hp_multiplier', 0.04}}
```

One of the problems with this definition is that it's unidirectional. And this is a reasonable thing to expect, since if it were unidirectional we'd have to define connections multiple times across multiple nodes like this:

```lua
tree = {}
tree[1] = {x = 0, y = 0, links = {2}}
tree[2] = {x = 48, y = 0, stats = {'4% Increased HP', 'hp_multiplier', 0.04}, links = {1, 3}}
tree[3] = {x = 96, y = 0, stats = {'6% Increased HP', 'hp_multiplier', 0.06}, links = {2, 4}}
tree[4] = {x = 144, y = 0, stats = {'4% Increased HP', 'hp_multiplier', 0.04, links = {3}}}
```

And while there's no big problem in having to do this, we can make it so that we only have to define connections once (in either direction) and then we can apply an operation that will automatically make connections also be defined in the opposing direction. 

The way we can do this is by going over the list of all nodes, and then for each node going over its links. For each link we find, we go over to that node and add the current node to its links. So, for instance, if we're on node 1 and we see that it's linked to 2, then we move over to node 2 and add 1 to its links list. In this way we'll make sure that whenever we have a definition going one way it will also go the other. In code this looks like this:

```lua
function SkillTree:new()
    ...
    self.tree = table.copy(tree)
    for id, node in ipairs(self.tree) do
        for _, linked_node_id in ipairs(node.links or {}) do
            table.insert(self.tree[linked_node_id], id)
        end
    end
    ...
end
```

The first thing to notice here is that instead of using the global `tree` variable now, we're copying it locally to the `self.tree` attribute and then using that attribute instead. Everywhere on the SkillTree, Node and Line objects we should change references to the global `tree` to the local SkillTree `tree` attribute instead. We need to do this because we're going to change the tree's definition by adding numbers to the links table of some nodes, and generally (because of what I outlined in article 10) we don't want to be changing global variables in that way. This means that every time we enter the SkillTree room, we'll copy the global definition over to a local one and use the local one instead.

Given this, we now go over all nodes in the tree and back-link nodes to each other like we said we would. It's important to use `node.links or {}` inside the `ipairs` call because some nodes might not have their links table defined. It's also important to note that we do this before creating Node and Line objects, even though it's not really necessary to do that. 

An additional thing we can do here is to note that sometimes a `links` table will have repeated values. Depending on how we define the `tree` table sometimes we'll place nodes bi-directionally, which means that they'll already be everywhere they should be. This isn't really a problem, except that it might result in the creation of multiple Line objects. So to prevent that, we can go over the tree again and make it so that all `links` tables only contain unique values:

```lua
function SkillTree:new()
    ...
    for id, node in ipairs(self.tree) do
        if node.links then
            node.links = fn.unique(node.links)
        end
    end
    ...
end
```

Now the only thing left to do is making it so that whenever we click a node, we check to see if its linked to an already bought node:

```lua
function Node:update(dt)
    ...
    if self.hot and input:pressed('left_click') then
        if current_room:canNodeBeBought(self.id) then
            if not fn.any(bought_node_indexes, self.id) then
                table.insert(bought_node_indexes, self.id)
            end
        end
    end
    ...
end
```

And so this means that if a node is being hovered over and the player presses the left click button, we'll check to see if this node can be bought through SkillTree's `canNodeBeBought` function (which we still have to implement), and then if it can be bought we'll add it to the global `bought_node_indexes` table. Here we also take care to not add a node twice to that table. Although if we add it more than once it won't really change anything or cause any bugs.

The `canNodeBeBought` function will work by going over the linked nodes to the node that was passed in and seeing if any of them are inside the `bought_node_indexes` table. If that's true then it means this node is connected to an already bought node which means that it can be bought:

```lua
function SkillTree:canNodeBeBought(id)
    for _, linked_node_id in ipairs(self.tree[id]) do
        if fn.any(bought_node_indexes, linked_node_id) then return true end
    end
end
```

And this should work as expected:

<p align="center">
<img src="https://vgy.me/4xbfE4.gif">
</p>

The very last idea we'll go over is how to apply our selected nodes to the player. This is simpler than it seems because of how we decided to structure everything in articles 11 and 12. The tree definition looks like this now:

```lua
tree = {}
tree[1] = {x = 0, y = 0, links = {2}}
tree[2] = {x = 48, y = 0, stats = {'4% Increased HP', 'hp_multiplier', 0.04}, links = {3}}
tree[3] = {x = 96, y = 0, stats = {'6% Increased HP', 'hp_multiplier', 0.06}, links = {4}}
tree[4] = {x = 144, y = 0, stats = {'4% Increased HP', 'hp_multiplier', 0.04}}
```

And if you notice, we have the second stat value being a string that should point to a variable defined in the Player object. In this case the variable is `hp_multiplier`. If we go back to the Player object and look for where `hp_multiplier` is used we'll find this:

```lua
function Player:setStats()
    self.max_hp = (self.max_hp + self.flat_hp)*self.hp_multiplier
    self.hp = self.max_hp
    ...
end
```

It's used in the `setStats` function as a multiplier for our base HP added by some flat HP value, which is what we expected. The behavior we want out of the tree is that for all nodes inside `bought_node_indexes`, we'll apply their stat to the appropriate player variable. So if we have nodes 2, 3 and 4 inside that table, then the player should have an `hp_multiplier` that is equal to 1.14 (0.04+0.06+0.04 + the base which is 1). We can do this fairly simply like this:

```lua
function treeToPlayer(player)
    for _, index in ipairs(bought_node_indexes) do
        local stats = tree[index].stats
        for i = 1, #stats, 3 do
            local attribute, value = stats[i+1], stats[i+2]
            player[attribute] = player[attribute] + value
        end
    end
end
```

We define this function in `tree.lua`. As expected, we're going over all bought nodes and then going over all their stats. For each stat we're taking the attribute (`'hp_multiplier'`) and the value (0.04, 0.06) and applying it to the player. In the example we talked the `player[attribute] = player[attribute] + value` line is parsed to `player.hp_multiplier = player.hp_multiplier + 0.04` or `player.hp_multiplier = player.hp_multiplier + 0.06`, depending on which node we're currently looping over. This means that by the end of the outer for, we'll have applied all passives we bought to the player's variables. 

It's important to note that different passives will need to be handled slightly differently. Some passives are booleans, others should be applied to variables which are Stat objects, and so on. All those differences need to be handled inside this function.

<br>

**224. (CONTENT)** Implement skill points. We have a global `skill_points` variable which holds how many skill points the player has. This variable should be decreased by 1 whenever the player buys a new node in the skill tree. The player should not be allowed to buy more nodes if he has no skill points. The player can buy a maximum of 100 nodes. You may also want to change these numbers around a bit if you feel like it's necessary. For instance, in my game the cost of each node increases based on how many nodes the player has already bought. 

**225. (CONTENT)** Implement a step before buying nodes where the player can cancel his choices. This means that the player can click on nodes as if they were being bought, but to confirm the purchase he has to hit the "Apply Points" button. All selected nodes can be cancelled if he clicks the "Cancel" button instead. This is what it looks like:

<p align="center">
<img src="https://user-images.githubusercontent.com/409773/41511103-eb209372-7246-11e8-982b-5dbbd14a0d04.gif">
</p>

**226. (CONTENT)** Implement the skill tree. You can implement this skill tree to whatever size you see fit, but obviously the bigger it is the more possible interactions there will be and the more interesting it will be as well. This is what my tree looks like for reference:

<p align="center">
<img src="https://user-images.githubusercontent.com/409773/41511104-eb46ceca-7246-11e8-8345-efff87ebfeab.gif">
</p>

Don't forget to add the appropriate behaviors for each different type of passive in  the `treeToPlayer` function!

<br>

## END

And with that we end this article. The next article will focus on the Console room and the one after that will be the final one. In the final article we'll go over a few things, one of them being saving and loading things. One aspect of the skill tree that we didn't talk about was saving the player's bought nodes. We want those nodes to remain bought through playthroughs as well as after the player closes the game, so in the final article we'll go over this in more detail.

And like I said multiple times before, if you don't feel like it you don't need to implement a skill tree. If you've followed along so far then you already have all the passives implemented from articles 11 and 12 and you can present them to the player in whatever way you see fit. I chose a tree, but you can choose something else if you don't feel like doing a big tree like this manually is a good idea.

<br>

---

If you liked these tutorials and want to support the writing of more like these in the future:

* ### [BYTEPATH on Steam](http://store.steampowered.com/app/760330/BYTEPATH/)
* ### [BYTEPATH-tutorial on gumroad](https://gumroad.com/l/BYTEPATH-tutorial)

Buying the tutorial gives you access to the game's full source code, answers to exercises from articles 1 through 9, the code separated by articles (what the code should look like at the end of each article) and a Steam key to the game.
