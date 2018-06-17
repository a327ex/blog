## Introduction

In this article we'll go over the implementation of all passives in the game. There are a total of about 120 different things we will implement and those are enough to be turned into a very big skill tree (the tree I made has about 900 nodes, for instance). 

This article will be filled with exercises tagged as content, and the way that will work is that I'll show you how to do something, and then give you a bunch of exercises to do that same thing but applying it to other stats. For instance, I will show you how to implement an HP multiplier, which is a stat that will multiply the HP of the player by a certain percentage, and then the exercises will ask for you to implement Ammo and Boost multipliers. In reality things will get a bit more complicated than that but this is the basic idea.

After we're done with the implementation of everything in this article we'll have pretty much have most of the game's content implemented and then it's a matter of finishing up small details, like building the huge skill tree out of the passives we implemented. :-D 

<br>

## Types of Stats

Before we start with the implementation of everything we need to first decide what kinds of passives our game will have. I already decided what I wanted to do so I'm going to just follow that, but you're free to deviate from this and come up with your own ideas. 

The game will have three main types of passive values: resources, stat multipliers and chances. 

*   Resources are HP, Boost and Ammo. These are values that are described by a `max_value` variable as well as a `current_value` variable. In the case of HP we have the maximum HP the player has, and then we also have the current amount.
*   Stat multipliers are multipliers that are applied to various values around the game. As the player goes around the tree picking up nodes, he'll be picking up stuff like "10% Increased Movement Speed", and so after he does that and starts a new match, we'll take all the nodes the player picked, pack them into those multiplier values, and then apply them in the game. So if the player picked nodes that amounted to 50% increased movement speed, then the movement speed multiplier will be applied to the `max_v` variable, so some `mvspd_multiplier`  variable will be 1.5 and our maximum velocity will be multiplied by 1.5 (which is a 50% increase).
*   Chances are exactly that, chances for some event to happen. The player will also be able to pick up added chance for certain events to happen in different circumstances. For instance, "5% Added Chance to Barrage on Cycle", which means that whenever a cycle ends (the 5 second period we implemented), there's a 5% chance for the player to launch a barrage of projectiles. If the player picks up tons of those nodes then the chance gets higher and a barrage happens more frequently. 

The game will have an additional type of node and an additional mechanic: notable nodes and temporary buffs.

*   Notable nodes are nodes that change the logic of the game in some way (although not always). For instance, there's a node that replaces your HP for Energy Shield. And with ES you take double damage, your ES recharges after you don't take damage for a while, and you have halved invulnerability time. Nodes like these are not as numerous as others but they can be very powerful and combined in fun ways.
*   Temporary buffs are temporary boosts to your stats. Sometimes you'll get a temporary buff that, say,  increases your attack speed by 50% for 4 seconds.

----

Knowing all this we can get started. To recap, the current resource stats we have in our codebase should look like this:

```lua
function Player:new(...)
    ...
  
    -- Boost
    self.max_boost = 100
    self.boost = self.max_boost
    ...

    -- HP
    self.max_hp = 100
    self.hp = self.max_hp

    -- Ammo
    self.max_ammo = 100
    self.ammo = self.max_ammo
  
    ...
end
```

The movement code values should look like this:

```lua
function Player:new(...)
    ...
  	
    -- Movement
    self.r = -math.pi/2
    self.rv = 1.66*math.pi
    self.v = 0
    self.base_max_v = 100
    self.max_v = self.base_max_v
    self.a = 100
	
    ...
end
```

And the cycle values should look like this (I renamed all previous references to the word "tick" to be "cycle" now for consistency):

```lua
function Player:new(...)
    ...
  
    -- Cycle
    self.cycle_timer = 0
    self.cycle_cooldown = 5

    ...
end
```

<br>

### HP multiplier

So let's start with the HP multiplier. In a basic way all we have to do is define a variable named `hp_multiplier` that starts as the value 1, and then we apply the increases from the tree to this variable and  multiply it by `max_hp` at some point. So let's do the first thing:

```lua
function Player:new(...)
    ...
  	
    -- Multipliers
    self.hp_multiplier = 1
end
```

Now the second thing is that we have to assume we're getting increases to HP from the tree. To do this we have to assume how these increases will be passed in and how they'll be defined. Here I have to cheat a little (since I already wrote the game once) and say that the tree nodes will be defined in the following format:

```lua
tree[2] = {'HP', {'6% Increased HP', 'hp_multiplier', 0.06}}
```

This means that node #2 is named `HP`, has as its description `6% Increased HP`, and affects the variable `hp_multiplier` by 0.06 (6%). There is a function named `treeToPlayer` which takes all 900~ of those node definitions and then applies them to the player object. It's important to note that the variable name used in the node definition has to be the same name as the one defined in the player object, otherwise things won't work out. This is a very thinly linked and error-prone method of doing it I think, but as I said in the previous article it's the kind of thing you can get away with because you're coding by yourself.

Now the final question is: when do we multiply `hp_multiplier` by `max_hp`? The natural option here is to just do it on the constructor, since that's when a new player is created, and a new player is created whenever a new Stage room is created, which is also when a new match starts. However, we'll do this at the very end of the constructor, after all resources, multipliers and chances have been defined:

```lua
function Player:new(...)
    ...
  
    -- treeToPlayer(self)
    self:setStats()
end
```

And so in the `setStats` function we can do this:

```lua
function Player:setStats()
    self.max_hp = self.max_hp*self.hp_multiplier
    self.hp = self.max_hp
end
```

And so if you set `hp_multiplier` to 1.5 for instance and run the game, you'll notice that now the player will have 150 HP instead of its default 100. 

Note that we also have to assume the existence of the `treeToPlayer` function here and pass the player object to that function. Eventually when we write the skill tree code and implement that function, what it will do is set the values of all multipliers based on the bonuses from the tree, and then after those values are set we can call `setStats` to use those to change the stats of the player. 

<br>

**123. (CONTENT)** Implement the `ammo_multiplier` variable.

**124. (CONTENT)** Implement the `boost_multiplier` variable.

<br>

### Flat HP

Now for a flat stat. Flat stats are direct increases to some stat instead of a percentage based one. The way we'll do it for HP is by defining a `flat_hp` variable which will get added to `max_hp` (before being multiplied by the multiplier):

```lua
function Player:new(...)
    ...
  	
    -- Flats
    self.flat_hp = 0
end
```

```lua
function Player:setStats()
    self.max_hp = (self.max_hp + self.flat_hp)*self.hp_multiplier
    self.hp = self.max_hp
end
```

Like before, whenever we define a node in the tree we want to link it to the relevant variable, so, for instance, a node that adds flat HP could look like this:

```lua
tree[15] = {'Flat HP', {'+10 Max HP', 'flat_hp', 10}}
```

<br>

**125. (CONTENT)** Implement the `flat_ammo` variable.

**126. (CONTENT)** Implement the `flat_boost` variable.

**127. (CONTENT)** Implement the `ammo_gain` variable, which adds to the amount of ammo gained when the player picks one up. Change the calculations in the `addAmmo` function accordingly.

<br>

### Homing Projectile

The next passive we'll implement is "Chance to Launch Homing Projectile on Ammo Pickup", but for now we'll focus on the homing projectile part. One of the attacks the player will have is a homing projectile so we'll just implement that as well now.

A projectile will have its homing function activated whenever the `attack` attribute is set to `'Homing'`. The code that actually does the homing will be the same as the code we used for the Ammo resource:

