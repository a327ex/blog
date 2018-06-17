`2016-01-01 07:05`

This post will explain how to make trails like the ones in [Downwell](https://store.steampowered.com/app/360740/) from scratch using [LÖVE](https://love2d.org/). There's another tutorial on this written by [Zack Bell](https://twitter.com/ZackBellGames) [here](http://zackbellgames.com/2015/10/26/gmtweekly-downwell-style-motion-trail/). If you don't know how to use LÖVE/Lua but have some programming knowledge then this tutorial might also serve as a learning exercise on LÖVE/Lua itself.

<p align="center">
<img src="https://i.imgur.com/gIA3Qxz.gif"/>
</p>

<br>

## LÖVE

Before starting with the actual effect we need to get some base code ready, basically just an organized way of creating and deleting entities as well as drawing them to the screen. Since LÖVE doesn't really come with this built-in we need to build it ourselves.

The first thing to do is create a folder for our project and in it a `main.lua` file with the following contents:

```lua
function love.load()

end

function love.update(dt)

end

function love.draw()

end
```

When a LÖVE project is loaded, `love.load()` is called on startup once and then `love.update` and `love.draw` are called every frame. To run this basic LÖVE project you can check out the [Getting Started](https://love2d.org/wiki/Getting_Started) page. After you manage to run it you should see a black screen.

<br>

## GameObject

Now, we'll need some entities in our game and for that we'll use an OOP library called [classic](https://github.com/rxi/classic). Like the github page says, after downloading the library and placing the classic folder in the same folder as the `main.lua` file, you can require it by doing:

```lua
Object = require 'classic/classic'

function love.load()
...
```

Now `Object` is a global variable that holds the definition of the classic library, and with it you can create new classes, like `Point = Object:extend()`. We'll use this to create a `GameObject` class which will be the main class we use for our entities:

```lua
-- in GameObject.lua
GameObject = Object:extend()

function GameObject:new()

end

function GameObject:update(dt)

end

function GameObject:draw()

end
```

Since we defined this in `GameObject.lua`, we need to require that file in the `main.lua` script, otherwise we won't have access to this definition:

```lua
-- in main.lua
Object = require 'classic/classic'
require 'GameObject'
```

And now `GameObject` is a global variable that holds the definition of the `GameObject` class. Alternatively, we could have defined the `GameObject` variable locally in `GameObject.lua` and then made it a global variable in `main.lua`, like so:

```lua
-- in GameObject.lua
local GameObject = Object:extend()

function GameObject:new()
...
...

return GameObject
```

```lua
-- in main.lua
Object = require 'classic/classic'
GameObject = require 'GameObject'
```

Sometimes, like when writing libraries for other people, this is a better way of doing things so you don't pollute their global state with your library's variables. This is what classic does as well, which is why you have to initialize it by assigning it to the `Object` variable. One good result of this is that since we're assigning a library to a variable, if you wanted to you could have named `Object` as `Class` instead, and then your class definitions would look like `GameObject = Class:extend()`.

Finally, a `GameObject` needs to have some properties. One simple setup that I've found useful was to make it so that the constructor calls for all my objects follow the same pattern of `ClassName(x, y, optional_arguments)`. I've found that all objects need to have some position (and if they don't then that can just be `0, 0`) and then `optional_arguments` is a table with as many optional arguments as you want it to have. For our `GameObjects` this would look like this:

```lua
-- in GameObject.lua
local GameObject = Object:extend()

function GameObject:new(x, y, opts)
  self.x, self.y = x, y
  local opts = opts or {} -- this handles the case where opts is nil
  for k, v in pairs(opts) do self[k] = v end
end
```

```lua
-- in main.lua
...
function love.load()
  game_object = GameObject(100, 100, {foo = 1, bar = true, baz = 'hue'})
  print(game_object.x, game_object.foo, game_object.bar, game_object.baz)
end
```

And so in this example you'd create a `GameObject` instance called `game_object` with those attributes. You can read up more about how the `for` in the constructor works [here](https://github.com/adonaac/blog/issues/8) in the **Objects, attributes and methods in Lua** section.

Another thing from the code above is the usage of the `print` function. It's a useful tool for debugging but you need to be running your application from a console, or if you're on Windows, whatever hooks you have created to run your project, you need to add the `--console` option so that a console shows up and `print` statements are printed there. `print` statements will not be printed on the main game screen and it's kind of a pain to debug things and tie them to the main screen, so I highly recommend figuring out how to make the console show up as well whenever you run the game.

<br>

## Object Creation and Deletion

The trail effect is achieved by creating multiple trail instances and quickly deleting them (like after 0.2 seconds they were created), so because of that we need some logic for object creation and deletion. Object creation was shown above, the only additional step we need is adding the object to a table that will hold all of them ([`table.insert`](https://wiki.garrysmod.com/page/table/insert)):

```lua
function love.load()
  game_objects = {}
  createGameObject(100, 100)
end

function love.update(dt)
  for _, game_object in ipairs(game_objects) do
    game_object:update(dt)
  end
end

function love.draw()
  for _, game_object in ipairs(game_objects) do
    game_object:draw()
  end
end

function createGameObject(x, y, opts)
  local game_object = GameObject(x, y, opts)
  table.insert(game_objects, game_object)
  return game_object -- return the instance in case we wanna do anything with it
end
```

So with this setup I created a function called `createGameObject`, which creates a new `GameObject` instance and adds it to the `game_objects` table. The objects in this table are being updated and drawn every frame in `love.update` and `love.draw` respectively (and they need to have both those functions defined). If you run this example nothing will happen still, but if you change `GameObject:draw` a bit to draw something to the screen, then you should see it being drawn ([`love.graphics.circle`](https://love2d.org/wiki/love.graphics.circle)):

```lua
function GameObject:draw()
  love.graphics.circle('fill', self.x, self.y, 25)
end
```

For deleting objects we need a few extra steps. First, for each `GameObject` we need to create a new variable called `dead` which will tell you if this `GameObject` is alive or not:

```lua
function GameObject:new(x, y, opts)
  ...
  self.dead = false
end
```

Then, we need to change our update logic a bit to take into account dead entities, and making sure that we remove them from the `game_objects` list ([`table.remove`](https://wiki.garrysmod.com/page/table/remove)):

```lua
function love.update(dt)
  for i = #game_objects, 1, -1 do
    local game_object = game_objects[i]
    game_object:update(dt)
    if game_object.dead then table.remove(game_objects, i) end
  end
end
```

And so whenever a `GameObject` instance has its `dead` attribute set to `true`, it will automatically get removed from the `game_objects` list. One of the important things to note here is that we're going through the list backwards. This is because in Lua, whenever something is removed from a list it gets resorted so as to leave no `nil` spaces in it. This means that if object `1` and `2` need to be deleted, if we go with a normal forwards loop, object `1` will be deleted, the list will be resorted and now object `2` will be object `1`, but since we already went to the next iteration, we'll not get to delete the original object `2` (because it's in the first position now). To prevent this from happening we go over the list backwards.

To test if this works out we can bind the deletion of a single instance we created to a mouse click ([`love.mousepressed`](https://love2d.org/wiki/love.mousepressed)):

```lua
function love.load()
  game_objects = {}
  game_object = createGameObject(100, 100)
end

...

function love.mousepressed(x, y, button)
  if button == 1 then -- 1 = left click
    game_object.dead = true
  end
end
```

Finally, one thing we can do with the mouse is binding `game_object`'s position to the mouse position ([`love.mouse.getPosition`](https://love2d.org/wiki/love.mouse.getPosition)):

```lua
function GameObject:update(dt)
  self.x, self.y = love.mouse.getPosition()
end
```

<br>

## Screen Resolution

Before we move on to making the actual trails and getting into the meat of this, we need to do one last thing. A game like Downwell has a really low native resolution that gets scaled up to the size of the screen, and this creates a good looking pixelated effect on whatever you draw to the screen. The default resolution a LÖVE game uses is `800x600` and this is a lot higher than what we need. So to decrease this to `320x240` what we can do is play with `conf.lua`, LÖVE's configuration file, and change the default resolution to our desired size. To do this, create a file named `conf.lua` at the same level that `main.lua` is in, and fill it with these contents ([`conf.lua`](https://love2d.org/wiki/Config_Files)):

```lua
function love.conf(t)
    t.window.width = 320
    t.window.height = 240
end
```

Now, to scale this up to, say, `960x720` (scaling it up by 3) while maintaining the pixel size of `320x240`, we need to draw the whole screen to a [canvas](https://love2d.org/wiki/Canvas) and then scale that up by 3. In this way, we'll always be working as if everything were at the small native resolution, but at the last step it will be drawn way bigger ([`love.graphics.newCanvas`](https://love2d.org/wiki/love.graphics.newCanvas)):

```lua
function love.load()
  ...
  main_canvas = love.graphics.newCanvas(320, 240)
  main_canvas:setFilter('nearest', 'nearest')
end

function love.draw()
  love.graphics.setCanvas(main_canvas)
  love.graphics.clear()
  for _, game_object in ipairs(game_objects) do
      game_object:draw()
  end
  love.graphics.setCanvas()

  love.graphics.draw(main_canvas, 0, 0, 0, 3, 3)
end
```

First we create a new canvas, `main_canvas`, with the native size and set its filter to `nearest` (so that it scales up with the nearest neighbor algorithm, essential for pixel art). Then, instead of just drawing the game objects directly to the screen, we set `main_canvas` with [`love.graphics.setCanvas`](https://love2d.org/wiki/love.graphics.setCanvas), clear the contents from this canvas from the last frame with [`love.graphics.clear`](https://love2d.org/wiki/love.graphics.clear), draw the game objects, and then draw the canvas scale up by 3 ([`love.graphics.draw`](https://love2d.org/wiki/love.graphics.draw)). This is what it looks like:

<p align="center">
<img src="https://i.imgur.com/UAy6TQH.png"/>
</p>

Compared to what it looked like before the pixel art effect is there, so it seems to be working. You might wanna also make the window itself bigger (instead of 320x240), and you can do that by using [`love.window.setMode`](https://love2d.org/wiki/love.window.setMode):

```lua
function love.load()
  ...
  love.window.setMode(960, 720)
end
```

If you're following along by coding this yourself you might have noticed that the mouse position code is now broken. This is because `love.mouse.getPosition` works in based on the screen size, and if the mouse is now on the middle of the screen `480, 360`, this is bigger than `320, 240`, which means the circle won't be drawn on the screen. Basically because of the way we're using the canvas we only work in the `320, 240` space, but the screen is bigger than that. To really solve this we should use some kind of camera system, but for this example we can just do it the quick way and divide the mouse position by 3:

```lua
function GameObject:update(dt)
  local x, y = love.mouse.getPosition()
  self.x, self.y = x/3, y/3
end
```

<br>

## Timing and Multiple Object Types

Now we have absolutely everything we need to actually make the trail effect. That was a lie. First we need a timing library. This is because we need an easy way to delete an object `n` seconds after it has been created and an easy way to tween an object's properties, since this is what trails generally do. To do this I'll use [HUMP](https://github.com/vrld/hump). To install it, download it and place it on the project's folder, then require the timer module:

```lua
Object = require 'classic/classic'
GameObject = require 'GameObject'
Timer = require 'hump/timer'
```

Now we can create an instance of a timer to use. I find it a good idea to create one timer per object that needs a timer, but for this examples it's fine it we just use a global one:

```lua
function love.load()
  timer = Timer()
  ...
end

function love.update(dt)
  timer.update(dt)
  ...
end
```

We can test to see if the timer works by using one of its functions, [`after`](http://hump.readthedocs.org/en/latest/timer.html#Timer.after). This function takes as arguments a number `n` and a function which is executed after `n` seconds:

```lua
function love.load()
  timer = Timer()
  timer.after(4, function() game_object.dead = true end)
  ...
end
```

And so in this example `game_object` will be deleted after the game has run for `4` seconds.

---

And this is finally the very last thing we need to do before we can actually do trails, which is defining multiple object types. Each trail object (that will get deleted really fast) will need to be an instance of some class, but it can't be the `GameObject` class, since we want that class to have the behavior that follows the mouse and draws the circle. So now what we need to do is come up with a way of supporting multiple object types. There are multiple ways of doing this, but I'll go with the simple one:

```lua
function createGameObject(type, x, y, opts)
  local game_object = _G[type](x, y, opts)
  table.insert(game_objects, game_object)
  return game_object -- return the instance in case we wanna do anything with it
end
```

All we changed in this function is accepting a `type` variable and then using that with `_G`. `_G` is the table that holds all global state in Lua. Since it's just a table, you can access the values in it by using the appropriate keys, which happen to be the names of the variables. So for instance `_G['game_object']` contains the `GameObject` instance that we've been using so far, and `_G['GameObject']` contains the `GameObject` class definition. So whenever using `createGameObject` instead of calling `createGameObject(x, y, opts)` we now have to do `createGameObject(class_name, x, y, opts)`:

```lua
function love.load()
  game_object = createGameObject('GameObject', 100, 100)
  ...
end
```

When we add the trail objects all we need to do to create them instead of a `GameObject` is changing the first parameter we pass to `createGameObject`.

<br>

## Trails

Now to make trails. One way trails can work is actually pretty simple. You have some object you want to emit some sort of trail effect, and then every `n` seconds (like 0.02) you create a `Trail` object that will look like you want the trail to look. This `Trail` object will usually be deleted very quickly (like 0.2 seconds after its creation) otherwise you'll get a really really long trail. We have everything we need to do that now, so let's start by creating the `Trail` class:

```lua
-- in Trail.lua
local Trail = Object:extend()

function Trail:new(x, y, opts)
  self.dead = false
  self.x, self.y = x, y
  local opts = opts or {} -- this handles the case where opts is nil
  for k, v in pairs(opts) do self[k] = v end

  timer.after(0.15, function() self.dead = true end)
end

function Trail:update(dt)

end

function Trail:draw()
  love.graphics.circle('fill', self.x, self.y, self.r)
end

return Trail
```

The first 4 lines of the constructor are exactly the same as for `GameObject`, so this is just standard object attributes. The important part comes in the next line that uses the timer. Here all we're doing is deleting this trail object `0.15` seconds after it has been created. And in the draw function we're simply drawing a circle and using `self.r` as its radius. This attribute will be specified in the creation call (via `opts`) so we don't need to worry about it in the constructor.

Next, we need to create the trails and we'll do this in the `GameObject` constructor. Every `n` seconds we'll create a new `Trail` instance at the current `game_object` position:

```lua
function GameObject:new(x, y, opts)
  ...
  timer.every(0.01, function() createGameObject('Trail', self.x, self.y, {r = 25}) end)
end

```

So here every `0.01` seconds, or every frame, we're creating a trail at `self.x, self.y` with `r = 25`. One important thing to realize is that `self.x, self.y` inside that function is always up to date with the current position instead of only the `self.x, self.y` values we have in the constructor. This is because that function is getting called every frame and because of the way [closures](http://www.lua.org/pil/6.1.html) work in Lua, that function has access to `self`, and `self` is always being updated so it all works out. And that should look like this:

<p align="center">
<img src="https://i.imgur.com/ApKr2ni.gif"/>
</p>

Not terribly amazing but we're getting somewhere.

<br>

## Downwell Trail Effect

To make the Downwell trail effect we want to erase lines from the trail only, either horizontally or vertically. To do this we need to separate the drawing of the main game object and the drawing of the trails into separate canvases, since we only want to apply the effect to one of those types of objects. The first thing we need to do to get this going is being able to differentiate between which objects are of which class when drawing, and currently we have no way of doing that. Instances of classes created by classic have no default attributes magically set to them that say "I'm of this class", so we have to do that manually.

```lua
-- in main.lua
function createGameObject(type, x, y, opts)
  local game_object = _G[type](type, x, y, opts)
  ...
end
```

```lua
-- in GameObject.lua
function GameObject:new(type, x, y, opts)
  self.type = type
  ...
end
```

```lua
-- in Trail.lua
function Trail:new(type, x, y, opts)
  self.type = type
  ...
end
```

All we've done here is added the `type` attribute to all objects, so that now we can do stuff like `if object.type == 'Trail'` and figure out what kind of object we're dealing with.

Now before we separate object drawing to different canvases we should create those:

```lua
function love.load()
  ...
  game_object_canvas = love.graphics.newCanvas(320, 240)
  game_object_canvas:setFilter('nearest', 'nearest')
  trail_canvas = love.graphics.newCanvas(320, 240)
  trail_canvas:setFilter('nearest', 'nearest')
end
```

Those creation calls are exactly the same as the ones we used for creating our `main_canvas`. Now, to separate trails and the game object, we need to draw the trails to `trail_canvas`, draw the game object to `game_object_canvas`, draw both of those to the `main_canvas`, and then draw `main_canvas` to the screen while scaling it up. And that looks like this:

```lua
function love.draw()
  love.graphics.setCanvas(trail_canvas)
  love.graphics.clear()
  for _, game_object in ipairs(game_objects) do
    if game_object.type == 'Trail' then
      game_object:draw()
    end
  end
  love.graphics.setCanvas()

  love.graphics.setCanvas(game_object_canvas)
  love.graphics.clear()
  for _, game_object in ipairs(game_objects) do
    if game_object.type == 'GameObject' then
      game_object:draw()
    end
  end
  love.graphics.setCanvas()

  love.graphics.setCanvas(main_canvas)
  love.graphics.clear()
  love.graphics.draw(trail_canvas, 0, 0)
  love.graphics.draw(game_object_canvas, 0, 0)
  love.graphics.setCanvas()

  love.graphics.draw(main_canvas, 0, 0, 0, 3, 3)
end
```

There's totally some dumb stuff going on here, like for instance we're going over the list of objects twice now (even though we're not drawing twice), and that could totally be optimized. But really for an example this small this doesn't matter at all. What's important is that it works!

So now that we've separated different things into different render targets we can move on with our effect. The way to erase lines from all the trails is to draw lines to the trail canvas using the `subtract` blend mode. What this blend mode does is literally just subtract the values of the thing you're drawing from what's already on the screen/canvas. So in this case what we want is to draw a bunch of white lines `(1, 1, 1, 1)` with `subtract` enabled to `trail_canvas` (after the trails have been drawn), in this way the places where those lines would appear will become `(0, 0, 0, 0)` ([`love.graphics.setBlendMode`](https://love2d.org/wiki/love.graphics.setBlendMode)):

```lua
function love.load()
  ...
  love.graphics.setLineStyle('rough')
end

function love.draw()
  love.graphics.setCanvas(trail_canvas)
  love.graphics.clear()
  for _, game_object in ipairs(game_objects) do
    if game_object.type == 'Trail' then
      game_object:draw()
    end
  end

  love.graphics.setBlendMode('subtract')
  for i = 0, 360, 2 do
    love.graphics.line(i, 0, i, 240)
  end
  love.graphics.setBlendMode('alpha')
  love.graphics.setCanvas()

  ...
end
```

And this is what that looks like:

<p align="center">
<img src="https://i.imgur.com/NSbYgIj.gif"/>
</p>

One important thing to do before drawing this is to set the line style to `'rough'` using [`love.graphics.setLineStyle`](https://love2d.org/wiki/love.graphics.setLineStyle), since the default is `'smooth'` and that doesn't work with the pixel art style generally. And if you didn't really understand what's going on here, here's what the lines by themselves would look like if they were drawn normally:

<p align="center">
<img src="https://i.imgur.com/7FHFqCX.png"/>
</p>

So all we're doing is subtracting that from the trail canvas, and since the only places in the trail canvas where there are things to be subtracted from are the trails, we get the effect only there. Another thing to note is that the `subtract` blend mode idea would only work if you have a black background. For instance, I tried this effect in my game and this is what it looks like:

<p align="center">
<img src="https://i.imgur.com/j7wqARC.gif"/>
</p>

If I were to use the subtract blend mode here it just would look even worse than it does. So instead what I did was use `multiply`. Like the name indicates, it just multiplies the values instead of subtracting them. So the white lines `(1, 1, 1, 1)` won't change the output in any way, while the gaps `(0, 0, 0, 0)` will make whatever collides with them transparent. In this way you get the same effect, the only difference is that with subtract the white lines themselves result in transparency, while with multiply the gaps between them do.

<br>

## Details

Now we already have the effect working but we can make it more interesting. For starters, we can randomly draw extra lines so that it creates some random gaps in the final result:

```lua
function love.load()
  ...
  trail_lines_extra_draw = {}
  timer.every(0.1, function()
    for i = 0, 360, 2 do
      if love.math.random(1, 10) >= 2 then trail_lines_extra_draw[i] = false
      else trail_lines_extra_draw[i] = true end
    end
  end)
end

function love.draw()
  ...
  love.graphics.setBlendMode('subtract')
  for i = 0, 360, 2 do
    love.graphics.line(i, 0, i, 240)
    if trail_lines_extra_draw[i] then
      love.graphics.line(i+1, 0, i+1, 240)
    end
  end
  love.graphics.setBlendMode('alpha')
  ...
end
```

So here every `0.1` seconds, for every line we're drawing, we set some values to true or false to an additional table, `trail_lines_extra_draw`. If the value for some line in this table is true, then whenever we get to drawing that line, we'll also draw the another line 1 pixel to its right. Since we're looping over lines on a 2 by 2 pixels basis, this will create a section where there are 3 lines being drawn at once, and this will create the effect of a few lines looking like they're missing from the trail. You can play with different chances and times (every `0.05` seconds?) to see what you think looks best.

<p align="center">
<img src="https://i.imgur.com/NXwykDp.gif"/>
</p>

---

Now another thing we can do is tween the radius of the trail circles down before they disappear. This will give the effect a much more trail-like feel:

```lua
function Trail:new(type, x, y, opts)
  ...
  timer.tween(0.3, self, {r = 0}, 'linear', function() self.dead = true end)
end
```

I deleted the previous `timer.after` call that was here and changed it for this new [`timer.tween`](http://hump.readthedocs.org/en/latest/timer.html#Timer.tween). The tween call takes a number of seconds, the target table, the target value inside that table to be tweened, the tween method, and then an optional function that gets called when the tween ends. In this case, we're tweening `self.r` to `0` over `0.3` seconds using the `linear` interpolation method, and then when that is done we kill the trail object. That looks like this:

<p align="center">
<img src="https://i.imgur.com/G0ajNov.gif"/>
</p>

---

Another thing we can do is, every frame, adding some random amount within a certain range to the radius of the trail circles being drawn. This will give the trails an extra bit of randomization:

```lua
-- in main.lua
function randomp(min, max)
    return (min > max and (love.math.random()*(min - max) + max)) or (love.math.random()*(max - min) + min)
end
```

```lua
-- in Trail.lua
function Trail:draw()
    love.graphics.circle('fill', self.x, self.y, self.r + randomp(-2.5, 2.5))
end
```

Here we just need to define a function called `randomp`, which returns a float between `min` and `max`. In this case we use it to add, to the radius of every trail, every frame, a random amount between `-2.5` and `2.5`. And that looks like this:

<p align="center">
<img src="https://i.imgur.com/RVRMTWZ.gif"/>
</p>

---

Something else that's possible is rotating the angle of the lines based on the angle of the velocity vector of whatever is generating the trails. To do that first we need to figure out the velocity of our game object. An easy way to achieve this is storing its previous position, then subtracting the current position from the previous one and getting the angle of that:

```lua
function GameObject:new(type, x, y, opts)
  ...
  self.previous_x, self.previous_y = x, y
end

function GameObject:update(dt)
  ...
  self.angle = math.atan2(self.y - self.previous_y, self.x - self.previous_x)
  self.previous_x, self.previous_y = self.x, self.y
end
```

To calculate the current angle we use the current `self.x, self.y` and the values for `x, y` from the previous frame. Then after that, at the end of the update function, we set the values of the current frame to the `previous` variables. If you want to test if this actually works you can try `print(math.deg(self.angle))`. Keep in mind that LÖVE uses a reverse angle system (up from 0 is negative).

After we have the angle we can try to rotate the lines being drawn like this ([`push`](https://love2d.org/wiki/love.graphics.push), [`pop`](https://love2d.org/wiki/love.graphics.pop)):

```lua
-- in main.lua
function pushRotate(x, y, r)
    love.graphics.push()
    love.graphics.translate(x, y)
    love.graphics.rotate(r or 0)
    love.graphics.translate(-x, -y)
end

function love.draw()
  ...
  pushRotate(160, 120, game_object.angle + math.pi/2)
  love.graphics.setBlendMode('subtract')
  for i = 0, 360, 2 do
    love.graphics.line(i, 0, i, 240)
    if trail_lines_extra_draw[i] then
      love.graphics.line(i+1, 0, i+1, 240)
    end
  end
  love.graphics.setBlendMode('alpha')
  ...
end
```

`pushRotate` is a function that will make everything drawn after it rotated by `r` with the pivot position being `x, y`. This is useful in a lot of situations and in this instance we're using it to rotate all the lines by the angle of `game_object`. I added `math.pi/2` to the angle because it came out sideways for whatever reason... Anyway, that should look like this:

<p align="center">
<img src="https://i.imgur.com/RBphuir.gif"/>
</p>

Not really sure if it looks better or worse, looks like it's fine for slow moving situations but when its too fast it can get a bit chaotic (probably because I stop the object and then the angle changes abruptly midway).

One problem with this way of doing things is that since all the lines are being rotated but they're being drawn at first to fit the screen, you'll get places where the lines simply don't exist anymore and the effect fails to happen, which causes the trails to look all white. To prevent this we can just draw lines way beyond the screen boundaries on all directions, such that with any rotation happening  lines will still be drawn anyway:

```lua
  ...
  trail_lines_extra_draw = {}
  timer.every(0.1, function()
    for i = -360, 720, 2 do
      if love.math.random(1, 10) >= 2 then trail_lines_extra_draw[i] = false
      else trail_lines_extra_draw[i] = true end
    end
  end)
  ...
  pushRotate(160, 120, game_object.angle + math.pi/2)
  love.graphics.setBlendMode('subtract')
  for i = -360, 720, 2 do
    love.graphics.line(i, -240, i, 480)
    if trail_lines_extra_draw[i] then
      love.graphics.line(i+1, -240, i+1, 480)
    end
  end
  love.graphics.setBlendMode('alpha')
  ...
```

---

Finally, one last cool thing we can do is changing the shape of the main ball and of the trails based on its velocity. We can use the same idea from the example above, except to calculate the magnitude of the game object's velocity instead of the angle:

```lua
function GameObject:update(dt)
  ...
  self.vmag = Vector(self.x - self.previous_x, self.y - self.previous_y):len()
  ...
end
```

`Vector` was initialized in `main.lua` and it comes from `HUMP`. I'll omit that code because you should be able to do that by now. Anyway, here we calculate a value called `vmag`. This value is basically an indicator of how fast the object is moving in any direction. Since we already have the angle the magnitude of the velocity is all we really need. With that information we can do the following:

```lua
-- in main.lua
function map(old_value, old_min, old_max, new_min, new_max)
    local new_min = new_min or 0
    local new_max = new_max or 1
    local new_value = 0
    local old_range = old_max - old_min
    if old_range == 0 then new_value = new_min
    else
        local new_range = new_max - new_min
        new_value = (((old_value - old_min)*new_range)/old_range) + new_min
    end
    return new_value
end
```

```lua
-- in GameObject.lua
function GameObject:update(dt)
  ...
  self.vmag = Vector(self.x - self.previous_x, self.y - self.previous_y):len()
  -- print(self.vmag)
  self.xm = map(self.vmag, 0, 20, 1, 2)
  self.ym = map(self.vmag, 0, 20, 1, 0.25)
  ...
end

function GameObject:draw()
  pushRotate(self.x, self.y, self.angle)
  love.graphics.ellipse('fill', self.x, self.y, self.xm*15, self.ym*15)
  love.graphics.pop()
end
```

And that looks like this:

<p align="center">
<img src="https://i.imgur.com/W4Nf21h.gif"/>
</p>

Let's go part by part. First the `map` function. This is a function that takes some value named `old_value`, two values named `old_min` and `old_max` representing the range that `old_value` can take, and two other values named `new_min` and `new_max`, representing the desired new range. With all this it returns a new value that corresponds to `old_value`, but if it were in the `new_min`, `new_max` range. For instance, if we do `map(0.5, 0, 1, 0, 100)` we'll get `50` back. If we do `map(0.5, 0, 1, 200, -200)` we'll get `0` back. If we do `map(0.5, 0, 1, -200, 100)` we'll get `-50` back. And so on...

We use this function to calculate the variables `self.xm` and `self.ym`. These variables will be used as multiplication factors to change the size of the ellipse we're drawing. One thing to keep in mind is that when drawing the ellipse we're first rotating everything by `self.angle`, this means that we should always consider what we want to happen when drawing as if we were at angle `0` (to the right), because all other angles will just happen automatically from that.

Practically, this means that we should consider the changes in shape of our ellipse from a horizontal perspective. So when the game object is going really fast we want the ellipse to stretch horizontally and shrink vertically, which means that the values we find for `self.xm` and `self.ym` have to reflect that. As you can see, for `self.xm`, when `self.vmag` is `0` we get `1` back, and when it's `20` we get `2`, meaning, when `self.vmag` is `20` we double the size of the ellipse horizontally, bringing it up to `30`. For `self.vmag` values greater than that we increase it even more. Similar logic applies to `self.ym` and how it shrinks.

It's important to note that those values (`0`, `20`, `2`, `0.25`) used in those two map functions up there were reached by trial and error and seeing what looks good. Most importantly, they're really exaggerated so that the effect can actually be seen well enough in these gifs. I would personally go with the values `0` and `60` instead of `20` if I were doing this normally.

Finally, we can also apply this to the `Trail` objects:

```lua
-- in GameObject.lua
function GameObject:new(type, x, y, opts)
    ...
    timer.every(0.01, function() createGameObject('Trail', self.x, self.y, {r = 20, 
        xm = self.xm, ym = self.ym, angle = self.angle})
    end)
end
```

```lua
-- in Trail.lua
function Trail:draw()
    pushRotate(self.x, self.y, self.angle)
    love.graphics.ellipse('fill', self.x, self.y, self.xm*(self.r + randomp(-2.5, 2.5)), 
        self.ym*(self.r + randomp(-2.5, 2.5)))
    love.graphics.pop()
end
```

We just change the way trails are being drawn to be the same as the main game object. We also make sure to send the trail object the information needed (`self.xm, self.ym, self.angle`), otherwise it can't really do anything. And that looks like this:

<p align="center">
<img src="https://i.imgur.com/umE0KON.gif"/>
</p>

There are a few bugs like the velocity going way too big and the `self.ym` value becoming negative, making each ellipse really huge, but this can be fixed by just doing some checks. Also, sometimes when you go kinda fast with abrupt angle changes you can see the shapes of each ellipse individually and that looks kinda broken. I don't know exactly how I'd fix that other than not having abrupt angle changes on an object with that kind of trail effect.

<br>

## END

That's it. I hope we've all learned something about friendship today.

<p align="center">
<img src="https://i.imgur.com/xRCw1X5.gif"/>
</p>
