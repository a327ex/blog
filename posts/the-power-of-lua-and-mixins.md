`2014-01-07 06:30`

This post deals with OOP and Lua, and since Lua has no OOP built-in by default I'll make use of a library. [kikito](https://github.com/kikito) has a [great OOP library](https://github.com/kikito/middleclass) made for Lua that supports [mixins](https://github.com/kikito/middleclass/wiki/Mixins). [rxi](https://github.com/rxi) also has [one](https://github.com/rxi/classic) (it's the one I use now). Either way, as stated in one of those links: *"Mixins can be used for sharing methods between classes, without requiring them to inherit from the same father"*. If you've read anything about component based system design, then you've read something like that quote but probably without the word mixin anywhere near it. Since I've been using mixins as the foundation of the entity code in Kara, in this post I'll explain their advantages and what you can possibly get out of them.

<p align="center">
<img src="https://i.imgur.com/2EryAGF.gif"/>
</p>

<br>

## Objects, attributes and methods in Lua

The one thing to remember about Lua is that everything in it is a table, except the things that aren't. But pretty much all objects are going to be made out of tables (essentially hash tables) that have attribute and method names as keys and attribute values and functions as values. So, for instance, this:

```lua
Entity = class('Entity')
  
-- This is the constructor!
function Entity:init(x, y)
    self.x = x
    self.y = y
end
  
object = Entity(300, 400)
```

... is, for all that matters to this post, the same as having a table named `object` with keys `init`, `x` and `y`:

```lua
-- in Lua, table.something is the same as table["something"]
-- so basically accessing a key in a table looks like accessing 
-- an attribute of an object
  
object = {}
object["init"] = function(x, y)
    object["x"] = x
    object.y = y
end
  
object.init(300, 400)
```

<br>

## Mixins

If you've looked at [this mixins explanation](https://github.com/kikito/middleclass/wiki/Mixins) it should be pretty clear what's going on. In case it isn't: an `include` call simply adds the defined functions of a certain mixin to a certain class, which means adding a new key to the table that is the object. In a similar fashion, a function that changes an object's attributes can be defined:

``` lua
Mixin = {
    mixinFunctionInit = function(self, x, y)
        self.x = x
        self.y = y
    end
}
  
Entity = class('Entity')
Entity:include(Mixin)
  
function Entity:init(x, y)
    self.mixinFunctionInit(self, x, y)
    -- here self:mixinFunctionInit(x, y) could also have been used
    -- since in Lua : is shorthand for calling a function while 
    -- passing self as the first argument
end
  
object = Entity(300, 400)
```

This example is exactly the same as the first one up there, except that instead of directly setting the `x` and `y` attributes, the mixin does it. The `include` call adds the `mixinFunctionInit` function to the class, then calling that function makes changing the object's attributes possible (by passing self, a reference to the object being modified). It's a very easy and cheap way of getting some flexibility into your objects. That flexibility can get you the following:

<br>

## Reusability

Component based systems were mentioned, and mixins are a way of getting there. For instance, these are the initial calls for two classes in Kara, the Hammer and a Blaster (an enemy):

```lua
Hammer = class('Hammer', Entity)
Hammer:include(PhysicsRectangle)
Hammer:include(Timer)
Hammer:include(Visual)
Hammer:include(Steerable)
...
  
Blaster = class('Blaster', Entity)
Blaster:include(Timer)
Blaster:include(PhysicsRectangle)
Blaster:include(Steerable)
Blaster:include(Stats)
Blaster:include(HittableRed)
Blaster:include(EnemyExploder)
...
```

Notice how the `PhysicsRectangle`, `Timer` and `Steerable` mixins repeat themselves? That's reusability in its purest form. `PhysicsRectangle` is a simple wrapper mixin for LÖVE's box2d implementation, and, as you may guess, makes it so that the object becomes a physics object of rectangular shape. The `Timer` mixin implements a wrapper for the main Timer class, containing calls like `after` (do something after `n` seconds) or `every` (do something every `n` seconds). A bit off-topic, but since you can pass functions as arguments in Lua, timers are pretty nifty and have pretty simple calls that look like this:

```lua
-- somewhere in the code of a class that has the Timer mixin
-- after 2 seconds explode this object by doing everything needed in an explosion
self.timer:tween(2, self, {tint_color = {255, 255, 255}}, 'in-out-cubic')
self.timer:after(2, function()
    self.dead = true
    self:createExplosionParticles()
    self:createDeathParticles()
    self:dealAreaDamage({radius = 10, damage = 50})
    self.world:cameraShake({intensity = 5, duration = 1})
end)
```

Anyway, lastly, the `Steerable` mixin implements steering behaviors, which are used to control how far and in which way something moves towards a target. With a bit of work steering behaviors can be used in all sorts of ways to achieve basic entity behaviors while allowing for the possibility of more complex ones later on. In any case, those three mixins were reused with changes only to the variables passed in on their initialization and sometimes on their update function. Otherwise, it's all the same code that can be used by multiple classes. Consider the alternative, where reuse would have to rely on inheritance and somehow Hammer and Blaster would have to be connected in some weird way, even though they're completely different things (one is something the Player uses as a tool of attack, the other is an Enemy).

