## Introduction

In this final article we'll talk about a few subjects that didn't fit into any of the previous ones but that are somewhat necessary for a complete game. In order, what we'll cover will be: saving and loading data, achievements, shaders and audio.

### Saving and Loading

Because this game doesn't require us to save level data of any kind, saving and loading becomes very very easy. We'll use a library called [bitser](https://github.com/gvx/bitser/) to do it and two of its functions: [`dumpLoveFile`](https://github.com/gvx/bitser/blob/master/USAGE.md#dumplovefile) and [`loadLoveFile`](https://github.com/gvx/bitser/blob/master/USAGE.md#loadlovefile). These functions will save and load whatever data we pass it to a file using [`love.filesystem`](https://love2d.org/wiki/love.filesystem). As the link states, the files are saved to different directories based on your operating system. If you're on Windows then the file will be saved in `C:\Users\user\AppData\Roaming\LOVE`. We can use [`love.filesystem.setIdentity`](https://love2d.org/wiki/love.filesystem.setIdentity) to change the save location. If we set the identity to `BYTEPATH` instead, then the save file will be saved in `C:\Users\user\AppData\Roaming\BYTEPATH`.

In any case, we'll only need two functions: `save` and `load`. They will be defined in `main.lua`. Let's start with the save function:

```lua
function save()
    local save_data = {}
    -- Set all save data here
    bitser.dumpLoveFile('save', save_data)
end
```

The save function is pretty straightforward. We'll create a new `save_data` table and in it we'll place all the data we want to save. For instance, if we want to save how many skill points the player has, then we'll just say `save_data.skill_points = skill_points`, which means that `save_data.skill_points` will contain the value that our `skill_points` global contains. The same goes for all other types of data. It's important though to keep ourselves to saving values and tables of values. Saving full objects, images, and other types of more complicated data likely won't work. 

In any case, after we add everything we want to save to `save_data` then we simply call `bitser.dumpLoveFile` and save all that data to the `'save'` file. This will create a file called `save` in `C:\Users\user\AppData\Roaming\BYTEPATH` and once that file exists all the information we care about being saved is saved. We can call this function once the game is closed or whenever a round ends. It's really up to you. The only problem I can think of in calling it only when the game ends is that if the game crashes then the player's progress will likely not be saved, so that might be a problem.

Now for the load function:

```lua
function load()
    if love.filesystem.exists('save') then
        local save_data = bitser.loadLoveFile('save')
	-- Load all saved data here
    else
        first_run_ever = true
    end
end
```

The load function works very similarly except backwards. We call `bitser.loadLoveFile` using the name of our saved file (`save`) and then put all that data inside a local `save_data` table. Once we have all the saved data in this table then we can assign it to the appropriate variables. So, for instance, if now we want to load the player's skill points we'll do `skill_points = save_data.skill_points`, which means we're assigning the saved skill points to our global skill points variable.

Additionally, the load function needs a bit of additional logic to work properly. If it's the first time the player has run the game then the save file will not exist, which means that when try to load it we'll crash. To prevent this we check to see if it exists with [`love.filesystem.exists`](https://love2d.org/wiki/love.filesystem.exists) and only load it if it does. If it doesn't then we just set a global variable `first_run_ever` to true. This variable is useful because generally we want to do a few things differently if it's the first time the player has run the game, like maybe running a tutorial of some kind or showing some message of some kind that only first timers need. The load function will be called once in `love.load` whenever the game is loaded. It's important that this function is called after the `globals.lua` file is loaded, since we'll be overwriting global variables in it.

And that's it for saving/loading. What actually needs to be saved and loaded will be left as an exercise since it depends on what you decided to implement or not. For instance, if you implement the skill tree exactly like in article 13, then you probably want to save and load the `bought_node_indexes` table, since it contains all the nodes that the player bought.

<br>

### Achievements

Because of the simplicity of the game achievements are also very easy to implement (at least compared to everything else xD). What we'll do is simply have a global table called `achievements`. And this table will be populated by keys that represent the achievement's name, and values that represent if that achievement is unlocked or not. So, for instance, if we have an achievement called `'50K'`, which unlocks whenever the player reaches 50.000 score in a round, then `achievements['50K']` will be true if this achievements has been unlocked and false otherwise.

To exemplify how this works let's create the `10K Fighter` achievement, which unlocks whenever the player reaches 10.000 score using the Fighter ship. All we have to do to achieve this is set `achievements['10K Fighter']` to true whenever we finish a round, the score is above 10K and the ship currently being used by the player is 'Fighter'. This looks like this:

```lua
function Stage:finish()
    timer:after(1, function()
        gotoRoom('Stage')

        if not achievements['10K Fighter'] and score >= 10000 and device = 'Fighter' then
            achievements['10K Fighter'] = true
            -- Do whatever else that should be done when an achievement is unlocked
        end
    end)
end
```

As you can see it's a very small amount of code. The only thing we have to make sure is that each achievement only gets triggered once, and we do that by checking to see if that achievement has already been unlocked or not first. If it hasn't then we proceed.

I don't know how Steam's achievement system work yet but I'm assuming that we can call some function or set of functions to unlock an achievement for the player. If this is the case then we would call this function here as we set `achievements['10K Fighter']` to true. One last thing to remember is that achievements need to be saved and loaded, so it's important to add the appropriate code back in the `save` and `load` functions.

<br>

### Shaders

In the game so far I've been using about 3 shaders and we'll cover only one. However since the others use the same "framework" they can be applied to the screen in a similar way, even though the contents of each shader varies a lot. Also, I'm not a shaderlord so certainly I'm doing lots of very dumb things and there are better ways of doing all that I'm about to say. Learning shaders was probably the hardest part of game development for me and I'm still not comfortable enough with them to the extend that I am with the rest of my codebase. 

With all that said, we'll implement a simple RGB shift shader and apply it only to a few select entities in the game. The basic way in which pixel shaders work is that we'll write some code and this code will be applied to all pixels in the texture passed into the shader. You can read more about the basics [here](https://love2d.org/wiki/Shader).

One of the problems that I found when trying to apply this pixel shader to different objects in the game is that you can't apply it directly in that object's code. For whatever reason (and someone who knows more would be able to give you the exact reason here), pixel shaders aren't applied properly whenever we use basic primitives like lines, rectangles and so on. And even if we were using sprites instead of basic shapes, the RGB shift shader wouldn't be applied in the way we want either because the effect requires us to go outside the sprite boundaries. But because the pixel shader is only applied to pixels in the texture, when we try to apply it it will only read pixels inside the sprite's boundary so our effect doesn't work.

To solve this I've defaulted to drawing the objects that I want to apply effect X to to a new canvas, and then applying the pixel shader to that entire canvas. In a game like this where the order of drawing doesn't really matter this has almost no drawbacks. However in a game where the order of drawing matters more (like a 2.5D top-downish game) doing this gets a bit more complicated, so it's not a general solution for anything.

<br>

#### rgb_shift.frag

Before we get into coding all this let's get the actual pixel shader out of the way, since it's very simple:

```lua
extern vec2 amount;
vec4 effect(vec4 color, Image texture, vec2 tc, vec2 pc) {
    return color*vec4(Texel(texture, tc - amount).r, Texel(texture, tc).g, 
    Texel(texture, tc + amount).b, Texel(texture, tc).a);
}
```

I place this in a file called `rgb_shift.frag` in `resources/shaders` and loaded it in the `Stage` room using [`love.graphics.newShader`](https://love2d.org/wiki/love.graphics.newShader). The entry point for all pixel shaders is the `effect` function. This function receives a `color` vector, which is the one set with `love.graphics.setColor`, except that instead of being in 0-255 range, it's in 0-1 range. So if the current color is set to 255, 255, 255, 255, then this vec4 will have values 1.0, 1.0, 1.0, 1.0. The second thing it receives is a `texture` to apply the shader to. This texture can be a canvas, a sprite, or essentially any object in LÖVE that is drawable. The pixel shader will automatically go over all pixels in this texture and apply the code inside the `effect` function to each pixel, substituting its pixel value for the value returned. Pixel values are always vec4 objects, for the 4 red, green, blue and alpha components.

The third argument `tc` represents the texture coordinate. Texture coordinates range from 0 to 1 and represent the position of the current pixel inside the pixel. The top-left corner is `0, 0` while the bottom-right corner is `1, 1`. We'll use this along with the [`texture2D`](http://www.shaderific.com/glsl-functions/) function (which in LÖVE is called `Texel`) to get the contents of the current pixel. The fourth argument `pc` represents the pixel coordinate in screen space. We won't use this for this shader.

Finally, the last thing we need to know before getting into the effect function is that we can pass values to the shader to manipulate it in some way. In this case we're passing a vec2 called `amount` which will control the size of the RGB shift effect. Values can be passed in with the [`send`](https://love2d.org/wiki/Shader:send) function.

Now, the single line that makes up the entire effect looks like this:

```lua
return color*vec4(
    Texel(texture, tc - amount).r, 
    Texel(texture, tc).g, 
    Texel(texture, tc + amount).b, 
    Texel(texture, tc).a);
```

What we're doing here is using the `Texel` function to look up pixels. But we don't wanna look up the pixel in the current position only, we also want to look for pixels in neighboring positions so that we can actually to the RGB shifting. This effect works by shifting different channels (in this case red and blue) in different directions, which gives everything a glitchy look. So what we're doing is essentially looking up the pixel in position `tc - amount` and `tc + amount`, and then taking and red and blue value of that pixel, along with the green value of the original pixel and outputting it. We could have a slight optimization here since we're grabbing the same position twice (on the green and alpha components) but for something this simple it doesn't matter.

<br>

#### Selective drawing

Since we want to apply this pixel shader only to a few specific entities, we need to figure out a way to only draw specific entities. The easiest way to do this is to mark each entity with a tag, and then create an alternate `draw` function in the `Area` object that will only draw objects with that tag. Defining a tag looks like this:

```lua
function TrailParticle:new(area, x, y, opts)
    TrailParticle.super.new(self, area, x, y, opts)
    self.graphics_types = {'rgb_shift'}
    ...
end
```

And then creating a new draw function that will only draw objects with certain tags in them looks like this:

```lua
function Area:drawOnly(types)
    table.sort(self.game_objects, function(a, b) 
        if a.depth == b.depth then return a.creation_time < b.creation_time
        else return a.depth < b.depth end
    end)

    for _, game_object in ipairs(self.game_objects) do 
        if game_object.graphics_types then
            if #fn.intersection(types, game_object.graphics_types) > 0 then
                game_object:draw() 
            end
        end
    end
end
```

 So this is exactly like that the normal `Area:draw` function except with some additional logic. We're using the [`intersection`](https://github.com/Yonaba/Moses/blob/master/doc/tutorial.md#intersection-array-) to figure out if there are any common elements between the objects `graphics_types` table and the `types` table that we pass in. For instance, if we decide we only wanna draw `rgb_shift` type objects, then we'll call `area:drawOnly({'rgb_shift'})`, and so this table we passed in will be checked against each object's `graphics_types`. If they have any similar elements between them then `#fn.intersection` will be bigger than 0, which means we can draw the object.

Similarly, we will want to implement an `Area:drawExcept` function, since whenever we draw an object to one canvas we don't wanna draw it again in another, which means we'll need to exclude certain types of objects from drawing at some point. That looks like this:

```lua
function Area:drawExcept(types)
    table.sort(self.game_objects, function(a, b) 
        if a.depth == b.depth then return a.creation_time < b.creation_time
        else return a.depth < b.depth end
    end)

    for _, game_object in ipairs(self.game_objects) do 
        if not game_object.graphics_types then game_object:draw() 
        else
            if #fn.intersection(types, game_object.graphics_types) == 0 then
                game_object:draw()
            end
        end
    end
end
```

So here we draw the object if it doesn't have `graphics_types` defined, as well as if its intersection with the `types` table is 0, which means that its graphics type isn't one of the ones specified by the caller.

<br>

#### Canvases + shaders

With all this in mind now we can actually implement the effect. For now we'll just implement this on the `TrailParticle` object, which means that the trail that the player and projectiles creates will be RGB shifted. The main way in which we can apply the RGB shift only to objects like TrailParticle looks like this:

```lua
function Stage:draw()
    ...
    love.graphics.setCanvas(self.rgb_shift_canvas)
    love.graphics.clear()
    	camera:attach(0, 0, gw, gh)
    	self.area:drawOnly({'rgb_shift'})
    	camera:detach()
    love.graphics.setCanvas()
    ...
end
```

This looks similar to how we draw things normally, except that now instead of drawing to `main_canvas`, we're drawing to the newly created `rgb_shift_canvas`. And more importantly we're only drawing objects that have the `'rgb_shift'` tag. In this way this canvas will contain all the objects we need so that we can apply our pixel shaders to later. I use a similar idea for drawing [`Shockwave`](http://kpulv.com/309/Dev_Log__Shader_Follow_Up/) and [`Downwell`](https://github.com/SSYGEN/blog/issues/9) effects.

Once we're done with drawing to all our individual effect canvases, we can draw the main game to `main_canvas` with the exception of the things we already drew in other canvases. So that would look like this:

```lua
function Stage:draw()
    ...
    love.graphics.setCanvas(self.main_canvas)
    love.graphics.clear()
        camera:attach(0, 0, gw, gh)
        self.area:drawExcept({'rgb_shift'})
        camera:detach()
	love.graphics.setCanvas()
  	...
end
```

And then finally we can apply the effects we want. We'll do this by drawing the `rgb_shift_canvas` to another canvas called `final_canvas`, but this time applying the RGB shift pixel shader. This looks like this:

```lua
function Stage:draw()
    ...
    love.graphics.setCanvas(self.final_canvas)
    love.graphics.clear()
        love.graphics.setColor(255, 255, 255)
        love.graphics.setBlendMode("alpha", "premultiplied")
  
        self.rgb_shift:send('amount', {
      	random(-self.rgb_shift_mag, self.rgb_shift_mag)/gw, 
      	random(-self.rgb_shift_mag, self.rgb_shift_mag)/gh})
        love.graphics.setShader(self.rgb_shift)
        love.graphics.draw(self.rgb_shift_canvas, 0, 0, 0, 1, 1)
        love.graphics.setShader()
  
  	love.graphics.draw(self.main_canvas, 0, 0, 0, 1, 1)
  	love.graphics.setBlendMode("alpha")
  	love.graphics.setCanvas()
  	...
end
```

Using the `send` function we can change the value of the `amount` variable to correspond to the amount of shifting we want the shader to apply. Because the texture coordinates inside the pixel shader are between values 0 and 1, we want to divide the amounts we pass in by `gw` and `gh`. So, for instance, if we want a shift of 2 pixels then `rgb_shift_mag` will be 2, but the value passed in will be 2/gw and 2/gh, since inside the pixel shader, 2 pixels to the left/right is represented by that small value instead of actually 2. We also draw the main canvas to the final canvas, since the final canvas should contain everything that we want to draw.

Finally outside this we can draw this final canvas to the screen:

```lua
function Stage:draw()
    ...
    love.graphics.setColor(255, 255, 255)
    love.graphics.setBlendMode("alpha", "premultiplied")
    love.graphics.draw(self.final_canvas, 0, 0, 0, sx, sy)
    love.graphics.setBlendMode("alpha")
    love.graphics.setShader()
end
```

We could have drawn everything directly to the screen instead of to the `final_canvas` first, but if we wanted to apply another screen-wide shader to the final screen, like for instance the [`distortion`](https://www.shadertoy.com/view/ldXGW4), then it's easier to do that if everything is contained in a canvas properly. 

And so all that would end up looking like this:

<p align="center">
<img src="https://vgy.me/x7PmTc.gif">
</p>

And as expected, the trail alone is being RGB shifted and looks kinda glitchly like we wanted.

<br>

### Audio

I'm not really big on audio so while there are lots of very interesting and complicated things one could do, I'm going to stick to what I know, which is just playing sounds whenever appropriate. We can do this by using [ripple](https://github.com/tesselode/ripple). 

This library has a pretty simple API and essentially it boils down to loading sounds using `ripple.newSound` and playing those sounds by calling `:play` on the returned object. For instance, if we want to play a shooting sound whenever the player shoots, we could do something like this:

```lua
-- in globals.lua
shoot_sound = ripple.newSound('resources/sounds/shoot.ogg')
```

```lua
function Player:shoot()
    local d = 1.2*self.w
    self.area:addGameObject('ShootEffect', ...
    shoot_sound:play()
    ...
end
```

And so in this very simple way we can just call `:play` whenever we want a sound to happen. The library also has additional goodies like changing the pitch of the sound, playing sounds in a loop, creating tags so that you can change properties of all sounds with a certain tag, and so on. In the actual game I ended up doing some additional stuff on top of this, but I'm not going to go over all that here. If you've bought the tutorial you can see all that in the `sound.lua` file.

<br>

## END

And this is the end of this tutorial. By no means have we covered literally everything that we could have covered about this game but we went over the most important parts. If you followed along until now you should have a good grasp on the codebase so that you can understand most of it, and if you bought the tutorial then you should be able to read the full source code with a much better understanding of what's actually happening there.

Hopefully this tutorial has been helpful so that you can get some idea of what making a game actually entails and how to go from zero to the final result. Ideally now that you have all this done you should use what you learned from this to make your own game instead of just changing this one, since that's a much better exercise that will test your "starting from zero" abilities. Usually when I start a new project I pretty much copypaste a bunch of code that I know has been useful between multiple projects, generally that's a lot of the "engine" code that we went over in articles 1 through 5.

Anyway, I don't know how to end this so... bye!

<br>

---

If you liked these tutorials and want to support the writing of more like these in the future:

* ### [BYTEPATH on Steam](http://store.steampowered.com/app/760330/BYTEPATH/)
* ### [BYTEPATH-tutorial on gumroad](https://gumroad.com/l/BYTEPATH-tutorial)

Buying the tutorial gives you access to the game's full source code, answers to exercises from articles 1 through 9, the code separated by articles (what the code should look like at the end of each article) and a Steam key to the game.