```lua
function Projectile:update(dt)
    ...
    self.collider:setLinearVelocity(self.v*math.cos(self.r), self.v*math.sin(self.r))

    -- Homing
    if self.attack == 'Homing' then
    	-- Move towards target
        if self.target then
            local projectile_heading = Vector(self.collider:getLinearVelocity()):normalized()
            local angle = math.atan2(self.target.y - self.y, self.target.x - self.x)
            local to_target_heading = Vector(math.cos(angle), math.sin(angle)):normalized()
            local final_heading = (projectile_heading + 0.1*to_target_heading):normalized()
            self.collider:setLinearVelocity(self.v*final_heading.x, self.v*final_heading.y)
        end
    end
end
```

The only thing we have to do differently is defining the `target` variable. For the Ammo object the `target` variable points to the player object, but in the case of a projectile it should point to a nearby enemy. To get a nearby enemy we can use the `getAllGameObjectsThat` function that is defined in the Area class, and use a filter that will only select objects that are enemies and that are close enough. To do this we must first define what objects are enemies and what objects aren't enemies, and the easiest way to do that is to just have a global table called `enemies` which will contain a list of strings with the name of the enemy classes. So in `globals.lua` we can add the following definition:

```lua
enemies = {'Rock', 'Shooter'}
```

And as we add more enemies into the game we also add their string to this table accordingly. Now that we know which object types are enemies we can easily select them:

```lua
local targets = self.area:getAllGameObjectsThat(function(e)
    for _, enemy in ipairs(enemies) do
    	if e:is(_G[enemy]) then
            return true
        end
    end
end)
```

We use the `_G[enemy]` line to access the class definition of the current string we're looping over. So `_G['Rock']` will return the table that contains the class definition of the `Rock` class. We went over this in multiple articles so it should be clear by now why this works.

Now for the other condition we want to select only enemies that are within a certain radius of this projectile. Through trial and error I came to a radius of about 400 units, which is not small enough that the projectile will never have a proper target, but not big enough that the projectile will try to hit offscreen enemies too much:

```lua
local targets = self.area:getAllGameObjectsThat(function(e)
    for _, enemy in ipairs(enemies) do
    	if e:is(_G[enemy]) and (distance(e.x, e.y, self.x, self.y) < 400) then
            return true
        end
    end
end)
```

`distance` is a function we can define in `utils.lua` which returns the distance between two positions:

```lua
function distance(x1, y1, x2, y2)
    return math.sqrt((x1 - x2)*(x1 - x2) + (y1 - y2)*(y1 - y2))
end
```

And so after this we should have our enemies in the `targets` list. After that all we want to do is get a random one of them and point that as the `target` that the projectile will move towards:

```lua
self.target = table.remove(targets, love.math.random(1, #targets))
```

And all that should look like this:

```lua
function Projectile:update(dt)
    ...

    self.collider:setLinearVelocity(self.v*math.cos(self.r), self.v*math.sin(self.r))

    -- Homing
    if self.attack == 'Homing' then
        -- Acquire new target
        if not self.target then
            local targets = self.area:getAllGameObjectsThat(function(e)
                for _, enemy in ipairs(enemies) do
                    if e:is(_G[enemy]) and (distance(e.x, e.y, self.x, self.y) < 400) then
                        return true
                    end
                end
            end)
            self.target = table.remove(targets, love.math.random(1, #targets))
        end
        if self.target and self.target.dead then self.target = nil end

        -- Move towards target
        if self.target then
            local projectile_heading = Vector(self.collider:getLinearVelocity()):normalized()
            local angle = math.atan2(self.target.y - self.y, self.target.x - self.x)
            local to_target_heading = Vector(math.cos(angle), math.sin(angle)):normalized()
            local final_heading = (projectile_heading + 0.1*to_target_heading):normalized()
            self.collider:setLinearVelocity(self.v*final_heading.x, self.v*final_heading.y)
        end
    end
end
```

There's an additional line at the end of the block where we acquire a new target, where we set `self.target` to nil in case the target has been killed. This makes it so that whenever the target for this projectile stops existing, `self.target` will be set to nil and a new target will be acquired, since the condition `not self.target` will be met and then the whole process will repeat itself. It's also important to mention that once a target has been acquired we don't do any more calculations, so there's no big need to worry about the performance of `getAllGameObjectsThat`, which is a function that naively loops over all objects currently alive in the game.

One extra thing we have to do is change how the projectile object behaves whenever it's not homing or whenever there's no target. Intuitively using `setLinearVelocity` first to set the projectile's velocity once, and then using it again inside the `if self.attack == 'Homing'` loop would make sense, since the velocity would only be changed if the projectile is in fact homing and if a target exists. But for some reason doing that results in all sorts of problems, so we have to make sure we only call `setLinearVelocity` once, and that implies something like this:

```lua
-- Homing
if self.attack == 'Homing' then
    ...
-- Normal movement
else self.collider:setLinearVelocity(self.v*math.cos(self.r), self.v*math.sin(self.r)) end
```

This is a bit more confusing than the previous setup but it works. And if we test all this and create a projectile with the `attack` attribute set to `'Homing'` it should look like this:

<p align="center">
<img src="https://vgy.me/gGMT23.gif">
</p>

<br>

**128. (CONTENT)** Implement the `Homing` attack. Its definition on the attacks table looks like this:

```lua
attacks['Homing'] = {cooldown = 0.56, ammo = 4, abbreviation = 'H', color = skill_point_color}
```

And the attack itself looks like this:

<p align="center">
<img src="https://vgy.me/s10lpx.gif">
</p>

Note that the projectile for this attack (as well as others that are to come) is slightly different. It's a rhombus half colored as white and half colored as the color of the attack (in this case `skill_point_color`), and it also has a trail that's the same as the player's.

<br>

### Chance to Launch Homing Projectile on Ammo Pickup

Now we can move on to what we wanted to implement, which is this chance-type passive. This one is has a chance to be triggered whenever we pick the Ammo resource up. We'll hold this chance in the `launch_homing_projectile_on_ammo_pickup_chance` variable and then whenever an Ammo resource is picked up, we'll call a function that will handle rolling the chances for this event to happen. 

But before we can do that we need to specify how we'll handle these chances. As I introduced in another article, here we'll also use the `chanceList` concept. If an event has 5% probability of happening, then we want to make sure that it will actually follow that 5% somewhat reasonably, and so it just makes sense to use chanceLists.

The way we'll do it is that after we call the `setStats` function on the Player's constructor, we'll also call a function called `generateChances` which will create all the chanceLists we'll use throughout the game. Since there will be lots and lots of different events that will need to be rolled we'll put all chanceLists into a table called `chances`, and organize things that so whenever we need to roll for a chance of something happening, we can do something like:

```lua
if self.chances.launch_homing_projectile_on_ammo_pickup_chance:next() then
    -- launch homing projectile
end
```

We could set up the `chances` table manually, so that every time we add a new `_chance` type variable that will hold the chances for some event to happen, we also add and generate its chanceList in the `generateChances` function. But we can be a bit clever here and decide that every variable that deals with chances will end with `_chance`, and then we can use that to our advantage:

```lua
function Player:generateChances()
    self.chances = {}
    for k, v in pairs(self) do
        if k:find('_chance') and type(v) == 'number' then
      
        end
    end
end
```

Here we're going through all key/value pairs inside the player object and returning true whenever we find an attribute that contains in its name the `_chance` substring, as well as being a number. If both those things are true then based on our own decision this is a variable that is dealing with chances of some event happening. So now all we have to do is then create the chanceList and add it to the `chances` table:

```lua
function Player:generateChances()
    self.chances = {}
    for k, v in pairs(self) do
        if k:find('_chance') and type(v) == 'number' then
      	    self.chances[k] = chanceList({true, math.ceil(v)}, {false, 100-math.ceil(v)})
      	end
    end
end
```

And so this will create a chanceList of 100 values, with `v` of them being true, and `100-v` of them being false. So if the only chance-type variable we had defined in our player object was the `launch_homing_projectile_on_ammo_pickup_chance` one, and this had the value 5 attached to it (meaning 5% probability of this event happening), then the chanceList would have 5 true values and 95 false ones, which gets us what we wanted.

