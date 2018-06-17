`2015-12-25 12:55`

In this post I'll explain how I approached creating a simple replay system using Lua. There are two main ways I know of to do this, one is storing inputs and ensuring your simulation is deterministic, another is storing all state needed every frame to play the game back. I decided to go with the second one for now because it seemed easier. The game I'll use as an example for this is [the one I made during the last Ludum Dare](http://ludumdare.com/compo/ludum-dare-34/?action=preview&uid=7173) and you can find the code for all this [here](https://github.com/adonaac/ld34).

<p align="center">
<img src="https://i.imgur.com/uv5rzC7.gif"/>
</p>

<br>

## Objects, attributes and methods in Lua

All objects in Lua are made out of tables (essentially hash tables), with attribute and method names being the keys, and values and functions being the values. So, for instance, suppose a class definition like this:

```lua
GameObject = class('GameObject')

function GameObject:new(x, y)
    self.x = x
    self.y = y
end
```

Since In Lua, `table.something` is the same as `table["something"]`, accessing a key in a table looks the same as accessing the attribute of an object. So we could say that the class definition above is essentially the same as this:

```lua
object = {}
object["new"] = function(x, y)
    object.x = x
    object["y"] = y
end
```

This property of the language has a few interesting results. For this article the one we're going to focus on in this post is the ability to add and change attributes of an object with a lot of ease. For instance, consider the slightly modified class definition:

```lua
GameObject = class('GameObject')

function GameObject:new(x, y, opts)
    self.x = x
    self.y = y
    for k, v in pairs(opts) do self[k] = v end
end
```

We've added an `opts` argument there and we're doing something with it in a loop. Assuming `opts` is a table, what the loop is doing is going through all of its keys and values (named `k` and `v` respectively) and then assigning `v` to `self[k]`. What this does is that if you have a `GameObject` construction call that looks like this:

```lua
game_object = GameObject(100, 200, {hp = 100, damage = 10, layer = 'Default'})
```

Now on top of `game_object.x` and `game_object.y` being `100` and `200`, you also have `game_object.layer` being `'Default'`, and `game_object.damage` being `10`. This is because the loop in the constructor went through all keys and values in the `opts` table and assigned them to the object being constructed by saying `self[k] = v`.

<br>

## Replay State

The replay system we're building uses this idea heavily. But before we explore that I need to explain at a high level how it will work. The way we're doing replays is by storing state every frame and then when we need to play it back we just read that stored state. The easiest way I could imagine of doing this was, when replaying, having multiple instances of a class called `ReplayObject`. These objects will be responsible for reading the saved state and changing themselves (much like in the `opts` example above) to match the state of some object that was saved on that frame. Then when all that is done for all objects, all replay objects will be drawn, and since their state is matching that of some original object that was recorded, it will be just like if it was the real thing.

If this didn't make much sense let's try looking at a real world example. Below I have the `draw` definition of an object in my game, a particle effect for when bats spawn:

```lua
function BatSpawnParticle:draw()
    love.graphics.setColor(52, 52, 54)
    local w, h = math.rangec(self.v, 0, 400, 0, 1), math.rangec(self.v, 0, 400, 3, 1)
    pushRotate(self.x, self.y, self.angle)
    draft:ellipse(self.x, self.y, math.ceil(self.m*math.max(self.v/8, 3)), 
        math.ceil(self.m*math.ceil(h)), 60, 'fill')
    love.graphics.pop()
    love.graphics.setColor(255, 255, 255)
end
```

Inside a `ReplayObject` we'll be reusing the `BatSpawnParticle.draw` function somehow, since the replay object will be sort of mimicking a `BatSpawnParticle`, and because of that we need it to have all the state that is used in that draw function. Usually this boils down to saving everything that uses an attribute from `self`, so in this case: `self.x`, `self.y`, `self.angle`, `self.m` and `self.v`. And this would look like this:

```lua
function BatSpawnParticle:update(dt)
    self.timer:update(dt)
    self.x = self.x + self.v*math.cos(self.angle)*dt
    self.y = self.y + self.v*math.sin(self.angle)*dt

    if recording_replay then
        table.insert(replay_data[frame], {source_class_name = self.class_name, 
            layer = self.layer, x = self.x, y = self.y, angle = self.angle, 
            v = self.v, m = self.m})
    end
end
```

`replay_data` is the table we're using to store everything, and `replay_data[frame]` is just another table that holds all the saved state for this frame. We'll do this for every class in the game that we're interesting in saving for playback. In any case, now that we've added all the state needed to draw this object when it's replaying, we can look at what a `ReplayObject` looks like.

<p align="center">
<img src="https://i.imgur.com/nBHBi6e.gif"/>
</p>

<br>

## ReplayObject

At its core the replay object is very simple and it looks like this:

```lua
ReplayObject = class('ReplayObject')

function ReplayObject:new(x, y, opts)
    self.x = x
    self.y = y
    for k, v in pairs(opts) do self[k] = v end
end

function ReplayObject:draw()
    _G[self.source_class_name].draw(self)
end
```
The single line in the draw function is the one that makes use of another object's draw function. If you remember when saving the state of `BatSpawnParticle`, I said `source_class_name = self.class_name`. All my objects have their class name stored in `self.class_name` (including ReplayObjects), so `source_class_name` stores the class name of the object that this replay object is supposed to mimick. In this case it stores the string `'BatSpawnParticle'`. Another thing to note is that all my class definitions look like this:

```lua
BatSpawnParticle = class('BatSpawnParticle')
```

`class` is a function that creates the class definition, and `BatSpawnParticle` is just a variable that holds that class definition. Variables are global by default in Lua, so `BatSpawnParticle` is now a global variable that holds that class definition. All global variables in Lua are stored in the table `_G`, so, for instance, we can access the draw function of `BatSpawnParticle` by saying `_G['BatSpawnParticle'].draw`. Again, since functions are also just key/value pairs in a table we can do that safely.

<p align="center">
<img src="https://i.imgur.com/rFD1CyM.gif"/>
</p>

Finally, one last thing about that line in replay object's draw function is that it passes `self` to the draw function. If you go back and look at the `BatSpawnParticle:draw` definition it uses a `:`. In Lua, that's a shorthand for passing `self` as the first parameter, so `BatSpawnParticle:draw()` is exactly the same as `BatSpawnParticle.draw(self)`. The trick here is that `ReplayObject` is passing itself as `self` to `BatSpawnParticle.draw`, but since it already has all the state needed by that function (since that was what was saved), the function will work properly and the replay object will be drawn as if it were a `BatSpawnParticle`. This trick is what allows us to use only ReplayObjects to mimick and draw every single object in the game while replaying.

<br>

## Level

Finally, one last thing is needed to bring this all together, which is coordinating `ReplayObject` creation/destruction based on the `replay_data` for this frame. I have a `Level` class where all my game objects are updated, and there I can do something like this:

```lua
function Level:update(dt)
    if replaying then
        -- replay update
        1. match number of replay objects to that of tables saved in replay_data[frame]
        for i, replay_object in replay objects
            2. clear replay_object
            3. morph replay_object into replay_data[frame][i] 
            update replay_object
    else
        -- normal update
    end
end
```

Step 1 is important because it means we'll be reusing replay objects as much as possible. If the number of saved tables in `replay_data[frame]` is bigger than the current number of replay objects then we create new ones, otherwise we delete extra ones. This is going to be happening every frame such that at all times the number of replay objects alive is exactly the same as the number of objects saved for that frame. And that looks like this:

```lua
...
if #replay_data[frame] > #self.game_objects then
    local d = #replay_data[frame] - #self.game_objects
    for i = 1, d do self:createGameObject('ReplayObject', 0, 0) end

elseif #self.game_objects > #replay_data[frame] then
    local d = #self.game_objects - #replay_data[frame] 
    for i = 1, d do table.remove(self.game_objects, #self.game_objects) end
end
```

Step 2 is needed so that we clear the state of this replay object so that it can be reused again. If we don't do this what can happen is that in one frame a certain `ReplayObject` is used as a `BatSpawnParticle` and in the next it's used as the `Player`, and since we aren't clearing it now it has state left over from when it was a `BatSpawnParticle` and this can quickly lead to bugs. An easy way to clear an object is like this:

```lua
function ReplayObject:clear()
    for k, v in pairs(self) do
        if k ~= 'new' and k ~= 'update' and k ~= 'draw' and k ~= 'clear' then
            self[k] = nil
        end
    end
end 
```

We just go over all keys and values in `self` and verify that they're not things we don't want to clear, for instance, we don't want to erase the reference to the `clear` function itself. So once we know it's safe we just `nil` that value. This way, all attributes from a previous frame will be cleared and the object will be brand new for step 3, where we make this replay object mimick a saved object. This boils down to a single simple line:

```lua
-- while going over all replay objects...
for k, v in pairs(replay_data[frame][i]) do replay_object[k] = v end
```

After this, we should be able to draw each replay object as if it were a normal object that we saved and so the replay system works completely. It's important to notice that what we did for `BatSpawnParticle` in saving its state needs to be done for ALL game objects that need their state saved. Depending on how many objects you have and how complex your draw operations are this can be a bit of work.

<br>

## Replaying

I glossed over some details but here's an important one: every frame you're doing `frame++` on your frame counter, and at the start of every frame you're starting saying `replay_data[frame] = {}` so that the table that is going to hold all stored data in this frame is initialized. When replaying all you have to do is say `frame = 1` and set some `replaying` bool to true, and then you'll be back at frame 1, reading the replay data you saved back then. This means that you can also go back and forwards in time at will, as the gif below shows.

<p align="center">
<img src="https://i.imgur.com/gYTuJP5.gif"/>
</p>

While replaying, pausing the replay means not increasing the frame counter. Jumping forwards in time means increasing the frame counter by an amount bigger than 1. Jumping backwards in time means decreasing the frame counter by an amount bigger than 1. Going to the end of the replay means setting the frame counter to the size of `replay_data`. Going to the start of the replay means setting the frame counter to `1` or to the number of the frame where you started recording. All of these are relatively simple operations that let you have control over your playback.

<br>

## END

There are some drawbacks to this way of doing things, the main one being the size of a replay. Initially I built this because I wanted to be able to watch players play the game and be able to learn something from it. For instance, in this video, the player takes 4 minutes to figure out what exactly he's supposed to do in the game: (click to watch)


<p align="center">
<a href="http://www.youtube.com/watch?v=akuJ9selnCw"><img src="https://i.imgur.com/j7Yv7Qh.png"/></a>
</p>

This is totally not a cool thing because I'd randomly guess that like 50% of people who play the game end up quitting before they figure out what they have to do. This is an easy mistake to fix but it's the kind of thing that's also easy to miss if you don't actively watch people playing your game, and a replay system helps with that.

Sadly, using this current technique (with no optimizations, although I've tried a few optimizations and they didn't help that much), about 10 seconds of gameplay results in a 10mb file. This is a prohibitive size if I want people to send files to me, so eventually I'll get around to implementing the other way where I only record inputs (and maybe write an article about it too) for this purpose. Overall though this is was a cool thing to build because now at least making gifs out of the game is easier, since I can just pause everything, go back, forwards, move the camera around, and so on.