<p align="center">
<img src="https://i.imgur.com/k4tirzV.gif"/>
</p>

And consider the other alternative, which is the normal component based system. It usually uses some sort of message passing or another mechanism to get variables from other components in
the same object (kinda yucky!). While the mixin based system simply changes those attributes directly, because when you're coding a mixin you always have a reference to the object you're 
coding to via the `self` parameter.

<br>

## Object Creation

The mutability offered by mixins (in this case Lua is also important) also helps when you're creating objects. For instance, this is the single call that I need to create new objects in my game:

```lua
create = function(self, type, x, y, settings)
    table.insert(self.to_be_created, 
                {type = type, x = x, y = y, settings = settings})
end
  
-- Example:
enemy = create('Enemy', 400, 300, {velocity = 200, hp = 5, damage = 2}) 
```

And then this gets called every frame to create the objects inside the `to_be_created` list (this must be done because I can't create objects while box2d runs its own update cycle,
and it usually happens that the `create` function is called precisely while box2d is running):

```lua
createPostWorldStep = function(self)
    for _, o in ipairs(self.to_be_created) do
        local entity = nil
        if o.type then entity = _G[o.type](self, o.x, o.y, o.settings) end
        self:add(entity)
    end
    self.to_be_created = {}
end
```

Essentially, whenever I create a class like this: `"Entity = class('Entity')"` I'm actually creating a global variable called Entity that holds the Entity class definition (it's the table that is used as a prototype for creating Entity instances). Since Lua's global state is inside a table (and that table is called `_G`), I can access the class by going through that global table and then just calling its constructor with the appropriate parameters passed. And the clevererest part of this is actually the `settings` table. All entities in Kara derive from an Entity class, which looks like this:

```lua
Entity = class('Entity')
  
function Entity:init(world, x, y, settings)
    self.id = getUID()
    self.dead = false
    self.world = world
    self.x = x
    self.y = y
    if settings then
        for k, v in pairs(settings) do self[k] = v end
    end
end
```

The last few lines being the most important ones, as the attributes of the class are changed according to the settings table. So whenever an object is created you get a choice to add whatever attributes (or even functions) you want to an object, which adds a lot of flexibility, since otherwise you'd have to create many different classes (or add various attributes to the same class) for all different sorts of random properties that can happen in a game.

<br>

## Readability

Reusable code is locked inside mixins and entity specific code is clearly laid out on that entity's file. This is a huge advantage whenever you're reading some entity's code because you never get lost into what is relevant and what isn't. For example, look at the code for the smoke/dust class:

```lua
Dust = class('Dust', Entity)
Dust:include(PhysicsRectangle)
Dust:include(Timer)
Dust:include(Fader)
Dust:include(Visual)
  
function Dust:init(world, x, y, settings)
    Entity.init(self, world, x, y, settings)
    self:physicsRectangleInit(self.world.world, x, y, 'dynamic', 16, 16)
    self:timerInit()
    self:faderInit(self.fade_in_time, 0, 160)
    self:visualInit(dust, Vector(0, 0)) 
  
    self.scale = math.prandom(0.5, 1)
    self.timer:tween(2, self, {scale = 0.25}, 'in-out-cubic')
    self.body:setGravityScale(math.prandom(-0.025, 0.025))
    self.body:setAngularVelocity(math.prandom(math.pi/8, math.pi/2))
    local r = math.prandom(0, 2*math.pi)
    local v = math.prandom(5, 10)
    self.body:setLinearVelocity(v*math.cos(r), v*math.sin(r))
    self:fadeIn(0, self.fade_in_time or 1, 'in-out-cubic')
    self:fadeOut(self.fade_in_time or 1, 1, 'in-out-cubic')
end
  
function Dust:update(dt)
    self.x, self.y = self.body:getPosition()
    self:timerUpdate(dt)
end
  
function Dust:draw()
    if debug_draw then self:physicsRectangleDraw() end
    self:faderDraw()
    self:visualDraw()
end
```

The entity specific code here happens in the constructor after the `visualInit` call. If I ever look back at this in another month, year or decade I will know that if I want to change the smoke's behavior I need to change something in that block of entity specific code. If I want to change how to smoke looks I have to change the first parameter in the `visualInit` call. If I have to change something in how the fading in or out of the effect works, I have to change something in the `faderInit` call. Everything is very properly divided and hidden and there's a clear way of knowing where everything is and what everything does.

<p align="center">
<img src="https://i.imgur.com/66uaX9E.gif"/>
</p>

<br>

## ENDING

Hopefully if you could understand any of that you can see why Lua is great and why mixins are the best. There are certainly drawbacks to using this, but **if you're coding by yourself or with only another programmer** then the rules don't matter because you can do anything you want. You are a shining star and the world is only waiting for you to shine your bright beautiful light upon it.