And so if we call `generateChances` on the player's constructor:

```lua
function Player:new(...)
    ...
  
    -- treeToPlayer(self)
    self:setStats()
    self:generateChances()
end
```

Then everything should work fine. We can now define the `launch_homing_projectile_on_ammo_pickup_chance` variable:

```lua
function Player:new(...)
    ...
  	
    -- Chances
    self.launch_homing_projectile_on_ammo_pickup_chance = 0
end
```

And if you wanna test that the roll system works, you can set that to a value like 50 and then call `:next()` a few times to see what happens.

The implementation of the actual launching will happen through the `onAmmoPickup` function, which will be called whenever Ammo is picked up:

```lua
function Player:update(dt)
    ...
    if self.collider:enter('Collectable') then
        ...
    
        if object:is(Ammo) then
            object:die()
            self:addAmmo(5)
            self:onAmmoPickup()
      	...
    end
end
```

And that function then would look like this:

```lua
function Player:onAmmoPickup()
    if self.chances.launch_homing_projectile_on_ammo_pickup_chance:next() then
        local d = 1.2*self.w
        self.area:addGameObject('Projectile', 
      	self.x + d*math.cos(self.r), self.y + d*math.sin(self.r), 
      	{r = self.r, attack = 'Homing'})
        self.area:addGameObject('InfoText', self.x, self.y, {text = 'Homing Projectile!'})
    end
end
```

And then all that would end up looking like this:

<p align="center">
<img src="https://vgy.me/0o8919.gif">
</p>

<br>

**129. (CONTENT)** Implement the `regain_hp_on_ammo_pickup_chance` passive. The amount of HP regained is 25 and should be added with the `addHP` function, which adds the given amount of HP to the `hp` value, making sure that it doesn't go above `max_hp`. Additionally, an `InfoText` object should be created with the text `'HP Regain!'` in `hp_color`.

**130. (CONTENT)** Implement the `regain_hp_on_sp_pickup_chance` passive. he amount of HP regained is 25 and should be added with the `addHP` function. An `InfoText` object should be created with the text `'HP Regain!'` in `hp_color`. Additionally, an `onSPPickup` function should be added to the Player class and in it all this work should be done (like we did with the `onAmmoPickup` function).

<br>

### Haste Area

The next passives we want to implement are "Chance to Spawn Haste Area on HP Pickup" and "Chance to Spawn Haste Area on SP Pickup". We already know how to do the "on Resource Pickup" part, so now we'll focus on the "Haste Area". A haste area is simply a circle that boosts the player's attack speed whenever he is inside it. This boost in attack speed will be applied as a multiplier, so it makes sense for us to implement the attack speed multiplier first.

<br>

### ASPD multiplier

We can define an ASPD multiplier simply as the `aspd_multiplier` variable and then multiply this variable by  our shooting cooldown:

```lua
function Player:new(...)
    ...
  	
    -- Multipliers
    self.aspd_multiplier = 1
end
```

```lua
function Player:update(dt)
    ...
  
    -- Shoot
    self.shoot_timer = self.shoot_timer + dt
    if self.shoot_timer > self.shoot_cooldown*self.aspd_multiplier then
        self.shoot_timer = 0
        self:shoot()
    end
end
```

The main difference that this one multiplier in particular will have is that lower values are better than higher values. In general, if a multiplier value is 0.5 then it's cutting whatever stat it's being applied to by half. So for HP, movement speed and pretty much everything else this is a bad thing. However, for attack speed lower values are better, and this can be simply explained by the code above. Since we're applying the multiplier to the `shoot_cooldown` variable, lower values means that this cooldown will be lower, which means that the player will shoot faster. We'll use this knowledge next when creating the `HasteArea` object.

<br>

### Haste Area

And now that we have the ASPD multiplier we can get back to this. What we want to do here is to create a circular area that will decrease `aspd_multiplier` by some amount as long as the player is inside it. To achieve this we'll create a new object named `HasteArea` which will handle the logic of seeing if the player is inside it or not and setting the appropriate values in case he is. The basic structure of the object looks like this:

```lua
function HasteArea:new(...)
    ...
  
    self.r = random(64, 96)
    self.timer:after(4, function()
        self.timer:tween(0.25, self, {r = 0}, 'in-out-cubic', function() self.dead = true end)
    end)
end

function HasteArea:update(dt)
    ...
end

function HasteArea:draw()
    love.graphics.setColor(ammo_color)
    love.graphics.circle('line', self.x, self.y, self.r + random(-2, 2))
    love.graphics.setColor(default_color)
end
```

For the logic behind applying the actual effect we have to keep track of when the player enters/leaves the area and then modify the `aspd_multiplier` value once that happens. The way to do this looks something like this:

```lua
function HasteArea:update(dt)
    ...
  	
    local player = current_room.player
    if not player then return end
    local d = distance(self.x, self.y, player.x, player.y)
    if d < self.r and not player.inside_haste_area then -- Enter event
        player:enterHasteArea()
    elseif d >= self.r and player.inside_haste_area then -- Leave event
    	player:exitHasteArea()
    end
end
```

We use a variable called `inside_haste_area` to keep track of whether the player is inside the area or not. This variable is set to true inside `enterHasteArea` and set to false inside `exitHasteArea`, meaning that those functions will only be called once when those events happen from the `HasteArea` object. In the Player class, both functions simply will apply the modifications necessary:

```lua
function Player:enterHasteArea()
    self.inside_haste_area = true
    self.pre_haste_aspd_multiplier = self.aspd_multiplier
    self.aspd_multiplier = self.aspd_multiplier/2
end

function Player:exitHasteArea()
    self.inside_haste_area = false
    self.aspd_multiplier = self.pre_haste_aspd_multiplier
    self.pre_haste_aspd_multiplier = nil
end
```

And so in this way whenever the player enters the area his attack speed will be doubled, and whenever he exits the area it will go back to normal. One big point that's easy to miss here is that it's tempting to put all this logic inside the `HasteArea` object instead of linking it back to the player via the `inside_haste_area` variable. The reason why we can't do this is because if we do, then problems will occur whenever the player enters/leaves multiple areas. As it is right now, the fact that the `inside_haste_area` variable exists means that we will only apply the buff once, even if the player is standing on top of 3 overlapping HasteArea objects.

<br>

**131. (CONTENT)** Implement the `spawn_haste_area_on_hp_pickup_chance` passive. An `InfoText` object should be created with the text `'Haste Area!'`. Additionally, an `onHPPickup` function should be added to the Player class.

**132. (CONTENT)** Implement the `spawn_haste_area_on_sp_pickup_chance`passive. An `InfoText` object should be created with the text `'Haste Area!'`.

<br>

### Chance to Spawn SP on Cycle

The next one we'll go for is `spawn_sp_on_cycle_chance`. For this one we kinda already know how to do it in its entirety. The "onCycle" part behaves quite similarly to "onResourcePickup", the only difference is that we'll call the `onCycle` function whenever a new cycle occurs instead of whenever a resource is picked. And the "spawn SP" part is simply creating a new SP resource, which we also already know how to do.

So for the first part, we need to go into the `cycle` function and call `onCycle`:

```lua
function Player:cycle()
    ...
    self:onCycle()
end
```

Then we add the `spawn_sp_on_cycle_chance` variable to the Player:

```lua
function Player:new(...)
    ...
  	
    -- Chances
    self.spawn_sp_on_cycle_chance = 0
end
```

And with that we also automatically add a new chanceList representing the chances of this variable. And because of that we can add the functionality needed to the `onCycle` function:

```lua
function Player:onCycle()
    if self.chances.spawn_sp_on_cycle_chance:next() then
        self.area:addGameObject('SkillPoint')
        self.area:addGameObject('InfoText', self.x, self.y, 
      	{text = 'SP Spawn!', color = skill_point_color})
    end
end

```

And this should work out as expected:

<p align="center">
<img src="https://vgy.me/So2n82.gif">
</p>

<br>

### Chance to Barrage on Kill

The next one is `barrage_on_kill_chance`. The only thing we don't really know how to do here is the "Barrage" part. Triggering events on kill is similar to the previous one, except instead of whenever a cycle happens, we'll call the player's `onKill` function whenever an enemy dies.

So first we add the `barrage_on_kill_chance` variable to the Player:

```lua
function Player:new(...)
    ...
  	
    -- Chances
    self.barrage_on_kill_chance = 0
end
```

Then we create the `onKill` function and call it whenever an enemy dies. There are two approaches to calling `onKill` whenever an enemy dies. The first is to just call it from every enemy's `die` or `hit` function. The problem with this is that as we add new enemies we'll need to add this same code calling `onKill` to all of them. The other option is to call `onKill` whenever a Projectile object collides with an enemy. The problem with this is that some projectiles can collide with enemies but not kill them (because the enemies have more HP or the projectile deals less damage), and so we need to figure out a way to tell if the enemy is actually dead or not. It turns out that figuring that out is pretty easy, so that's what I'm gonna go with:

```lua
function Projectile:update(dt)
    ...
  	
    if self.collider:enter('Enemy') then
        ...

        if object then
            object:hit(self.damage)
            self:die()
            if object.hp <= 0 then current_room.player:onKill() end
        end
    end
end
```

So all we have to do is after we call the enemy's `hit` function is to simply check if the enemy's HP is 0 or not. If it is it means he's dead and so we can call `onKill`.

Now for the barrage itself. The way we'll code is that by default, 8 projectiles will be shot within 0.05 seconds of each other, with an angle of between -math.pi/8 and +math.pi/8 of the angle the player is pointing towards. The barrage projectiles will also have the attack that the player has. So if the player has homing projectiles, then all barrage projectiles will also be homing. All that translates to this:

```lua
function Player:onKill()
    if self.chances.barrage_on_kill_chance:next() then
        for i = 1, 8 do
            self.timer:after((i-1)*0.05, function()
                local random_angle = random(-math.pi/8, math.pi/8)
                local d = 2.2*self.w
                self.area:addGameObject('Projectile', 
            	self.x + d*math.cos(self.r + random_angle), 
            	self.y + d*math.sin(self.r + random_angle), 
            	{r = self.r + random_angle, attack = self.attack})
            end)
        end
        self.area:addGameObject('InfoText', self.x, self.y, {text = 'Barrage!!!'})
    end
end
```

Most of this should be pretty straightforward. The only notable thing is that we use `after` inside a for loop to separate the creation of projectiles by 0.05 seconds between each other. Other than that we simply create the projectile with the given constraints. All that should look like this:

<p align="center">
<img src="https://vgy.me/sHS5hc.gif">
</p>

<br>

For the next exercises (and every one that comes after them), don't forget to create `InfoText` objects with the appropriate colors so that the player can tell when something happened. 

**133. (CONTENT)** Implement the `spawn_hp_on_cycle_chance` passive. 

**134. (CONTENT)** Implement the `regain_hp_on_cycle_chance` passive. The amount of HP regained is 25.

**135. (CONTENT)** Implement the `regain_full_ammo_on_cycle_chance` passive. 

**136. (CONTENT)** Implement the `change_attack_on_cycle_chance` passive. The new attack is chosen at random.

**137. (CONTENT)** Implement the `spawn_haste_area_on_cycle_chance` passive.

**138. (CONTENT)** Implement the `barrage_on_cycle_chance` passive.

**139. (CONTENT)** Implement the `launch_homing_projectile_on_cycle_chance` passive.

**140. (CONTENT)** Implement the `regain_ammo_on_kill_chance` passive. The amount of ammo regained is 20.

**141. (CONTENT)** Implement the `launch_homing_projectile_on_kill_chance` passive.

**142. (CONTENT)** Implement the `regain_boost_on_kill_chance` passive. The amount of boost regained is 40.

**143. (CONTENT)** Implement the `spawn_boost_on_kill_chance` passive.

<br>

### Gain ASPD Boost on Kill

We already implemented an "ASPD Boost"-like passive before with the `HasteArea` object. Now we want to implement another where we have a chance to get an attack speed boost whenever we kill an enemy. However, if we try to implement this in the same way that we implement the previous ASPD boost we would soon encounter problems. To recap, this is how we implement the boost in `HasteArea`:

```lua
function HasteArea:update(dt)
    HasteArea.super.update(self, dt)

    local player = current_room.player
    if not player then return end
    local d = distance(self.x, self.y, player.x, player.y)
    if d < self.r and not player.inside_haste_area then player:enterHasteArea()
    elseif d >= self.r and player.inside_haste_area then player:exitHasteArea() end
end
```

And then `enterHasteArea` and `exitHasteArea` look like this:

```lua
function Player:enterHasteArea()
    self.inside_haste_area = true
    self.pre_haste_aspd_multiplier = self.aspd_multiplier
    self.aspd_multiplier = self.aspd_multiplier/2
end

function Player:exitHasteArea()
    self.inside_haste_area = false
    self.aspd_multiplier = self.pre_haste_aspd_multiplier
    self.pre_haste_aspd_multiplier = nil
end
```

If we tried to implement the `aspd_boost_on_kill_chance` passive in a similar way it would look something like this:

```lua
function Player:onKill()
    ...
    if self.chances.aspd_boost_on_kill_chance:next() then
        self.pre_boost_aspd_multiplier = self.aspd_multiplier
    	self.aspd_multiplier = self.aspd_multiplier/2
    	self.timer:after(4, function()
      	    self.aspd_multiplier = self.pre_boost_aspd_multiplier
            self.pre_boost_aspd_multiplier = nil
      	end)
    end
end
```

Here we simply do what we did for the HasteArea boost. We store the current attack speed multiplier, halve it, and then after a set duration (in this case 4 seconds), we restore it back to its original value. The problem with doing things this way happens whenever we want to stack these boosts together. 

Consider the situation where the player has entered a HasteArea and then gets an ASPD boost on kill. The problem here is that if the player exits the HasteArea before the 4 seconds for the boost duration are over then his `aspd_multiplier` variable will be restored to pre-ASPD boost levels, meaning that leaving the area will erase all other existing attack speed boosts.

And then also consider the situation where the player has an ASPD boost active and then enters a HasteArea. Whenever the boost duration ends the HasteArea effect will also be erased, since the `pre_boost_aspd_multiplier` will restore `aspd_multiplier` to a value that doesn't take into account the attack speed boost from the HasteArea. But even more worryingly, whenever the player exits the HasteArea he will now have permanently increased attack speed, since the save attack speed when he entered it was the one that was boosted from the ASPD boost.

So the main way we can fix this is by introducing a few variables:

```lua
function Player:new(...)
    ...
  	
    self.base_aspd_multiplier = 1
    self.aspd_multiplier = 1
    self.additional_aspd_multiplier = {}
end
```

Instead of only having the `aspd_multiplier` variable, now we'll have `base_aspd_multiplier` as well as `additional_aspd_multiplier`. `aspd_multiplier` will hold the current multiplier affected by all boosts. `base_aspd_multiplier` will hold the initial multiplier affected only by percentage increases. So if we have 50% increased attack speed from the tree, it will be applied on the constructor (in `setStats`) to `base_aspd_multiplier`. Then `additional_aspd_multiplier` will contain the added values of all boosts. So if we're inside a HasteArea, we would add the appropriate value to this table and then multiply its sum by the base every frame. So our update function for instance would look like this:

```lua
function Player:update(dt)
    ...
  	
    self.additional_aspd_multiplier = {}
    if self.inside_haste_area then table.insert(self.additional_aspd_multiplier, -0.5) end
    if self.aspd_boosting then table.insert(self.additional_aspd_multiplier, -0.5) end
    local aspd_sum = 0
    for _, aspd in ipairs(self.additional_aspd_multiplier) do
        aspd_sum = aspd_sum + aspd
    end
    self.aspd_multiplier = self.base_aspd_multiplier/(1 - aspd_sum)
end
```

In this way, every frame we'd be recalculating the `aspd_multiplier` variable based on the base as well as the boosts. There are a few multipliers that will make use of functionality very similar to this, so I'll just create a general object for this, since repeating it every time and with different variable names would be tiresome.

The `Stat` object looks like this:

```lua
Stat = Object:extend()

function Stat:new(base)
    self.base = base

    self.additive = 0
    self.additives = {}
    self.value = self.base*(1 + self.additive)
end

function Stat:update(dt)
    for _, additive in ipairs(self.additives) do self.additive = self.additive + additive end

    if self.additive >= 0 then
        self.value = self.base*(1 + self.additive)
    else
        self.value = self.base/(1 - self.additive)
    end

    self.additive = 0
    self.additives = {}
end

function Stat:increase(percentage)
    table.insert(self.additives, percentage*0.01)
end

function Stat:decrease(percentage)
    table.insert(self.additives, -percentage*0.01)
end
```

And the way we'd use it for our attack speed problem is like this:

```lua
function Player:new(...)
    ...
  	
    self.aspd_multiplier = Stat(1)
end

function Player:update(dt)
    ...
  
    if self.inside_haste_area then self.aspd_multiplier:decrease(100) end
    if self.aspd_boosting then self.aspd_multiplier:decrease(100) end
    self.aspd_multiplier:update(dt)
  
    ...
end
```

We would be able to access the attack speed multiplier at any point after `aspd_multiplier:update` is called by saying `aspd_multiplier.value`, and it would return us the correct result based on the base as well as the  all possible boosts applied. Because of this we need to change how the `aspd_multiplier` variable is used:

```lua
function Player:update(dt)
    ...
  
    -- Shoot
    self.shoot_timer = self.shoot_timer + dt
    if self.shoot_timer > self.shoot_cooldown*self.aspd_multiplier.value then
        self.shoot_timer = 0
        self:shoot()
    end
end
```

Here we just change `self.shoot_cooldown*self.aspd_multiplier` to `self.shoot_cooldown*self.aspd_multiplier.value`, since things wouldn't work out otherwise. Additionally, we can also change something else here. The way our `aspd_multiplier` variable works now is contrary to how every other variable in the game works. When we say that we get increased 10% HP, we know that `hp_multiplier` is 1.1, but when we say that we get increased 10% ASPD, `aspd_multiplier` is 0.9 instead. We can change this very and make `aspd_multiplier` behave the same way as other variables by dividing instead of multiplying it to `shoot_cooldown`:

```lua
if self.shoot_timer > self.shoot_cooldown/self.aspd_multiplier.value then
```

In this way, if we get a 100% increase in ASPD, its value will be 2 and we will be halving the cooldown between shots, which is what we want. Additionally we need to change the way we apply our boosts and instead of calling `decrease` on them we will call `increase`:

```lua
function Player:update(dt)
    ...
  
    if self.inside_haste_area then self.aspd_multiplier:increase(100) end
    if self.aspd_boosting then self.aspd_multiplier:increase(100) end
    self.aspd_multiplier:update(dt)
end
```

Another thing to keep in mind is that because `aspd_multiplier` is a `Stat` object and not just a number, whenever we implement the tree and import its values to the Player object we'll need to treat them differently. So the `treeToPlayer` function that I mentioned earlier will have to take this into account as well.

In any case, in this way we can easily implement "Gain ASPD Boost on Kill" correctly:

```lua
function Player:new(...)
    ...
  
    -- Chances
    self.gain_aspd_boost_on_kill_chance = 0
end
```

```lua
function Player:onKill()
    ...
  	
    if self.chances.gain_aspd_boost_on_kill_chance:next() then
        self.aspd_boosting = true
        self.timer:after(4, function() self.aspd_boosting = false end)
        self.area:addGameObject('InfoText', self.x, self.y, 
      	{text = 'ASPD Boost!', color = ammo_color})
    end
end
```

We can also delete the `enterHasteArea` and `exitHasteArea` functions, as well as changing how the HasteArea object works slightly:

```lua
function HasteArea:update(dt)
    HasteArea.super.update(self, dt)

    local player = current_room.player
    if not player then return end
    local d = distance(self.x, self.y, player.x, player.y)
    if d < self.r then player.inside_haste_area = true
    elseif d >= self.r then player.inside_haste_area = false end
end
```

Instead of any complicated logic like we had before, we simply set the Player's `inside_haste_area` attribute to true or false based on if the player is inside the area or not, and then because of the way we implemented the `Stat` object, the application of the attack speed boost that comes from a HasteArea will be done automatically.

<br>

**144. (CONTENT)** Implement the `mvspd_boost_on_cycle_chance` passive. A "MVSPD Boost" gives the player 50% increased movement speed for 4 seconds. Also implement the `mvspd_multiplier` variable and multiply it in the appropriate location.

**145. (CONTENT)** Implement the `pspd_boost_on_cycle_chance` passive. A "PSPD Boost" gives projectiles created by the player 100% increased movement speed for 4 seconds. Also implement the `pspd_multiplier` variable and multiply it in the appropriate location.

**146. (CONTENT)** Implement the `pspd_inhibit_on_cycle_chance` passive. A "PSPD Inhibit" gives projectiles created by the player 50% decreased movement speed for 4 seconds.

<br>

### While Boosting

These next passives we'll implement are the last ones of the "On Event Chance" type. So far all the ones we've focused on are chances of something happening on some event (on kill, on cycle, on resource pickup, ...) and these ones won't be different, since they will be chances for something to happen while boosting.

The first one we'll do is `launch_homing_projectile_while_boosting_chance`. The way this will work is that there will be a normal chance for the homing projectile to be launched, and this chance will be rolled on an interval of 0.2 seconds whenever we're boosting. This means that if we boost for 1 second, we'll roll this chance 5 times.

A good way of doing this is by defining two new functions: `onBoostStart` and `onBoostEnd` and then doing whatever it is we want to do to active the passive when the boost start, and then deactivate it when it ends. To add those two functions we need to change the boost code a little:

```lua
function Player:update(dt)
    ...
  
    -- Boost
    ...
    if self.boost_timer > self.boost_cooldown then self.can_boost = true end
    ...
    if input:pressed('up') and self.boost > 1 and self.can_boost then self:onBoostStart() end
    if input:released('up') then self:onBoostEnd() end
    if input:down('up') and self.boost > 1 and self.can_boost then 
        ...
        if self.boost <= 1 then
            self.boosting = false
            self.can_boost = false
            self.boost_timer = 0
            self:onBoostEnd()
        end
    end
    if input:pressed('down') and self.boost > 1 and self.can_boost then self:onBoostStart() end
    if input:released('down') then self:onBoostEnd() end
    if input:down('down') and self.boost > 1 and self.can_boost then 
        ...
        if self.boost <= 1 then
            self.boosting = false
            self.can_boost = false
            self.boost_timer = 0
            self:onBoostEnd()
        end
    end
    ...
end
```

Here we add `input:pressed` and `input:released`, which return true only whenever those events happen, and with that we can be sure that `onBoostStart` and `onBoostEnd` will only be called once when those events happen. We also add `onBoostEnd` to inside the `input:down` conditional in case the player doesn't release the button but the amount of boost available to him ends and therefore the boost ends as well.

Now for the `launch_homing_projectile_while_boosting_chance` part:

```lua
function Player:new(...)
    ...
  
    -- Chances
    self.launch_homing_projectile_while_boosting_chance = 0
end

function Player:onBoostStart()
    self.timer:every('launch_homing_projectile_while_boosting_chance', 0.2, function()
        if self.chances.launch_homing_projectile_while_boosting_chance:next() then
            local d = 1.2*self.w
            self.area:addGameObject('Projectile', 
          	self.x + d*math.cos(self.r), self.y + d*math.sin(self.r), 
                {r = self.r, attack = 'Homing'})
            self.area:addGameObject('InfoText', self.x, self.y, {text = 'Homing Projectile!'})
        end
    end)
end

function Player:onBoostEnd()
    self.timer:cancel('launch_homing_projectile_while_boosting_chance')
end
```

Here whenever a boost starts we call `timer:every` to roll a chance for the homing projectile every 0.2 seconds, and then whenever a boost ends we cancel that timer. Here's what that looks like if the chance of this event happening was 100%:

<p align="center">
<img src="https://vgy.me/qrF772.gif">
</p>

<br>

**147. (CONTENT)** Implement the `cycle_speed_multiplier` variable. This variable makes the cycle speed faster or slower based on its value. So, for instance, if `cycle_speed_multiplier` is 2 and our default cycle duration is 5 seconds, then applying it would turn our cycle duration to 2.5 instead.

**148. (CONTENT)** Implement the `increased_cycle_speed_while_boosting` passive. This variable should be a boolean that signals if the cycle speed should be increased or not whenever the player is boosting. The boost should be an increase of 200% to cycle speed multiplier.

**149. (CONTENT)** Implement the `invulnerability_while_boosting` passive. This variable should be a boolean that signals if the player should be invulnerable whenever he is boosting. Make use of the `invincible` attribute which already exists and serves the purpose of making the player invincible.

<br>

### Increased Luck While Boosting

The final "While Boosting" type of passive we'll implement is "Increased Luck While Boosting". Before we can implement it though we need to implement the `luck_multiplier` stat. Luck is one of the main stats of the game and it works by increasing the chances of favorable events to happen. So, let's say you have 10% chance to launch a homing projectile on kill. If `luck_multiplier` is 2, then this chance becomes 20% instead. 

The way to implement this turns out to be very very simple. All "chance" type passives go through the `generateChances` function, so we can just implement this there:

```lua
function Player:generateChances()
    self.chances = {}
    for k, v in pairs(self) do
        if k:find('_chance') and type(v) == 'number' then
      	    self.chances[k] = chanceList(
            {true, math.ceil(v*self.luck_multiplier)}, 
            {false, 100-math.ceil(v*self.luck_multiplier)})
        end
    end
end
```

And here we simply multiply `v` by our `luck_multiplier` and it should work as expected. With this we can go on to implement the `increased_luck_while_boosting` passive like this:

```lua
function Player:onBoostStart()
    ...
    if self.increased_luck_while_boosting then 
    	self.luck_boosting = true
    	self.luck_multiplier = self.luck_multiplier*2
    	self:generateChances()
    end
end

function Player:onBoostEnd()
    ...
    if self.increased_luck_while_boosting and self.luck_boosting then
    	self.luck_boosting = false
    	self.luck_multiplier = self.luck_multiplier/2
    	self:generateChances()
    end
end
```

Here we implement it like we initially did for the `HasteArea` object. The reason we can do this now is because there will not be any other passives that will give the Player a luck boost, which means that we don't have to worry about multiple boosts possibly overriding each other. If we had multiple passives giving boosts to luck, then we'd need to make it a `Stat` object like we did for the `aspd_multiplier`.

Also importantly, whenever we change our luck multiplier we also call `generateChances` again, otherwise our luck boost will not really affect anything. There's a downside to this which is that all lists get reset, and so if some list randomly selected a bunch of unlucky rolls and then it gets reset here, it could select a bunch of unlucky rolls again instead of following the chanceList property where it would be less likely to select more unlucky rolls as time goes on. But this is a very minor problem that I personally don't really worry about.

<br>

### HP Spawn Chance Multiplier

Now we'll go over `hp_spawn_chance_multiplier`, which increases the chance that whenever the Director spawns a new resource, that resource will be an HP one. This is a fairly straightforward implementation if we remember how the Director works:

```lua
function Player:new(...)
    ...
  	
    -- Multipliers
    self.hp_spawn_chance_multiplier = 1
end
```

```lua
function Director:new(...)
    ...
  
    self.resource_spawn_chances = chanceList({'Boost', 28}, 
    {'HP', 14*current_room.player.hp_spawn_chance_multiplier}, {'SkillPoint', 58})
end
```

On article 9 we went over the creation of the chances for each resource to be spawned. The `resource_spawn_chances` chanceList holds those chances, and so all we have to do is make sure that we use `hp_spawn_chance_multiplier` to increase the chances that the HP resource will be spawned according to the multiplier.

It's also important here to initialize the Director after the Player in the Stage room, since the Director depends on variables the Player has while the Player doesn't depend on the Director at all.

<br>

**150. (CONTENT)** Implement the `spawn_sp_chance_multiplier` passive.

**151. (CONTENT)** Implement the `spawn_boost_chance_multiplier` passive.

<br>

Given everything we've implemented so far, these next exercises can be seen as challenges. I haven't gone over most aspects of their implementation, but they're pretty simple compared to everything we've done so far so they should be straightforward.

<br>

**152. (CONTENT)** Implement the `drop_double_ammo_chance` passive. Whenever an enemy dies there will be a chance that it will create two Ammo objects instead of one.

**153. (CONTENT)** Implement the `attack_twice_chance` passive. Whenever the player attacks there will be a chance to call the `shoot` function twice.

**154. (CONTENT)** Implement the `spawn_double_hp_chance` passive. Whenever an HP resource is spawned by the Director there will be a chance that it will create two HP objects instead of one.

**155. (CONTENT)** Implement the `spawn_double_sp_chance` passive. Whenever a SkillPoint resource is spawned by the Director there will be a chance that it will create two SkillPoint objects instead of one.

**156. (CONTENT)** Implement the `gain_double_sp_chance` passive. Whenever the player collects a SkillPoint resource there will be a chance that he will gain two skill points instead of one.

<br>

### Enemy Spawn Rate

The `enemy_spawn_rate_multiplier` will control how fast the Director changes difficulties. By default this happens every 22 seconds, but if `enemy_spawn_rate_multiplier` is 2 then this will happen every 11 seconds instead. This is another rather straightforward implementation:

```lua
function Player:new(...)
    ...
  	
    -- Multipliers
    self.enemy_spawn_rate_multiplier = 1
end
```

```lua
function Director:update(dt)
    ...
  	
    -- Difficulty
    self.round_timer = self.round_timer + dt
    if self.round_timer > self.round_duration/self.stage.player.enemy_spawn_rate_multiplier then
        ...
    end
end
```

So here we just divide `round_duration` by `enemy_spawn_rate_multiplier` to get the target round duration.

<br>

**157. (CONTENT)** Implement the `resource_spawn_rate_multiplier` passive.

**158. (CONTENT)** Implement the `attack_spawn_rate_multiplier` passive.

<br>

And here are some more exercises for some more passives. These are mostly multipliers that couldn't fit into any of the classes of passives talked about before but should be easy to implement.

**159. (CONTENT)** Implement the `turn_rate_multiplier` passive. This is a passive that increases or decreases the speed with which the Player's ship turns.

**160. (CONTENT)** Implement the `boost_effectiveness_multiplier` passive. This is a passive that increases or decreases the effectiveness of boosts. This means that if this variable has the value of 2, a boost will go twice as fast or twice as slow as before.

**161. (CONTENT)** Implement the `projectile_size_multiplier` passive. This is a passive that increases or decreases the size of projectiles.

**162. (CONTENT)** Implement the `boost_recharge_rate_multiplier` passive. This is a passive that increases or decreases how fast boost is recharged. 

**163. (CONTENT)** Implement the `invulnerability_time_multiplier` passive. This is a passive that increases or decreases the duration of the player's invulnerability after he's hit.

**164. (CONTENT)** Implement the `ammo_consumption_multiplier` passive. This is a passive that increases or decreases the amount of ammo consumed by all attacks.

**165. (CONTENT)** Implement the `size_multiplier` passive. This is a passive that increases or decreases the size of the player's ship. Note that that the positions of the trails for all ships, as well as the position of projectiles as they're fired need to be changed accordingly.

**166. (CONTENT)** Implement the `stat_boost_duration_multiplier` passive. This is a passive that increases of decreases the duration of temporary buffs given to the player.

<br>

### Projectile Passives

Now we'll focus on a few projectile passives. These passives will change how our projectiles behave in some fundamental way. These same ideas can also be implemented in the `EnemyProjectile` object and then we can create enemies that use some of this as well. For instance, there's a passive that makes your projectiles orbit around you instead of just going straight. Later on we'll add an enemy that has tons of projectiles orbiting it as well and the technology behind it is the same for both situations.

### 90 Degree Change

We'll call this passive `projectile_ninety_degree_change` and what it will do is that the angle of the projectile will be changed by 90 degrees periodically. The way this looks is like this:

<p align="center">
<img src="https://vgy.me/F3r2OS.gif">
</p>

Notice that the projectile roughly moves in the same direction it was moving towards as it was shot, but its angle changes rapidly by 90 degrees each time. This means that the angle change isn't entirely randomly decided and we have to put some thought into it.

The basic way we can go about this is to say that `projectile_ninety_degree_change` will be a boolean and that the effect will apply whenever it is true. Because we're going to apply this effect in the `Projectile` class, we have two options in regards to how we'll read from it that the Player's `projectile_ninety_degree_change` variable is true or not: either pass that in in the `opts` table whenever we create a new projectile from the `shoot` function, or read that directly from the player by accessing it through `current_room.player`. I'll go with the second solution because it's easier and there are no real drawbacks to it, other than having to change `current_room.player` to something else whenever we move some of this code to `EnemyProjectile`. The way all this would look is something like this:

```lua
function Player:new(...)
    ...
  	
    -- Booleans
    self.projectile_ninety_degree_change = false
end
```

```lua
function Projectile:new(...)
    ...

    if current_room.player.projectile_ninety_degree_change then

    end
end
```

Now what we have to do inside the conditional in the Projectile constructor is to change the projectile's angle each time by 90 degrees, but also respecting its original direction. What we can do is first change the angle by either 90 degrees or -90 degrees randomly. This would look like this:

```lua
function Projectile:new(...)
    ...

    if current_room.player.projectile_ninety_degree_change then
        self.timer:after(0.2, function()
      	    self.ninety_degree_direction = table.random({-1, 1})
            self.r = self.r + self.ninety_degree_direction*math.pi/2
      	end)
    end
end
```

<p align="center">
<img src="https://vgy.me/8tfgY0.gif">
</p>

Now what we need to do is figure out how to turn the projectile in the other direction, and then turn it back in the other, and then again, and so on. It turns out that since this is a periodic thing that will happen forever, we can use `timer:every`:

```lua
function Projectile:new(...)
    ...
  	
    if current_room.player.projectile_ninety_degree_change then
        self.timer:after(0.2, function()
      	    self.ninety_degree_direction = table.random({-1, 1})
            self.r = self.r + self.ninety_degree_direction*math.pi/2
            self.timer:every('ninety_degree_first', 0.25, function()
                self.r = self.r - self.ninety_degree_direction*math.pi/2
                self.timer:after('ninety_degree_second', 0.1, function()
                    self.r = self.r - self.ninety_degree_direction*math.pi/2
                    self.ninety_degree_direction = -1*self.ninety_degree_direction
                end)
            end)
      	end)
    end
end
```

At first we turn the projectile in the opposite direction that we turned it initially, which means that now it's facing its original angle. Then, after only 0.1 seconds, we turn it again in that same direction so that it's facing the opposite direction to when it first turned. So, if it was fired facing right, what happened is: after 0.2 seconds it turned up, after 0.25 it turned right again, after 0.1 seconds it turned down, and then after 0.25 seconds it will repeat by turning right then up, then right then down, and so on. 

Importantly, at the end of each `every` loop we change the direction it should turn towards, otherwise it wouldn't oscillate between up/down and would keep going up/down instead of straight. Doing all that looks like this:

<p align="center">
<img src="https://vgy.me/nDoGUW.gif">
</p>

<br>

**167. (CONTENT)** Implement the `projectile_random_degree_change` passive, which changes the angle of the projectile randomly instead. Unlike the 90 degrees one, projectiles in this one don't need to retain their original direction.

**168. (CONTENT)** Implement the `angle_change_frequency_multiplier` passive. This is a passive that increases or decreases the speed with which angles change in the previous 2 passives. If `angle_change_frequency_multiplier` is 2, for instance, then instead of angles changing with 0.25 and 0.1 seconds, they will change with 0.125 and 0.05 seconds instead.

<br>

### Wavy Projectiles

Instead of abruptly changing the angle of our projectile, we can do it softly using the `timer:tween` function, and in this way we can get a wavy projectile effect that looks like this:

<p align="center">
<img src="https://vgy.me/cKQ2Km.gif">
</p>

The idea is almost the same as the previous examples but using `timer:tween` instead:

```lua
function Projectile:new(...)
    ...
  	
    if current_room.player.wavy_projectiles then
        local direction = table.random({-1, 1})
        self.timer:tween(0.25, self, {r = self.r + direction*math.pi/8}, 'linear', function()
            self.timer:tween(0.25, self, {r = self.r - direction*math.pi/4}, 'linear')
        end)
        self.timer:every(0.75, function()
            self.timer:tween(0.25, self, {r = self.r + direction*math.pi/4}, 'linear',  function()
                self.timer:tween(0.5, self, {r = self.r - direction*math.pi/4}, 'linear')
            end)
        end)
    end
end
```

Because of the way `timer:every` works, in that it doesn't start performing its functions until after the initial duration, we first do one iteration of the loop manually, and then after that the every loop takes over. In the first iteration we also use an initial value of math.pi/8 instead of math.pi/4 because we only want the projectile to tween half of what it usually does, since it starts in the middle position (as it was just shot from the Player) instead of on either edge of the oscillation.

<br>

**169. (CONTENT)** Implement the `projectile_waviness_multiplier` passive. This is a passive that increases or decreases the target angle that the projectile should reach when tweening. If `projectile_waviness_multiplier` is 2, for instance, then the arc of its path will be twice as big as normal.

<br>

### Acceleration and Deceleration

Now we'll go for a few passives that change the speed of the projectile. The first one is "Fast -> Slow" and the second is "Slow -> Fast", meaning, the projectile starts with either fast or slow velocity, and then transitions into either slow or fast velocity. This is what "Fast -> Slow" looks like:

<p align="center">
<img src="https://vgy.me/nzGiGF.gif">
</p>

The way we'll implement this is pretty straightforward. For the "Fast -> Slow" one we'll tween the velocity to double its initial value quickly, and then after a while tween it down to half its initial value. And for the other we'll simply do the opposite.

```lua
function Projectile:new(...)
    ...
  	
    if current_room.player.fast_slow then
        local initial_v = self.v
        self.timer:tween('fast_slow_first', 0.2, self, {v = 2*initial_v}, 'in-out-cubic', function()
            self.timer:tween('fast_slow_second', 0.3, self, {v = initial_v/2}, 'linear')
        end)
    end

    if current_room.player.slow_fast then
        local initial_v = self.v
        self.timer:tween('slow_fast_first', 0.2, self, {v = initial_v/2}, 'in-out-cubic', function()
            self.timer:tween('slow_fast_second', 0.3, self, {v = 2*initial_v}, 'linear')
        end)
    end
end
```

<br>

**170. (CONTENT)** Implement the `projectile_acceleration_multiplier` passive. This is a passive that controls how fast or how slow a projectile accelerates whenever it changes to a higher velocity than its original value.

**171. (CONTENT)** Implement the `projectile_deceleration_multiplier` passive. This is a passive that controls how fast or how slow a projectile decelerates whenever it changes to a lower velocity than its original value.

<br>

### Shield Projectiles

This one is a bit more involved than the others because it has more moving parts to it, but this is what the end result should look like:

<p align="center">
<img src="https://vgy.me/8FfF10.gif">
</p>

As you can see, the projectiles orbit around the player and also sort of inherit its movement direction. The way we can achieve this is by using a circle's [parametric equation](http://mathworld.wolfram.com/ParametricEquations.html). In general, if we want A to orbit around B with some radius R then we can do something like this:

```lua
Ax = Bx + R*math.cos(time)
Ay = By + R*math.sin(time)
```

Where `time` is a variable that goes up as times passes. Before we get to implementing this let's set everything else up. `shield_projectile_chance` will be a chance-type variable instead of a boolean, meaning that every time a new projectile will be created there will be a chance it will orbit the player.

```lua
function Player:new(...)
    ...
  	
    -- Chances
    self.shield_projectile_chance = 0
end

function Player:shoot()
    ...
    local shield = self.chances.shield_projectile_chance:next()
  	
    if self.attack == 'Neutral' then
        self.area:addGameObject('Projectile', self.x + 1.5*d*math.cos(self.r), 
        self.y + 1.5*d*math.sin(self.r), {r = self.r, attack = self.attack, shield = shield})
	...
end
```

Here we define the `shield` variable with the roll of if this projectile should be orbitting the player or not, and then we pass that in the `opts` table of the `addGameObject` call. Here we have to repeat this step for every attack type we have. Since we'll have to make future changes like this one, we can just do something like this instead now:

```lua
function Player:shoot()
    ...
  	
    local mods = {
        shield = self.chances.shield_projectile_chance:next()
    }

    if self.attack == 'Neutral' then
        self.area:addGameObject('Projectile', self.x + 1.5*d*math.cos(self.r), 
        self.y + 1.5*d*math.sin(self.r), table.merge({r = self.r, attack = self.attack}, mods))
        ...
end
```

And so in this way, in the future we'll only have to add things to the `mods` table. The `table.merge` function hasn't been defined yet, but you can guess what it does based on how we're using it here.

```lua
function table.merge(t1, t2)
    local new_table = {}
    for k, v in pairs(t2) do new_table[k] = v end
    for k, v in pairs(t1) do new_table[k] = v end
    return new_table
end
```

It simply joins two tables together with all their values into a new one and then returns it.

Now we can start with the actual implementation of the `shield` functionality. At first we want to define a few variables, like the radius, the orbit speed and so on. For now I'll define them like this:

```lua
function Projectile:new(...)
    ...
  	
    if self.shield then
        self.orbit_distance = random(32, 64)
        self.orbit_speed = random(-6, 6)
        self.orbit_offset = random(0, 2*math.pi)
    end
end
```

`orbit_distance` represents the radius around the player. `orbit_speed` will be multiplied by `time`, which means that higher absolute values will make it go faster, while lower ones will make it go slower. Negative values will make the projectile turn in the other direction which adds some randomness to it. `orbit_offset` is the initial angle offset that each projectile will have. This also adds some randomness to it and prevents all projectiles from being started at roughly the same position. And now that we have all these defined we can apply the circle's parametric equation to the projectile's position:

```lua
function Projectile:update(dt)
    ...
  
    -- Shield
    if self.shield then
        local player = current_room.player
        self.collider:setPosition(
      	player.x + self.orbit_distance*math.cos(self.orbit_speed*time + self.orbit_offset),
      	player.y + self.orbit_distance*math.sin(self.orbit_speed*time + self.orbit_offset))
    end
  	
    ...
end
```

It's important to place this after any other calls we may make to `setLinearVelocity` otherwise things won't work out. We also shouldn't forget to add the global `time` variable and increase it by `dt` every frame. If we do that correctly then it should look like this:

<p align="center">
<img src="https://vgy.me/sMW91I.gif">
</p>

And this gets the job done but it looks wrong. The main thing wrong with it is that the projectile's angles are not taking into account the rotation around the player. One way to fix this is to store the projectile's position last frame and then get the angle of the vector that makes up the subtraction of the current position by the previous position. Code is worth a thousand words so that looks like this:

```lua
function Projectile:new(...)
    ...
  	
    self.previous_x, self.previous_y = self.collider:getPosition()
end

function Projectile:update(dt)
    ...
  	
    -- Shield
    if self.shield then
        ...
        local x, y = self.collider:getPosition()
        local dx, dy = x - self.previous_x, y - self.previous_y
        self.r = Vector(dx, dy):angle()
    end

    ...
  
    -- At the very end of the update function
    self.previous_x, self.previous_y = self.collider:getPosition()
end
```

And in this way we're setting the `r` variable to contain the angle of the projectile while taking into account its rotation. Because we're using `setLinearVelocity` and using that angle, it means that when we draw the projectile in `Projectile:draw` and use `Vector(self.collider:getLinearVelocity()):angle())` to get our direction, everything will be set according what we set the `r` variable to. And so all that looks like this:

<p align="center">
<img src="https://vgy.me/ZPu74F.gif">
</p>

And this looks about right. One small problem that you can see in the gif above is that as projectiles are fired, if they turn into shield projectiles they don't do it instantly. There's a 1-2 frame delay where they look like normal projectiles and then they disappear and appear orbiting the player. One way to fix this is to just hide all shield projectiles for 1-2 frames and then unhide them:

```lua
function Projectile:new(...)
    ...
  	
    if self.shield then
        ...
    	self.invisible = true
    	self.timer:after(0.05, function() self.invisible = false end)
    end
end

function Projectile:draw()
    if self.invisible then return end
    ...
end
```

And finally, it would be pretty OP if shield projectiles could just stay there forever until they hit an enemy, so we need to add a projectile duration such that after that duration ends the projectile will be killed:

```lua
function Projectile:new(...)
    ...
  	
    if self.shield then
    	...
    	self.timer:after(6, function() self:die() end)
    end
end
```

And in this way after 6 seconds of existence our shield projectiles will die.

<br>

## END

I'm going to end it here because the editor I'm using to write this is starting to choke on the size of this article. In the next article we'll continue with the implementation of more passives, as well as adding all player attacks, enemies, and passives related to them. The next article also marks the end of implementation of all content in the game, and the ones coming after that will focus on how to present that content to the player (SkillTree and Console rooms).

<br>

---

If you liked these tutorials and want to support the writing of more like these in the future:

* ### [BYTEPATH on Steam](http://store.steampowered.com/app/760330/BYTEPATH/)
* ### [BYTEPATH-tutorial on gumroad](https://gumroad.com/l/BYTEPATH-tutorial)

Buying the tutorial gives you access to the game's full source code, answers to exercises from articles 1 through 9, the code separated by articles (what the code should look like at the end of each article) and a Steam key to the game.
